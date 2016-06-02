---
layout: post
title:  图形匠人笔记2—MIT 6.837之assignment 2--Transformations & Additional Primitives
category: graphic
---

# 前言 #

MIT的作业做到第六个的时候，我暂时放弃了，原因有两个，第一，我认为了解了基本的光线求交的原理，以及对光反射，折射，阴影的绘制的了解，基本算是遁入空门；第二，第六个作业里的Voxel rendering还是有些难，所以就偷懒了。为了固化知识，还是硬着头皮来写吧。这个作业里主要包括Perspective Camera的实现，基本的光源渲染，还有三角形和平面的实现。

# PerspectiveCamera的实现 #

其实对比上一章节的平行投影，这个应该叫做透视投影。在这种投影中，光线的传播方向不是平行的，相对于显示矩形上不同的坐标，方向不同。距离显示矩形上中心一段距离的一个位置，我们叫做视点，这一段距离叫做焦距。生成每个像素的射线的方向就是：从这个视点出发到显示矩形上的任意点的向量的方向，如图：

![perspective_camera.png](http://groups.csail.mit.edu/graphics/classes/6.837/F04/assignments/assignment2/perspective_camera.png "perspective_camera.png")

和平行投影一样，我们必须继承基类Camera的几个虚方法：

![classCamera__inherit__graph.png](/images/notes/mit_graphic/classCamera__inherit__graph.png "classCamera__inherit__graph.png")

最关键的是要实现GenerateRay函数，和OrthographicCamera类似，传进来的坐标已经是从屏幕坐标转换到了坐标从(0,0)到(1,1)的矩形空间内。然而和平行投影不同的是，透视投影需要不同的参数来决定：

{% highlight cpp %}
class PerspectiveCamera : public Camera
{
public:
    PerspectiveCamera(Vec3f &center, Vec3f &direction, Vec3f &up, float angle);

    Ray generateRay(Vec2f point);
    void setRatio(float ratio)
    {
        mRatio = ratio;
    }

    CameraType getCameraType()
    {
        return CameraType::Perspective;
    }

    float getTMin() const
    {
        return 0.0;
    }
private:
    Vec3f mCenter;
    Vec3f mDirection;
    Vec3f mUp;
    Vec3f mHorizontal;
    float mAngle;
    float mRatio;
};
{% endhighlight %}

从构造函数可以看出，最基本的数据是需要视点mCenter，方向向量mDirection以及垂直于方向向量向上的向量mUp，在构造函数中，考虑到从文件读出的数据可能不互相垂直，所以要通过一些向量叉乘来得到绝对的三个相互垂直的向量：

{% highlight cpp %}
PerspectiveCamera::PerspectiveCamera(Vec3f &center, Vec3f &direction, Vec3f &up, float angle)
    : mCenter(center), mDirection(direction), mUp(up), mAngle(angle)
{
    float v = mUp.Dot3(mDirection);

    if (v != 0)
    {
        Vec3f temp;
        Vec3f::Cross3(temp, mDirection, mUp);
        Vec3f::Cross3(mUp, temp, mDirection);
    }

    if (mDirection.Length() != 1)
    {
        mDirection.Normalize();    
    }

    if (mUp.Length() != 1)
    {
        mUp.Normalize();    
    }

    Vec3f::Cross3(mHorizontal, mDirection, mUp);

    mRatio = 1.0;
}
{% endhighlight %}

下面我开始关注最重要的GenerateRay函数，这是透视投影的精华所在。首先我们有些设定，前面说过透视投影有焦距，伴随着焦距，还有透视角度（View Angle）, 如图：

![Perspective_angle.png](/images/notes/mit_graphic/Perspective_angle.png "Perspective_angle.png")

这个角度可以是基于显示矩形的宽，也可以是高，这个没有强制，选哪个都行，代码中选择了高作为角度计算。因为我们不能预先确定显示区域的分辨率，所以做了预先的判断，先把GenerateRay的代码列上：

{% highlight cpp %}
Ray PerspectiveCamera::generateRay(Vec2f point)
{
    float x_ndc = point.x();
    float y_ndc = point.y();

    float screenWidth = 0.f;
    float screenHeight = 0.f;

    if (mRatio > 1.f)
    {
        screenWidth = 2 * mRatio;
        screenHeight = 2.f;
    }
    else
    {
        screenWidth = 2.f;
        screenHeight = 2 * mRatio;
    }

    float left = - screenWidth / 2.0;
    float top  = - screenHeight / 2.0;

    float u = x_ndc * screenWidth + left;
    float v = y_ndc * screenHeight + top;
    float near = screenHeight / (2.f * tanf(mAngle / 2.0));

    Vec3f originalDir = near * mDirection + u * mHorizontal + v * mUp;

    if (originalDir.Length() != 0)
    {
        originalDir.Normalize();
    }

    Ray r(mCenter, originalDir);
    return r;
}
{% endhighlight %}

一般情况下，最终的显示区域都是宽长于高，所以选择长或者高为2，对比传进来的长宽比ratio来确定高是2还是2倍的ratio，最后如上图的几何关系我们可以得出焦距的公式为：

$$
d=\cfrac{h}{2\tan(\alpha/2)}
$$

我们知道焦距是视点到显示区域平面的距离，所以显示平面上的点相当于是以视点为原点在mDirection向量方向上的坐标，现在还差mUp和mHorizontal的方向上的坐标就可以得到显示区域上点的完整坐标。和平行投影一样，屏幕坐标映射到了(0,0)到(1,1)的矩形后，还要映射至(-screenWidth/2, -screenHeight/2)，(screenWidth/2, screenHeight/2)的矩形，最后得到在mHorizontal和mUp上的坐标：

{% highlight cpp %}
float u = x_ndc * screenWidth + left;
float v = y_ndc * screenHeight + top;
{% endhighlight %}

最终显示矩形任意一点的坐标可以通过以下代码算出：

{% highlight cpp %}
Vec3f p = mCenter + near*mDirection + u*mHorizontal + v*mUp;
{% endhighlight %}

如代码所示，再将得到的点减去视点，即mCenter就得到了从视点出发到显示屏幕的射线的方向，GenerateRay的代码省略了减去mCenter的操作。

另外，很多渲染软件都是用Perspective Projections Matrix来获取屏幕坐标的，这个矩阵的推导的数学原理和我上面说的一样，但是推导起来稍微麻烦一些，我再专门写文叙述。

# Diffuse shading #

光照的着色处理是图形学中很重要的部分，OpenGL和DirectX的Shadding Language的主要就是实现一些特殊的光照渲染。这里的光照和上篇文章里说的光线还不太一样，光线是我们看到其他的东西的基础，或者说光线是我们绘制场景的基础，然而光照处理是场景中有类似灯，阳光这样的光源，给物体带来的效果。

光照模型有几种，现在先实现一个比较简单的模型，Diffusing Shading也可以叫Directional Lighting。这种光源只有方向，颜色，没有其他属性了。课程网站提供了Light基类，和继承自他的DirectionalLight类：

![classLight__inherit__graph.png](/images/notes/mit_graphic/classLight__inherit__graph.png "classLight__inherit__graph.png")

代码太简单，我就不列了。

接下来，我们看看具体如何计算光照模型。如下图：

![shad-light-beam3.png](http://www.scratchapixel.com/images/upload/shading-intro/shad-light-beam3.png "shad-light-beam3.png")

L是光源照射来的反方向，因为计算向量点乘方便，就直接用反方向的向量计算了。所以对于这一点的光照计算，会用到光源反向量和当前物体点的法向量的点乘，即公式为：

$$
C=C_{r}C_{l}(\vec{n}.\vec{l}).
$$

为了避免得到负的叉乘的值，我们最好加上少许处理：

$$
C=C_{r}C_{l}max(\vec{n}.\vec{l}, 0).
$$

其中Cr是当前物体的颜色，Cl是光源的颜色。另外，我们可能有多个Directional Light, 为了计算一个点的综合光模型，我们有一个统一的公式，代码实现也是基于他：

$$
C_{pixel}=C_{r}C_{a} + \sum_{i=0}^{i=N-1}C_{r}C_{l_{i}}max(\vec{n}.\vec{l_{i}}, 0).
$$

其中Ca是Ambient Color, 也算Ambient shading, 可以理解为整个场景中所有物体的一种平均色，类似蓝天。这个公式假设有N个Directional Light。代码实现就不在这里罗列了，需要注意的是编程的时候不要让RBG的值超越[0, 1]的范围。

# Triangle类和Plane类的实现 #

和Sphere一样，Triangle和Plane都是继承自Object3D，主要是要实现不同的intersect，在这里就不对代码进行分析了，直接把数学原理讨论清楚即可。

## Plane类 ##

Plane相对简单一些，相当于计算一条线与平面的交点。即将光线函数代入球体方程。假设平面方程为`\(f(\vec{p})=0\)`，我们将射线函数代作为`\( \vec{p}\)`代入方程中，得到：

$$
f(\vec{p}(t))=0 或者 f(\vec{o} + t\vec{d})=0
$$

和Sphere一样，要通过上面的方程求出t值，但是对于平面的求交点，可以用更便捷的计算方法，在三维立体几何里，平面的通用方程是:

$$
f(x,y,z)=ax+by+cz+d=0
$$

上个方程中，a, b, c三个系数可以组成平面的法向量，即：

$$
\vec{n}=\begin{bmatrix}
    a \\
    b \\
    c
\end{bmatrix}
$$

这个结论很容易理解，法向量就是垂直于平面的向量，可以知道平面上任意两点组成的向量都和法向量垂直，所以假设任意平面任意两个点：`\((X_{1}, Y_{1}, Z_{1})\)`, `\((X_{2}, Y_{2}, Z_{2})\)`, 代入平面方程得到两个式子：

$$
\begin{align}
aX_{1}+bY_{1}+cZ_{1}+d=0 \\
aX_{2}+bY_{2}+cZ_{2}+d=0
\end{align}
$$

第二个式子减去第一个式子就得到法向量是三个系数a, b, c组成的。所以，平面方程可以改写成向量的模式，假设平面上任意一点`\( \vec{p}\)`，则平面方程可以表示为：

$$
\vec{p}.\vec{n} + d=0
$$

将`\(\vec{o} + t\vec{d}\)`代入得到：

$$
(\vec{o} + t\vec{d}).\vec{n} + d=0
$$

t就可以轻松求出来。

## Triangle类 ##

在图形学中，在处理复杂模型的时候，有一种通用办法是被广泛接受的。那就是将复杂模型的表面用小三角形来分割，类似于积分的思路。所以三角形Triangle类的instersect方法的实现很关键，他是渲染复杂模型的基础。三角形求教点用到了 Barycentric coordinates。可以借助[ScratchaPixel](http://www.scratchapixel.com)的图看一下：

![ScratchPixel-Triangle](http://www.scratchapixel.com/images/upload/ray-triangle/triray2.png  "ScratchPixel-Triangle")

通俗来讲，已知有三角形的三个顶点的坐标：P0, P1, P2，三角形里任意一点可以表示为：

$$
p(b_{1}, b_{2})=(1-b_{1}-b_{2})p_{0}+b_{1}p_{1}+b_{2}p_{2}, b_{1}\ge0, b_{2}\ge0, b_{1}+b_{2}\le1
$$


将参数化的射线`\(\vec{o} + t\vec{d}\)`代入上式得到：

$$
\vec{o} + t\vec{d}=(1-b_{1}-b_{2})p_{0}+b_{1}p_{1}+b_{2}p_{2}
$$

为了方便简洁，设`\(\vec{e_{1}}=p_{1}-p_{0}\)`, `\(\vec{e_{2}}=p_{2}-p_{0}\)`, `\(\vec{s}=o-p_{0}\)`,重新整理得到：

$$
\begin{pmatrix}
-\vec{d} & \vec{e_{1}} & \vec{e_{2}}
\end{pmatrix}
\begin{bmatrix}
    t \\
    b_{1} \\
    b_{2}
\end{bmatrix}=\vec{s}.
$$

这是一个`\(Ax=B\)`型的线性方程，我们可以用克莱默法则(Cramer's rule)得到解，至于这个法则的证明在此处略去，以后单独开文章论述，毕竟直接给出结论显得很突兀。首先我们简化成`\(Ax=B\)`型，即设

$$
A=\begin{pmatrix}
-\vec{d} & \vec{e_{1}} & \vec{e_{2}}
\end{pmatrix},
x=\begin{bmatrix}
    t \\
    b_{1} \\
    b_{2}
\end{bmatrix},
b=\vec{s}
$$

根据法则的定义，矩阵A的秩必须大于0，x才会有唯一解，所以省去一些计算，我们直接得到：

$$
\begin{bmatrix}
    t \\
    b_{1} \\
    b_{2}
\end{bmatrix}=\frac{1}{Det
\begin{pmatrix}
-\vec{d} & \vec{e_{1}} & \vec{e_{2}}
\end{pmatrix}
}\begin{bmatrix}
Det\begin{pmatrix}
\vec{s} & \vec{e_{1}} & \vec{e_{2}}
\end{pmatrix} \\
Det\begin{pmatrix}
-\vec{d} & \vec{s} & \vec{e_{2}}
\end{pmatrix} \\
Det\begin{pmatrix}
-\vec{d} & \vec{e_{1}} & \vec{s}
\end{pmatrix}
\end{bmatrix}
$$

Det(A)表示矩阵A的秩(Determinant)，代码只是一些繁琐的操作，就不在这罗列了。

# 尾声 #
每一次留尾声这个Section，总是想得写点啥，貌似也没啥可写。总之，这次是Latex大爆发，哈哈！

# 参考 #

1. [Scratchapixel 2.0](http://www.scratchapixel.com "Scratchapixel 2.0").
2. [Physically Based Rendering, Second Edition: From Theory To Implementation](http://pbrt.org/).
