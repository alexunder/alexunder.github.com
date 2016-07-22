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

# 反射 #

# 折射 #

# 尾声 #

# 参考 #
