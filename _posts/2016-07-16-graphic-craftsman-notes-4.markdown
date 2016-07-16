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

其中`\(C_{l}\)`是光源的颜色值，两个向量，其中`\(\vec{e}\)`是目标点到眼睛的方向，`\(\vec{r}\)`是反射向量
# 尾声 #

# 参考 #
