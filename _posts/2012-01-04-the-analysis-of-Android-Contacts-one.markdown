---
layout: post
title: Android Contacts 代码分析
category: Android 
---

# 前言

本来不打算发表Android方面的文章，但是本着记录生活的精神，还是应该将工作当中总结出来的东西写出来。本人刚接触Android时间不长，所以写的东西难免会有错误，还希望看到我文章的高手为我指出。

# Android Contacts总览

Contacts应用是由Google Android团队编写的Android原生应用。在应用层面上涉及到Contacts.apk, ContactProvider.apk。其他相关的在Framwork，以及framework与linux内核之间的SQLite.Contacts.apk只是界面层的逻辑，主要实现UI的流程。对于联系人的查询，存储，增加和删除都在ContactProvider.apk中封装，是对底层的SQLite进行封装。数据的操作最终都是在底层的SQLite的C代码中进行。

# 1. Contacts

## 初始化

当你在Android的界面中，点击Contacts的图标，Contacts应用会秀出来。先看在Contacts的AndroidManifest.xml中：

{% highlight xml%}
	<intent-filter>
           <action android:name="com.android.contacts.action.LIST_DEFAULT" />
           <category android:name="android.intent.category.DEFAULT" />
           <category android:name="android.intent.category.TAB" />
	</intent-filter>
{% endhighlight %}

这是每次默认点击图标时的intent，这是ContactsListActivity，他继承自ListActivity：

![ContactsListActivity](\images\article\ContactsListActivity.PNG  "Android Contact")

所以先运行：

{% highlight java %}
ContactsListActivity::OnCreate
{% endhighlight %}

在ContactsListActivity::OnCreate中会调用SetupListView,用来创建联系人的List。

在setupListView中，通过getListView获得当前ListActivity的ListView对象，通过getLayoutInflater获得当前Activity的UI布局layout对象，然后新建ContactItemListAdapter，继承自CursorAdaptor，还继承了SectionIndexer, OnScrollListener, PinnedHeaderListView.PinnedHeaderAdapter等inteface:

![ContactItemListAdapter](\images\article\ContactItemListAdapter.PNG  "Android Contact")

CursorAdapter负责将从ContentProvider中获得数据对象Cursor填充到相应的View中，新建的时候将当前的Activity的this对象传入，如果需要更新Cursor数据，在查询或改变的时候，调用ContactItemListAdapter::setSuggestionsCursor来设置新的Cursor数据，通过实现getView函数来用新数据刷新UI, getView定义如下：

{% highlight java %}
public abstract View getView (int position, View convertView, ViewGroup parent)
{% endhighlight %}

在getView中如果有可重用的老view(即convertView)的话,可直接使用，如果老的为null,代码中调用newView来新建view，通过bindView来绑定数据，以上三个函数都是由ContactItemListAdapter来实现。

通过调用ListView::setOnScrollListener来设置当用户上拉下拉ListView所需要做的处理，传进去的是ContactItemListAdapter对象，因为前文已述：ContactItemListAdapter还继承了OnScrollListener接口，ContactItemListAdapter实现了：

{% highlight java %}
public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount)
public void onScrollStateChanged(AbsListView view, int scrollState)
{% endhighlight %}

## 关于查询联系人

在ContactsListActivity类中有一个对象，如下定义：

{% highlight java %}
private QueryHandler mQueryHandler;
{% endhighlight %}

QueryHandler有如下继承关系：

![QueryHandler](\images\article\QueryHandler.PNG  "Android Contact")

AsyncQueryHandler顾名思义就是异步的对数据库进行操作的封装，在Contacts中其他部分多次用到。我们可以调用如下接口实现对数据库的操作：

{% highlight java %}
public final void cancelOperation (int token)
public void handleMessage (Message msg)
public final void startDelete (int token, Object cookie, Uri uri, String selection, String[] selectionArgs)
public final void startInsert (int token, Object cookie, Uri uri, ContentValues initialValues)
public void startQuery (int token, Object cookie, Uri uri, String[] projection, String selection, String[] selectionArgs, String orderBy)
public final void startUpdate (int token, Object cookie, Uri uri, ContentValues values, String selection, String[] selectionArgs)
{% endhighlight %}

我们认为AsyncQueryHandler里实现了异步操作，如果我们调用以上除了handleMesssage以外的接口都不会给界面带来阻塞感。在ContactsListActivity中,我们的QueryHandler只自己实现了onQueryComplete，当每次查询结束的时候，这个回调函数可以根据实际情况更新Cursor数据。

当ContactsListActivity在onResume，onRestart，以及ContentProvider状态或内容改变的时候(ContactItemListAdapter::onContentChanged与ContactsListActivity::checkProviderState)都会调用查询函数startQuery，查询出来的Cursor交给ContactItemListAdapter中就可以了，前文说过的。

在onResume调用了registerProviderStatusObserver，注册了ContentObserver来处理当ContentProvider对象变化时
所需要做的处理，所以有如下定义的ContentObserver对象：

{% highlight java %}
 private ContentObserver mProviderStatusObserver = new ContentObserver(new Handler()) {

        @Override
        public void onChange(boolean selfChange) {
            checkProviderState(true);
        }
    };
{% endhighlight %}

当ContentProvider对象有了变化，就会调用上述onChange。当然前提是必须运行registerProviderStatusObserver：

{% highlight java %}
private void registerProviderStatusObserver() {
        getContentResolver().registerContentObserver(ProviderStatus.CONTENT_URI,
                false, mProviderStatusObserver);
    }
{% endhighlight %}

这样就可以达到UI同步数据库的改变了。

## SimContactsActiviy

联系人应用中必然会有管理SIM卡联系人的功能，这个就是SimContactsActivity,继承关系如下：

![SimContactsActivity](\images\article\SimContactsActivity.PNG  "Android Contact")

ContactsADNList负责和SIM卡联系人数据操作，通过ContentProvider来实现，ContentProvider是个跨进程的公用接口，比如通过ContentProvider可以操作联系人的SQLite数据库，也可以通过它实现视频数据播放的服务。不同的ContentProvider服务是通过传进去不一样的URI来实现的，ContactsADNList对SIM卡操作的URI是：

<pre>
content://icc/adn
</pre>

和ContactsListActiviy类似，ContactsADNList也用了AsyncQueryHandler来封装，调用上文提到的接口来实现数据操作.ContactsADNList实现了更多的回调函数：

{% highlight java %}
protected void onDeleteComplete (int token, Object cookie, int result)
protected void onInsertComplete (int token, Object cookie, Uri uri)
protected void onQueryComplete (int token, Object cookie, Cursor cursor)
protected void onUpdateComplete (int token, Object cookie, int result)
{% endhighlight %}

SimContactsActivity则专注于和用户打交道，无论用户要增加数据，删除数据还是要查询数据，都是通过基类ContactsADNList所提供的方法，但是具体的流程逻辑都是SimContactsActivity自己实现。值得一说的是整体删除所有SIM卡记录以及将SIM记录拷贝到，他们都用到了线程，线程对象基于Thread：

![DeleteAllSimContactsThread](\images\article\DeleteAllSimContactsThread.PNG  "Android Contact")

public void run()是执行线程的函数：

{% highlight java %}
@Override
         public void run() {
             ContentResolver resolver = getContentResolver();
             mCursor.moveToPosition(-1);

             if (mCursor != null && mCursor.getCount() != 0) {
                 mCheckedContactsUri.clear();
                 mCursor.moveToPosition(-1);
                 while (mCursor.moveToNext()) {
                     final String tag = mCursor.getString(0);
                     final String number = mCursor.getString(1);
                     mCheckedContactsUri.add(tag);
                     mCheckedContactsUri.add(number);
                 }

                 int count = mCheckedContactsUri.size();
                 for (int i = 0; i < count; i = i + 2) {
                     if(mCanceled) {
                         break;
                     }

                     Message msg = mHandler.obtainMessage();
                     msg.what = MSG_DELETING_NAME;
                     msg.obj = mCheckedContactsUri.get(i);
                     msg.sendToTarget();
                     resolver.delete(mSimUri, "tag='" + mCheckedContactsUri.get(i) + "' AND number='" + mCheckedContactsUri.get(i + 1) + "'", null);
                     mProgressDialog.incrementProgressBy(1);
                 }

                 Message msg = mHandler.obtainMessage();
                 msg.what = MSG_INVALIDATE_VIEWS;
                 msg.sendToTarget();
             }
                
             mCheckedContactsUri.clear();

             if(mProgressDialog != null) {
                 mProgressDialog.dismiss();
             }

         }

{% endhighlight %}

为了实现删除每个Items时可以和UI实时交互，函数中用了消息机制。在SimContactsActivity类中定义了Handler对象：

{% highlight java %}
private Handler mHandler;
{% endhighlight %}

在新建Handler时，实现handleMessage，来处理线程中发来的消息，而在上述线程函数中，通过：

{% highlight java %}
Message msg = mHandler.obtainMessage();
{% endhighlight %}

从消息池获取消息，其实我理解就是获取内存，然后通过msg.sendToTarget()来发送消息，然后在handleMessage中处理：

{% highlight java %}
mHandler = new Handler() {
             public void handleMessage(Message msg) {
                 if(msg.what == MSG_INVALIDATE_VIEWS) {
                     mCursor = SimContactsActivity.this.managedQuery(mSimUri, COLUMN_CONTACTS,
                         null, null, null);
                     mSimCardContactsCount = mCursor.getCount();
                     setAdapter();
                 } else if (msg.what == MSG_DELETING_NAME) {
                     CharSequence message = getString(R.string.deleting_multiple);
                     String name = (String)msg.obj;
                     mProgressDialog.setMessage(message + "\n  " + name);
                 }
             }
         };
{% endhighlight %}


# 参考资料

[http://developer.android.com/index.html](http://developer.android.com)