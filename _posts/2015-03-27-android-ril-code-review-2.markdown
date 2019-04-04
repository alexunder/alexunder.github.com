---
layout: post
title:  Android RIL code review 2
category: Android
---

# Overview
In this section, I will discuss the details inside the libril.so, these details include:

* RIL event mechanism.
* Communications with frameworks.
* Supports for vendor RIL.

# RIL event mechanism

As the last section's show us, it's clear that RIL_startEventLoop create a thread named eventloop, its code will show below:

{% highlight c %}
    static void * eventLoop(void *param) {
        int ret;
        int filedes[2];

        if (signal(SIGPIPE, rilSignalHandler)== SIG_ERR) {
            LOGE ("register SIGPIPE handler fail!!!");
        }
        ril_event_init();

        pthread_mutex_lock(&s_startupMutex);

        s_started = 1;
        pthread_cond_broadcast(&s_startupCond);

        pthread_mutex_unlock(&s_startupMutex);

        ret = pipe(filedes);

        if (ret < 0) {
            LOGE("Error in pipe() errno:%d", errno);
            return NULL;
        }

        s_fdWakeupRead = filedes[0];
        s_fdWakeupWrite = filedes[1];

        fcntl(s_fdWakeupRead, F_SETFL, O_NONBLOCK);

        ril_event_set (&s_wakeupfd_event, s_fdWakeupRead, true,
                processWakeupCallback, NULL);

        rilEventAddWakeup (&s_wakeupfd_event);

        // Only returns on error
        ril_event_loop();
        LOGE ("error in event_loop_base errno:%d", errno);
        // kill self to restart on error
        kill(0, SIGKILL);

        return NULL;
    }
{% endhighlight %}
   
It invokes ril_event_init to initialize two ril_event structure objects: timer_list and pending_list:

{% highlight c %}
    void ril_event_init()
    {
        MUTEX_INIT();

        FD_ZERO(&readFds);
        init_list(&timer_list);
        init_list(&pending_list);
        memset(watch_table, 0, sizeof(watch_table));
    }
{% endhighlight %}

At the end of this function, it also zeros a ril_event structure pointer array which has 8 elements: watch_table. We will know these objects later on. So let's check the structure ril_event:

{% highlight c %}
    struct ril_event {
        struct ril_event *next;
        struct ril_event *prev;

        int fd;
        int index;
        bool persist;
        struct timeval timeout;
        ril_event_cb func;
        void *param;
    };
{% endhighlight %}

We can see that ril_event is a doubly linked list. 

We use ril_event_init to initialize all event tables, including timer_list, pending_list and watch_table, their declarations are like below:

    static struct ril_event * watch_table[MAX_FD_EVENTS];
    static struct ril_event timer_list;
    static struct ril_event pending_list;

We use ril_event_set to fill the ril_event object with input parameters. 

{% highlight c %}
    void ril_event_set(struct ril_event * ev, int fd, bool persist, ril_event_cb func, void * param)
    {
        dlog("~~~~ ril_event_set %x ~~~~", (unsigned int)ev);
        memset(ev, 0, sizeof(struct ril_event));
        ev->fd = fd;
        ev->index = -1;
        ev->persist = persist;
        ev->func = func;
        ev->param = param;
        fcntl(fd, F_SETFL, O_NONBLOCK);
    }
{% endhighlight %}

The fd maybe the file descriptor of the socket, and set it as non-blocking status via invoking fcntl.

We also use ril_event_add to add the new event to watch_table, and set the fd into fd_set . So,  if some data from these fds are available, the callback function from ril_event  can be invoked, there is a thread(eventloop) do this thing, we will talk about it later. We also use ril_event_del to remove the specific ril_event object from  watch_table. And the function "ril_timer_add", we can use it to add the event to timer_list, and sort it with time out period's ascending order.

And now, we will go back to the function "eventloop". After initializing ril_event tables, there is a pipe defined, then put the read fd and the related informations into a ril-event, and add it to the watch_table, the implementation of the callback function is below:

{% highlight c %}
static void processWakeupCallback(int fd, short flags, void *param) {
    char buff[16];
    int ret;

    LOGV("processWakeupCallback");

    /* empty our wakeup socket out */
    do {
        ret = read(s_fdWakeupRead, &buff, sizeof(buff));
    } while (ret > 0 || (ret < 0 && errno == EINTR));
}
{% endhighlight %}

We can write some data to the write fd of the pipe to wake up the thread, it seems that no reason to do that. But the interesting thing is that every time if there is the new ril_event object, will be added, the trigger function  "triggerEvLoop" will be invoked. The rilEventAddWakeup detail will show this:

{% highlight c %}
   static void rilEventAddWakeup(struct ril_event *ev) {
       ril_event_add(ev);
       triggerEvLoop();
   } 
{% endhighlight %}

I guess if rilEventAddWakeup will be called in another thread, this invoking will activate the eventloop thread. The triggerEvLoop function will trigger processWakeupCallback function:

{% highlight c %}
    static void triggerEvLoop() {
        int ret;
        if (!pthread_equal(pthread_self(), s_tid_dispatch)) {
            /* trigger event loop to wakeup. No reason to do this,
             * if we're in the event loop thread */
             do {
                ret = write (s_fdWakeupWrite, " ", 1);
             } while (ret < 0 && errno == EINTR);
        }
    }
{% endhighlight %}

## The core part of eventloop

The most important part of eventloop is ril_event_loop, it uses select system call to manage some file descriptors. And it has some details need to analyze.

{% highlight c %}
    void ril_event_loop()
    {
        int n;
        fd_set rfds;
        struct timeval tv;
        struct timeval * ptv;


        for (;;) {

            // make local copy of read fd_set
            memcpy(&rfds, &readFds, sizeof(fd_set));
            if (-1 == calcNextTimeout(&tv)) {
                // no pending timers; block indefinitely
                dlog("~~~~ no timers; blocking indefinitely ~~~~");
                ptv = NULL;
            } else {
                dlog("~~~~ blocking for %ds + %dus ~~~~", (int)tv.tv_sec, (int)tv.tv_usec);
                ptv = &tv;
            }
            printReadies(&rfds);
            n = select(nfds, &rfds, NULL, NULL, ptv);
            printReadies(&rfds);
            dlog("~~~~ %d events fired ~~~~", n);
            if (n < 0) {
                if (errno == EINTR) continue;

                LOGE("ril_event: select error (%d)", errno);
                // bail?
                return;
            }

            // Check for timeouts
            processTimeouts();
            // Check for read-ready
            processReadReadies(&rfds, n);
            // Fire away
            firePending();
        }
    }
{% endhighlight %}

It's a dead loop, but it has an error's return.  So let's start from  calcNextTimeout.

We use this function to check if we have pending timers in timer_list. If there is no element in timer_list, the function returns  -1, and select system call will block itself indefinitely. We should go inside this function:
	
{% highlight c %}
	static int calcNextTimeout(struct timeval * tv)
	{
	    struct ril_event * tev = timer_list.next;
	    struct timeval now;
	
	    getNow(&now);
	
	    // Sorted list, so calc based on first node
	    if (tev == &timer_list) {
	        // no pending timers
	        return -1;
	    }
	
	    dlog("~~~~ now = %ds + %dus ~~~~", (int)now.tv_sec, (int)now.tv_usec);
	    dlog("~~~~ next = %ds + %dus ~~~~",
	            (int)tev->timeout.tv_sec, (int)tev->timeout.tv_usec);
	    if (timercmp(&tev->timeout, &now, >)) {
	        timersub(&tev->timeout, &now, tv);
	    } else {
	        // timer already expired.
	        tv->tv_sec = tv->tv_usec = 0;
	    }
	    return 0;
	}
{% endhighlight %}
	
If we have the pending timer event, we will compare the first element's timeout value with the current time (start from system's booting). If tev->timeout is larger than the current time, the event tev don't overtime, so set the difference between tev's timeout value and the current time into tv. vise versa, if tev->timeout is smaller than the current time, we will set zero to tv. So back to ril_event_loop, the select system call can get the timeout value from calcNextTimeout. I think we should know how to get the pending timer, so let's check the detail of ril_timer_add:

{% highlight c %}
    void ril_timer_add(struct ril_event * ev, struct timeval * tv)
    {
        dlog("~~~~ +ril_timer_add ~~~~");
        MUTEX_ACQUIRE();

        struct ril_event * list;
        if (tv != NULL) {
            // add to timer list
            list = timer_list.next;
            ev->fd = -1; // make sure fd is invalid

            struct timeval now;
            getNow(&now);
            timeradd(&now, tv, &ev->timeout);

            // keep list sorted
            while (timercmp(&list->timeout, &ev->timeout, < )
                && (list != &timer_list)) {
                list = list->next;
            }
            // list now points to the first event older than ev
            addToList(ev, list);
        }

        MUTEX_RELEASE();
        dlog("~~~~ -ril_timer_add ~~~~");
    }
{% endhighlight %}

As previous code snippets show us, the timer_list was initialized by init_list in ril_event_init, so if the timer_list has no elements, the timer_list's next pointer and prev pointer all point to itself. We add the relative time interval to the current time, then set this sum to the new ril_event ev, then find the place that keeps the larger one located in the back, so the first one is the oldest event.

After that, we can confirm the timeout value that will be passed into the select system call. If the invoking to select has an error, it will break the dead loop, but the error from interrupt is ignored. If some file descriptors' data are available, the final 3 functions will be invoked:

* processTimeouts.
* processReadReadies
* firePending

## The final tree kicks

If the data comes, we left three kicks. So let's dive into the first kick:

{% highlight c %}
    static void processTimeouts()
    {
        dlog("~~~~ +processTimeouts ~~~~");
        MUTEX_ACQUIRE();
        struct timeval now;
        struct ril_event * tev = timer_list.next;
        struct ril_event * next;

        getNow(&now);
        // walk list, see if now >= ev->timeout for any events

        dlog("~~~~ Looking for timers <= %ds + %dus ~~~~", (int)now.tv_sec, (int)now.tv_usec);
        while ((tev != &timer_list) && (timercmp(&now, &tev->timeout, >))) {
            // Timer expired
            dlog("~~~~ firing timer ~~~~");
            next = tev->next;
            removeFromList(tev);
            addToList(tev, &pending_list);
            tev = next;
        }
        MUTEX_RELEASE();
        dlog("~~~~ -processTimeouts ~~~~");
    }
{% endhighlight %}

It's simple to interpret: we traverse the time_list to find the event whose timeout is smaller than the current time,  in other words, we want to find the timeout event, and remove it from the timer_list, then add to the pending_list.

The second kick:

{% highlight c %}
    static void processReadReadies(fd_set * rfds, int n)
    {
        dlog("~~~~ +processReadReadies (%d) ~~~~", n);
        MUTEX_ACQUIRE();

        for (int i = 0; (i < MAX_FD_EVENTS) && (n > 0); i++) {
            struct ril_event * rev = watch_table[i];
            if (rev != NULL && FD_ISSET(rev->fd, rfds)) {
                addToList(rev, &pending_list);
            if (rev->persist == false) {
                removeWatch(rev, i);
                }
                n--;
            }
        }

        MUTEX_RELEASE();
        dlog("~~~~ -processReadReadies (%d) ~~~~", n);
    }
{% endhighlight %}

In this function, we scan watch_table to check if an element is ready for receiving data, then add it to the pending_list. Whether we delete the ready item from watch_table depends on persist value in this item.

The final kick:

{% highlight c %}
    static void firePending()
    {
        dlog("~~~~ +firePending ~~~~");
        struct ril_event * ev = pending_list.next;
        while (ev != &pending_list) {
            struct ril_event * next = ev->next;
            removeFromList(ev);
            ev->func(ev->fd, 0, ev->param);
            ev = next;
        }
        dlog("~~~~ -firePending ~~~~");
    }
{% endhighlight %}

And now, the pending_list includes all the events will be ready to read, so we traverse the pending_list, fire all of them, invoking their own callback functions.

# Temporary End

This note is long enough, the left content will be delivered in the following post.
