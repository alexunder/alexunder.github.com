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

# 折射 #

# 尾声 #

# 参考 #
