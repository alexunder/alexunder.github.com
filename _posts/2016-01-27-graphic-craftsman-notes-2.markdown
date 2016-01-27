---
layout: post
title:  图形匠人笔记1—MIT 6.837之assignment 1--Ray Casting
category: wheel
---

#前言#

这是MIT Graphic的第二个编程作业，其实写此文时，我已经做到了第六个作业。但是写技术笔记远远比写代码要麻烦得多，当然相应的收获也会更大。言归正传，这个作业主要要求以下几个方面：

* 理解Ray tracing的基本方法，并且用面向对象的方法设计实现Ray tracing的基础架构。
* 实现基本的3D的圆球体类，即Sphere。
* 实现平行投影的类，即OrthographicCamera。

#Ray Tracing 基本原理#

按我目前肤浅的理解，目前图像学有两大渲染方式，即实时渲染(_Real time rendering_)和光线跟踪(_Ray Tracing_)。虽然他们背后的数学原理大同小异，但是实现起来确实有不同。前者为了达到Real time的要求，更多得关注在性能上，很多计算都搬到了硬件里去做，GPU就是在这个背景下应运而生。他的渲染方式主要是将需要绘制的模型的三维坐标作为输入，加上材质，颜料，光照，经过3D到2D的变换矩阵，还加上深度信息(Z-buffer), 绘制到一块2D区域上，表现为一个多边形，最后送到屏幕上显示出来，总体流程是从Objects到Pixels。而Ray Tracing不考虑性能，更多的考虑如何绘制的逼真，所以反其道而行之，即从Pixels到Objects的过程，好莱坞的CGI电脑特效就是用的这种方法。这种方法更多的应用了光线传递的原理，即我们人眼看到的东西都是因为有光纤照到了眼睛里。如下图：

![alt text](/lighttoeye.png"lighttoeye.png")

![lighttoeye.png](/home/xuchenyu/mohism/github/X-toys/graphic/assighment1/note/lighttoeye.png  "lighttoeye.png")

所以我们的基本方法就是在3D坐标系统中选一个观察点即照相机(_Camera_)，当然摄像机会定义一个2D的显示区域，再加上输出图片分辨率信息，我们就能确认每个像素点的坐标，然后我们遍历每一个像素坐标，根据照相机的种类(后面的章节会提到)，生成射线，计算射线与场景里的物体是否有交点，没有交点则给该像素绘上背景色，有交点的话则综合考虑这一点的光照，以及物体的材质等因素，得到该点颜色值即RBG值，然后着色。基本算法伪码为：

    for each pixel do
        compute viewing ray
        find first object hit by ray and its surface normal n
        set pixel color to value computed from hit point, light, and n

#圆球类的实现以及总体渲染流程#

从现在开始，我们将用代码将Ray Tracing的奥妙展示出来。

神说：要有光。于是便有了光。

在Ray Tracing中光线与光源有区别，光源我们后面的章节再讨论。我们定义光线为从一个点出发的一个向量，所以光线类Ray的数据成员有一个3D坐标中的起始点，还有一个表示方向的向量，其实类Ray的实现，assignment都提供了，我就在下面列一下：

{% highlight cpp %}
class Ray {
public:
  // CONSTRUCTOR & DESTRUCTOR
  Ray () {}
  Ray (const Vec3f &orig, const Vec3f &dir) {
    origin = orig; 
    direction = dir; }
  Ray (const Ray& r) {*this=r;}

  // ACCESSORS
  const Vec3f& getOrigin() const { return origin; }
  const Vec3f& getDirection() const { return direction; }
  Vec3f pointAtParameter(float t) const {
    return origin+direction*t; }
private:
  // REPRESENTATION
  Vec3f origin;
  Vec3f direction;
};
{% endhighlight %}

其中有一个成员函数pointAtParameter引入了一个变量t，这个t很关键，是用来表示与物体交点的参数。其实这个光线类表示了一个以t为单变量的函数：

$$
\vec{p}(t)=\vec{o} + t\vec{d}
$$

#平行投影#

#尾声#

#参考资料#

1. [Scratchapixel 2.0](http://www.scratchapixel.com "Scratchapixel 2.0").


