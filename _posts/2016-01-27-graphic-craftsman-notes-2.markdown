---
layout: post
title:  图形匠人笔记1—MIT 6.837之assignment 1--Ray Casting
category: wheel
---

# 前言 #

这是MIT Graphic的第二个编程作业，其实写此文时，我已经做到了第六个作业。但是写技术笔记远远比写代码要麻烦得多，当然相应的收获也会更大。言归正传，这个作业主要要求以下几个方面：

* 理解Ray tracing的基本方法，并且用面向对象的方法设计实现Ray tracing的基础架构。
* 实现基本的3D的圆球体类，即Sphere。
* 实现平行投影的类，即OrthographicCamera。    

# Ray Tracing 基本原理 #

按我目前肤浅的理解，目前图像学有两大渲染方式，即实时渲染(_Real time rendering_)和光线跟踪(_Ray Tracing_)。虽然他们背后的数学原理大同小异，但是实现起来确实有不同。前者为了达到Real time的要求，更多得关注在性能上，很多计算都搬到了硬件里去做，GPU就是在这个背景下应运而生。他的渲染方式主要是将需要绘制的模型的三维坐标作为输入，加上材质，颜料，光照，经过3D到2D的变换矩阵，还加上深度信息(Z-buffer), 绘制到一块2D区域上，表现为一个多边形，最后送到屏幕上显示出来，总体流程是从Objects到Pixels。而Ray Tracing不考虑性能，更多的考虑如何绘制的逼真，所以反其道而行之，即从Pixels到Objects的过程，好莱坞的CGI电脑特效就是用的这种方法。这种方法更多的应用了光线传递的原理，即我们人眼看到的东西都是因为有光纤照到了眼睛里。如下图：

![lighttoeye.png](/images/notes/mit_graphic/lighttoeye.png  "lighttoeye.png")

所以我们的基本方法就是在3D坐标系统中选一个观察点即照相机(_Camera_)，当然摄像机会定义一个2D的显示区域，再加上输出图片分辨率信息，我们就能确认每个像素点的坐标，然后我们遍历每一个像素坐标，根据照相机的种类(后面的章节会提到)，生成射线，计算射线与场景里的物体是否有交点，没有交点则给该像素绘上背景色，有交点的话则综合考虑这一点的光照，以及物体的材质等因素，得到该点颜色值即RBG值，然后着色。基本算法伪码为：

    for each pixel do
        compute viewing ray
        find first object hit by ray and its surface normal n
        set pixel color to value computed from hit point, light, and n

# 光线类与圆球类的实现 #

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

上面函数中，除了t是标量以外其他都是三维向量，其中`\(\vec{o}\)`虽然不表示方向，但是他表示三维坐标系中的一个点，也是由三个标量组成，所以可以看做为向量。

神说：要有球。

有了光线，必须还要有球来被光纤普照。为了软件的扩展性，我们必须用面向对象(Object Oriented)的方法来设计球体，因为我们的渲染程序不仅只处理圆球体，还要处理其他物体，比如平面体，正方体，长方体等，所以我们需要一个基类来抽象出所有物体的共同的属性和方法。我们的基类叫Object3D，定义如下：

{% highlight cpp %}
class Object3D
{
public:
    Object3D(Material * m)
        :mMaterial(m)
    {
    }

    Object3D()
    {
        mMaterial = NULL;
    }

    virtual ~Object3D()
    {
    }

    virtual bool intersect(const Ray &r, Hit &h, float tmin) = 0;
protected:
    Material * mMaterial;
};
{% endhighlight %}

然后不同的物体，比如球体，三角形去继承这个基类，实现不同的代码流程。比如Ray Tracing渲染中最重要的intersect方法，接下来我们要着重分析球体的intersect方法。从基类的函数定义可以知道，intersect的输入参数里有Ray对象，Hit对象以及一个float变量。Ray对象就是用来和球体求交点的光线，交点求出来以后，要把光线与物体的交点上的相关信息存入Hit对象，比如代表交点的t值，交点位于球体上的法线以及物体在交点上的颜色信息，Hit类定义如下：

{% highlight cpp %}
class Hit {
public:
  // CONSTRUCTOR & DESTRUCTOR
  Hit() { material = NULL; }
  Hit(float _t, Material *m) { 
    t = _t; material = m; }
  Hit(const Hit &h) { 
    t = h.t; 
    material = h.material; 
    intersectionPoint = h.intersectionPoint; }
  ~Hit() {}

  // ACCESSORS
  float getT() const { return t; }
  Material* getMaterial() const { return material; }
  Vec3f getIntersectionPoint() const { return intersectionPoint; }
  
  // MODIFIER
  void set(float _t, Material *m, const Ray &ray) {
    t = _t; material = m; 
    intersectionPoint = ray.pointAtParameter(t); }

  void set(float _t, Material *m, const Vec3f &p) {
    t = _t; material = m;
    intersectionPoint = p; }
private: 
  // REPRESENTATION
  float t;
  Material *material;
  Vec3f intersectionPoint;
};
{% endhighlight %}

接下来看看如何求Ray在球体上的交点，其实这是一个高中立体几何问题，即将光线函数代入球体方程。假设球体方程为`\(f(\vec{p})=0\)`，我们将射线函数代作为`\( \vec{p}\)`代入方程中，得到：

$$
f(\vec{p}(t))=0 或者 f(\vec{o} + t\vec{d})=0
$$

稍微知道立体几何的球体方程如下：

`\[
(x-o_x)^2+(x-o_y)^2+(x-o_z)^2-R^2=0
\]`

上面的球体方程是单变量形式，也叫标量形式，我们可以写成向量的形式：

`\[
(\vec{p}-\vec{c}).(\vec{p}-\vec{c})-R^2=0
\]`

任何满足上面方程的点即`\( \vec{p}\)`都在圆球上，另外`\( \vec{c}\)`是球体的中心坐标。如果我们把光射线的函数方程代入圆球体方程，得出t值，我们就能确定了光线和球体的交点，这样球体的intersect接口就可以解决了。

`\[
(\vec{o} + t\vec{d}-\vec{c}).(\vec{o} + t\vec{d}-\vec{c})-R^2=0
\]`

重新调整系数得到：

`\[
(\vec{d}.\vec{d})t^2+2\vec{d}.(\vec{o}-\vec{c})t+(\vec{o}-\vec{c}).(\vec{o}-\vec{c})-R^2=0
\]`

这个方程其实就是一元二次方程，最终可以简写成这个样子：

`\[
At^2 + Bt + C = 0
\]`

我相信学过初中数学的对这个式子再熟悉不过了，只要我们判断一下`\( B^2-4AC\)`是否大于0，或者等于0就能求出交点。大于0一般有两个交点，即光线穿过球体而过，等于0有一个交点，即此光线为球体的切线，初中数学的东西我就不详细叙述了，最后t的解为如下形式：

`\[
t=\cfrac{-B\pm\sqrt{B^2-4AC}}{2A}
\]`

有了数学原理，就可以列代码了，代码也很清楚的表达了我刚才说地东西：

{% highlight cpp %}
bool Sphere::intersect(const Ray &r, Hit &h, float tmin)
{
    Vec3f temp = r.getOrigin() - mCenterPoint;
    Vec3f rayDirection = r.getDirection();

    double a = rayDirection.Dot3(rayDirection);
    double b = 2*rayDirection.Dot3(temp);
    double c = temp.Dot3(temp) - mRadius*mRadius;

    double discriminant = b*b - 4*a*c;

    if (discriminant > 0)
    {
        discriminant = sqrt(discriminant);
        double t = (- b - discriminant) / (2*a);

        if (t < tmin)
            t = (- b + discriminant) / (2*a);

        if (t < tmin || t > T_MAX)
            return false;

        h.set(t, mMaterial, r);
        return true;
    }

    return false;
}
{% endhighlight %}

# 平行投影 #

神說：要有人。

这个人就是观察者，即抽象为观察者，我们设计的类叫Camera，也用到了所谓的面向对象，


# 尾声 #

# 参考资料 #

1. [Scratchapixel 2.0](http://www.scratchapixel.com "Scratchapixel 2.0").


