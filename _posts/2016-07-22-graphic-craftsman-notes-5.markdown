---
layout: post
title: 图形匠人笔记4—MIT 6.837之assignment 4-- 阴影(Shadow),反射(Refletion)以及折射(Refraction)的实现
category: graphic
---

# 前言 #

阴影，反射和折射都是常见的物理现象，而且可以直接被人类的眼睛感知到，所以必须在图形学里有相应的模拟方式。为了更直观的在代码中实现这三种光学现象，首选得引入一个新的类来方便的应用递归大法。然后再从原理阐述，数学计算以及代码实现几个方面分别探讨这三种光学现象。

# RayTracer类的定义 #

之前都是直接在渲染函数里的双循环里计算每个点的颜色值，为了利用递归方法，必须放到一个函数里去做。所以增加了RayTracer类，所以这里只罗列类的定义，具体实现还是在后面阐述。

{% highlight cpp %}
class RayTracer
{
public:
    RayTracer(SceneParser *s, int max_bounces, float cutoff_weight, 
              bool shadows);
    Vec3f traceRay(Ray &ray, float tmin, int bounces, float weight, 
                           float indexOfRefraction, Hit &hit) const;
private:
    Vec3f mirrorDirection(const Vec3f &normal, const Vec3f &incoming) const;
    bool transmittedDirection(const Vec3f &normal, const Vec3f &incoming,
            float index_i, float index_t, Vec3f &transmitted) const;
private:
    SceneParser * mParser;
    int mBounces;
    int mCutoffWeight;
    bool mRenderShadow;
};
{% endhighlight %}

需要特别注意的是，构造函数中的max_bounces是用来控制递归的深度，以防一直递归下去耗尽系统资源。在主渲染函数里，就变得异常清爽：

{% highlight cpp %}
for (j = 0; j < height; j++)
    for (i = 0; i < width; i++)
    {           
        fprintf(stderr, "Render,pixel at x=%d, y=%d\n", i, j);                             
        float u = (i + 0.5) / width;                                                       
        float v = (j + 0.5) / height;                                                      
        Vec2f p(u, v);
        Ray r = pCamera->generateRay(p);                                                   

        int bounces = 0;                                                                   
        Hit hit;                                                                           
        Vec3f pixelcolor = pTracer->traceRay(r, 0.0, bounces, weight, airRefraction, hit); 

        pixelcolor.Clamp();
        outImg.SetPixel(i, j, pixelcolor);                                                 
    }
{% endhighlight %}

# 阴影 #

阴影是光在传播过程中，由于物体间的相互遮挡，其中的一个物体挡住了光的传播，从而在光线的相反方向上处于靠后面的物体接受的光辐射量减少，形成了阴影的效果。具体实现过程不需要数学计算，所以我在这里先大体描述一下算法过程：

1. 像正常光线追踪的渲染方法一样，生成一个射线对象，即Ray objcet。
2. 如果此射线与场景中的任意物体（只有一个物体，因为只取处在最前面的物体）有交集，计算出交点。
3. 遍历场景中的光源体从交点出发，方向为当前光源的方向，形成一个新的Ray对象，再查看场景中是否有与此Ray对象有交点的物体。
4. 如果有交点，则说明这个光源方向上已经被其他物体遮挡，形成阴影，则放弃当前的分支，什么也不做。如果没有，则继续传统的渲染。

代码实现为:

{% highlight cpp %}
bool ishit = false;
ishit = objGroups->intersect(ray, hit, tm);

if (ishit)
{
    //The main
    if (bounces == 0)
        RayTree::SetMainSegment(ray, 0, hit.getT());

    Material * pM = hit.getMaterial();
    Vec3f normal = hit.getNormal();
    Vec3f point = hit.getIntersectionPoint();

    Vec3f diffuseColor = pM->getDiffuseColor();
    pixelColor = diffuseColor * ambientLight; 

    int k;
    for (k = 0; k < numberLights; k++)
    {
        Light * plight = mParser->getLight(k);

        Vec3f lightDir;
        Vec3f lightColor;
        float distance;
        plight->getIllumination(point, lightDir, lightColor, distance);
        float d = lightDir.Dot3(normal);

        if (d < 0)
            d = 0.0;

        Vec3f tempColor = lightColor * diffuseColor;
        tempColor *= d;
        tempColor += pM->Shade(ray, hit, lightDir, lightColor);

        if (mRenderShadow)
        {
            Ray shadowRay(point, lightDir);
            Hit shadowHit;
            //Be careful, tm should be larger than one
            if (!objGroups->intersect(shadowRay, shadowHit, tm))
            {
                pixelColor += tempColor;
            }
            else
            {   //Shadow ray debug
                RayTree::AddShadowSegment(shadowRay, 0, shadowHit.getT());
            }
        }
        else
        {
            pixelColor += tempColor;
        }
    }
{% endhighlight %}

关键的地方在if (mRenderShadow)的case中，如果新形成的射线shadowRay没有和其他物体产生交点，就将此光源的Phong Shading的结果加到最终颜色值里。

# 反射 #

反射是光线照到特殊材料上后将大部分光线又反射出去。其实图形学中，尤其是物理渲染中，对于反射有很多数学模型，比如BRDF(Bidirectional Reflectance Distribution Function), BSSRDF(Bidirectional Scattering-surface reflectance distribution function)。但是这些复杂的计算模型，我还没有研习到，所以在这里只是实现一个最简单的反射绘制方法。

## 数学原理 ##

最关键的一步是求出反射光线的向量,一般的反射情景如图：

![shad2-cosinespec.png](http://scratchapixel.com/images/upload/shading-intro2/shad2-cosinespec.png  "shad2-cosinespec.png")

我们最终目的是要求出向量`\(\vec{R}\)`的方向。根据反射的物理规律，光的入射角和反射角是一样的，即入射光向量`\(\vec{L}\)`与相应点的法向量的角度和反射向量与法线的角度一样如图：

![reflective_vector.png](/images/notes/mit_graphic/reflective_vector.png "reflective_vector.png")

如上图所示，假设分别有单位法向量`\(\vec{N}\)`，单位入射向量`\(\vec{L}\)`， 以及反射向量`\(\vec{R}\)`。需要注意的是，我们的计算模型中，入射向量是指向光源的向量。具体计算方法就是，将入射向量和反射向量末端连接，形成一个新的向量`\(\vec{X}\)`，方向`\(\vec{R}\)`指向`\(\vec{L}\)`，这样就组成了一个等腰三角形，根据向量加法，可以得到：

$$
\vec{L}-\vec{R}=\vec{X}
$$

$$
\vec{R}=\vec{L}-\vec{X}
$$

然后，左边半个三角形用来协助计算`\(\vec{X}\)`。在这个小三角形当中，因为`\(\vec{L}\)`是单位长度的，所以可以利用夹角`\(\alpha\)`来表示和法向量重合的这个向量，根据三角函数，与法向量重合的这一边的长度为`\(\cos{\alpha}\)`,而`\(\cos{\alpha}\)`的值可以通过`\(\vec{L}\)`与`\(\vec{N}\)`的点乘获得：

$$
\cos{\alpha}=\vec{L}.\vec{N}
$$

可以得到与法向量重合的那条边的向量表达，即`\((\vec{N}.\vec{L})\vec{N}\)`

$$
\vec{X}=2(\vec{L}-(\vec{N}.\vec{L})\vec{N})
$$

最后得到反射向量的计算公式：

$$
\vec{R}=2(\vec{N}.\vec{L})\vec{N}-\vec{L}
$$

## 代码实现 ##

首先根据Assignment描述，专门写一个函数来计算反射光的向量，即用上面推导出来的计算方式。

{% highlight cpp %}
Vec3f RayTracer::mirrorDirection(const Vec3f &normal, const Vec3f &incoming) const
{
    Vec3f N(normal);
    Vec3f L(incoming);
    N.Normalize();
    L.Normalize();
    float d = N.Dot3(L);
    Vec3f reflection = (L - 2*d*N);
    return reflection;
}
{% endhighlight %}

因为入射向量是和前面的数学演算是相反的，所以代码稍有不同。求出发射向量后即可继续构造一个Ray对象，继续递归调用traceRay, 返回的颜色向量还要乘以反射颜色向量，最终加到像素的颜色。

{% highlight cpp %}
Vec3f rc = pM->getReflectiveColor();
//Process the reflective situation
if (rc.Length() > 0.0 && bounces < mBounces)
{
    Vec3f incomingDir = ray.getDirection();
    Vec3f reflectiveDir = mirrorDirection(normal, incomingDir);
    Ray reflectiveRay(point, reflectiveDir);
    Hit reflectiveHit;
    pixelColor += rc*traceRay(reflectiveRay, tm, bounces + 1, weight*rc.Length(),
        indexOfRefraction, reflectiveHit);

    float t = reflectiveHit.getT();
    if (t < epsilon)
        t = 10000.0;

    RayTree::AddReflectedSegment(reflectiveRay, 0, t);
}

{% endhighlight %}

这是局部处理反射的代码，我们最好从全局把握一下流程，在这里可以利用一下文学编程(Literate programming)

{% highlight cpp %}
bool ishit = false;
ishit = objGroups->intersect(ray, hit, tm);

if (ishit)
{
    <Get the Hiy info and related materials info>
    <Calculate the product between diffuse color and ambient color>

    int k;
    for (k = 0; k < numberLights; k++)
    {
        <Do shading calculatations for every single light source>
        <Shadow processing for every single light source>
    }

    <Calculating the reflective situation>
    <Calculating the refraction>
}
{% endhighlight %}

# 尾声 #

前段时间，一直以为自己搞定了折射，等到写笔记的时候，才发现根本就没正确实现，于是返回头恶补，调试，拖了三四天终于搞定了，但是迫于折射计算的复杂性，我决定单开一篇note来记录。按照尾声的惯例，我放几个生成的效果图。

先来一个阴影效果：

![output4_13.png](/images/notes/mit_graphic/output4_13.png "output4_13.png")

再来一个阴影加反射：

![output4_04d.png](/images/notes/mit_graphic/output4_04d.png "output4_04d.png")

# 参考 #

1. [Scratchapixel 2.0](http://www.scratchapixel.com "Scratchapixel 2.0").
2. [Mathematics for 3D Game Programming and Computer Graphics, Third Edition](https://www.amazon.com/Mathematics-Programming-Computer-Graphics-Third/dp/1435458869).