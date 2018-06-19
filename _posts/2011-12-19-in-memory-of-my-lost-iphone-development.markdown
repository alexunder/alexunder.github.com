---
layout: post
title: 纪念我那逝去的iphone开发
category: blog
---

# 前言

我曾经拥有两台iphone,但不是同时拥有，他们先后都离我而去，至今黯然叹息。在那些拥有iphone的岁月里，作为程序员的我当然要寻求为她编程。但是在没有MAC电脑的前提下，这个过程相当艰辛，充满荆棘：首先是越狱；然后为了在ubuntu上编译iphone tool chains的环境，我痛苦的纠结了多次，渡过了不少不眠夜；后来，编译环境弄好后，我移植了我初学程序时在Windows PC上写的俄罗斯方块，调试只能用文件记录，诸多不方便，经过一番折腾，我的tetris终于跑起来了。

正当我准备继续为iphone努力的时候，我的第二个iphone很不幸的丢失了，我不愿意再买一个iphone了，我想我与iphone注定无缘。我的iphone开发之路就此在中道奔殂。为了纪念那逝去的岁月，为了纪念我的辛劳，我将我当初编译tool chains的笔记呈现于此，以及将Tetris的代码push进github: [https://github.com/alexunder/iphonedev](https://github.com/alexunder/iphonedev) ，不求能对别人有所帮助，但求对得起自己。废话不说了。

# 编译的环境的构建

## Foreword

如果要编译可以在iphone上运行的程序，我们可以有两种选择：

- 使用MAC系统，即是当前的Mac OS X Leopard系统，然后下载iphone sdk。此方法不适合我等geek之辈，装MAC得费不少功夫，虽说这个方法是开发iphone软件的官方推荐方法，但是至少不适合我。
- 开源工具链。这是极为牛逼的黑客通过逆向手段得到的方法，可以在Gnu linux上进行iphone开发。


## Start here

系统准备：我的iphone目前的版本是3.1.2，所以我就要以这个版本为基础构造编译环境。在构造之前，我们需要三样东西：

- 第一个就是iphone_sdk_3.1.2，官方用法需要它，我们也需要它，之前的链接不能用了，所以我就删除了，应该在官网上还能找到。
- 还有一个就是iphone 3.1.2 os firmware，即系统固件3.1.2。
- 在google code上的托管项目中有黑客们建立的一个项目，目的是在linux上构建编译iphone的环境，这是其链接: [http://code.google.com/p/iphonedevonlinux/wiki/Installation](http://code.google.com/p/iphonedevonlinux/wiki/Installation)


## Follow here

我选择的 gnu linux 是久负盛名的Ubuntu 9.10.首先，我们要用svn将编译需要的一些脚本文件check out出来，命令如下：

{% highlight bash %}
mkdir -p ~/Projects/iphone/
cd ~/Projects/iphone/toolchain
svn checkout http://iphonedevonlinux.googlecode.com/svn/trunk/ ./
{% endhighlight %}

在toolchain的目录(即下面都是脚本文件的那一级)下，建立files/firmware目录级：
将iphone_sdk_3.1.2_with_xcode_3.1.4__leopard__9m2809.dmg拷贝到刚才建立的files目录下，将iPhone1,2_3.1.2_7D11_Restore.ipsw拷贝到files下面的firmware目录下。
当然在gnu linux编译 toolchain前，我们还必须将以下软件包安装到我们的 desktop上：

{% highlight bash %}
apt-get install \
  automake \
  bison \
  cpio \
  flex \
  g++ \
  g++-4.3 \
  g++-4.3-multilib \
  gawk \
  gcc-4.3 \
  git-core \
  gobjc-4.3 \
  gzip \
  libbz2-dev \
  libcurl4-openssl-dev \
  libssl-dev  \
  make \
  mount \
  subversion \
  sudo \
  tar \
  unzip \
  uuid \
  uuid-dev \
  wget \
  xar \
  zlib1g-dev \
{% endhighlight %}

如果你的计算机是64位的，需要刻意安装以下软件包：

{% highlight bash %}
apt-get install g++-4.3-multilib gcc-4.3-multilib gobjc-4.3-multilib
{% endhighlight %}

如果我们的Ubuntu上gcc的版本不是4.3的话，我们要用回4.3。如果你执意要用4.4的话，在编译toolchain的时候会出现很多编译错误。解决方案就是转到/usr/bin目录下，删除gcc, g++,  gobjc, 运行以下几个ln命令建立和所需要版本编译器的链接：

{% highlight bash %}
ls  -sf  gcc-4.3    gcc
ls  -sf  g++-4.3    g++
ls  -sf  gobjc-4.3  gobjc 
{% endhighlight %}

然后转到toolchain目录下，执行 以下命令：

{% highlight bash %}
chmod u+x toolchain.sh
./toolchain.sh all
{% endhighlight %}

当然这样执行也许会很完美，一步到位生成了我们需要的工具链。但是在大多情况下，执行过程会有很多错误，所以我们最好一步一步来，分解的命令如下：

{% highlight bash %}
./toolchain.sh headers
./toolchain.sh firmware
./toolchain.sh darwin_sources
./toolchain.sh build
{% endhighlight %}

这些命令的执行有的非常耗时。

如果遇到"We need the decryption key for 018-6028-014.dmg."的问题，可以将toolchain.sh相关位置改成以下模样：

[Tool Chain](images/article/iphone_tool_chain_sh.png  "Iphone Development")

其实就是将以下字符串赋给DECRYPTION_KEY_SYSTEM：
DECRYPTION_KEY_SYSTEM= “a8a886d56011d2d98b190d0a498f6fcac719467047639cd601fd53a4a1d93c24e1b2ddc6”。

## Going here

当./toolchain.sh build执行完毕，没有错误的话，我们的工具链就编译完毕了，在toolchain\toolchain目录下生成了几个目录：bld, pre,src,sys这几个目录，包括了编译iphone软件所需要的工具，代码，头文件，以及库，在pre\bin下有我们所需要的交叉编译器，如果要编译toolchain\apps\HelloToolchain目录下的makefile 文件，还必须将交叉编译器arm-apple-darwin9-gcc的路径加入到Linux的环境变量中去，比如加入到/home/xxx/.profile文件当中如下命令：

{% highlight bash %}
export PATH=/home/xxx/toolchain/toolchain/pre/bin:$PATH
{% endhighlight %}

改好之后，如果编译makefile的时候有如下链接错误：
'ld: library not found -lobjc' 或者 'ld: framework not found Foundation',
说明我们没有准备好iphone相关的链接库，解决方法如下：

{% highlight bash %}
In ~/ toolchain/toolchain/sys, rename folder System to System2
In ~/toolchain/toolchain/sys/usr, rename folder lib to lib2
Copy folder ~/toolchain/sdks/iPhoneOS3.1.2.sdk/System to ~/toolchain/toolchain/sys
Copy folder ~ /toolchain/sdks/iPhoneOS3.1.2.sdk/usr/lib to ~ /toolchain/toolchain/sys/usr
{% endhighlight %}

编译成功后，需要通过ssh的方式通过wifi网络（这是目前唯一的最方便的方式了）将程序目录上传到iphone 上。目录命名方式为：yourappname.app,目录中至少有程序文件，info.plist, default.png, info.plist是xml格式的程序信息描述文件。

上传之后，还需要程序签名，程序名字叫ldid，需要装在iphone上。安装方式简述下，就是先通过iphone里的Cydia安装apt-get，mobile terminal工具，然后通过apt-get install ldid 来安装。

在makefile中应该下上以下内容，尤其是第一个命令，权限不对，程序图标就是不会出现在iphone 上：

{% highlight bash %}
chmod -R 755 /Applications/HelloToolchain.app
ldid -S /Applications/HelloToolchain.app/HelloToolchain
{% endhighlight %}

## End here

HelloToolchain成功运行，这篇笔记就该告一段落了。

# Programming 笔记


## Calling the object’s member function.

Given an object named myWidget, a message can be sent to its powerOn method this way:

{% highlight objective-c %}
returnvalue = [ mywidget poweron ];
{% endhighlight %}

The C++ equivalent of this might look like:

{% highlight cpp %}
returnValue = myWidget->powerOn();
{% endhighlight %}

The C equivalent might declare a function inside of its flat namespace:

{% highlight c %}
returnValue = widget_powerOn(myWidget);
{% endhighlight %}

## Class and method declarations.

{% highlight objc %}
#import <Foundation/Foundation.h>

@interface MyWidget : BaseWidget
{
    BOOL isPoweredOn;
    @private float speed;
    @protected float mass;
    @protected float gyroscope;
}
+ (id)alloc;
+ (BOOL)needsBatteries;
- (BOOL)powerOn;
- (void)setSpeed:(float)_speed;
- (void)setSpeed:(float)_speed withMass:(float)_mass;
- (void)setSpeed:(float)_speed withGyroscope:(float)_gyroscope;
@end
{% endhighlight %}


预处理命令#import代替#include，用来引用所需要的头文件。
@interface命令定义的interface 类似于C++中的类的声明：


{% highlight objc %}
@interface xxxxxx
	xxxxx
@end
{% endhighlight %}

+号表示是non-static member function，-表示static member function，member function 在括号外面声明。

## About UIKit


### 创建windows.

获取iphone的全屏幕的尺寸数据：

{% highlight objc %}
CGRect windowRect = [ UIHardware fullScreenApplicationContentRect ];
{% endhighlight %}
           
UIHardware是framework中的object,通过调用fullScreenApplicationContentRect来获取整个屏幕的矩形数据。

通过矩形数据来创建windows

{% highlight objc %}
UIWindow *window = [ UIWindow alloc ];
[ window initWithContentRect: windowRect ];
{% endhighlight %}


### 创建view.

{% highlight objc %}
CGRect viewRect = [ UIHardware fullScreenApplicationContentRect ];
viewRect.origin.x = viewRect.origin.y = 0.0;
UIView *mainView = [ [ MainView alloc ] initWithFrame: viewRect ];
{% endhighlight %}

### 显示view.

{% highlight objc %}
[ window setContentView: mainView ];
[ window orderFront: self ];
[ window makeKey: self ];
[ window _setHidden: NO ];
{% endhighlight %}

h4. 对view的继承.

{% highlight objc %}
@interface MainView : UIView
{

}

- (id)initWithFrame:(CGRect)rect;
- (void)dealloc;
@end
{% endhighlight %}

initWithFrame是初始化的时候调用的函数，dealloc是退出析构的时候调用。这两个函数是继承类必须继承的函数。


# 参考资料

1 [http://code.google.com/p/iphonedevonlinux/wiki/Installation](http://code.google.com/p/iphonedevonlinux/wiki/Installation)
2 iphone developer guide