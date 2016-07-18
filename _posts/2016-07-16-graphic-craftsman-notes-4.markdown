---
layout: post
title: 图形匠人笔记3—MIT 6.837之assignment 3-- Phong Shading
category: graphic
---

# 前言 #

这个作业还包含了给RayTracer添加OpenGL的要求，暂时不叙述任何有关OpenGL的内容，所以这里之讨论Phong Shading.

# Phong Shading的数学解释 #

Phong Shading是一个姓冯的越南裔美国人发明的计算光照的一种方法，简单来说就是反射光在物件上面的一些效果计算。如图：

![PhongModel.png](/images/notes/mit_graphic/PhongModel.png  "PhongModel.png")

在现实中的效果如下图：

![shad2-plasticball.png](http://www.scratchapixel.com/images/upload/shading-intro2/shad2-plasticball.png  "shad2-plasticball.png")

其计算公式为：

$$
C=C_{l}{max(0, \vec{e}.\vec{r})}^p
$$

其中`\(C_{l}\)`是光源的颜色值，两个向量，其中`\(\vec{e}\)`是目标点到眼睛的方向，`\(\vec{r}\)`是反射向量。p是冯氏指数(Phong exponent)。这个指数决定里如上图的光亮效果。p的不懂取值，产生的效果如下图：

![Phong_Exponent.png](/images/notes/mit_graphic/Phong_Exponent.png  "Phong_Exponent.png")

一般情况下，已知量为光源入射向量，法向量以及目标点到观察者的向量`\(\vec{e}\)`，所以反射向量可以这样算出来：

$$
\vec{r}=2(\vec{l}.\vec{n})\vec{n}-\vec{l}
$$

随后，Blinn–Phong shading model又被提了出来，这个模型更易于计算，这个模型用了新的向量`\(\vec{h}\)`，即：

$$
\vec{h}=\frac{\vec{e}+\vec{l}}{\lvert\vec{e}+\vec{l}\rvert}
$$

最终的计算公式是：

$$
c=c_{l}{(\vec{h}.\vec{n})}^p
$$

# C++实现 #

{% highlight cpp %}
Vec3f PhongMaterial::Shade(const Ray &ray, const Hit &hit, const Vec3f &dirToLight, 
                                                     const Vec3f &lightColor) const
{
    Vec3f eyeDir = ray.getDirection();
    eyeDir.Negate();

    Vec3f eyePlusLight = eyeDir + dirToLight;
    eyePlusLight.Normalize();

    float hn = eyePlusLight.Dot3(hit.getNormal());
    hn = pow(hn, mPhongComponent);

    Vec3f color = lightColor * mHighLightColor;
    color = hn * color;
    return color;
}
{% endhighlight %}

# 尾声 #

这个section是我留着用来逗逼的吗？明明没啥用，每次都有。

# 参考 #

1. [Scratchapixel 2.0](http://www.scratchapixel.com "Scratchapixel 2.0").
2. [Fundamentals of Computer Graphics](https://www.amazon.com/Fundamentals-Computer-Graphics-Fourth-Marschner/dp/1482229390).
