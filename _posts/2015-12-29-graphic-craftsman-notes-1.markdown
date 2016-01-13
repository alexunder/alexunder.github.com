---
layout: post
title:  图形匠人笔记0—MIT 6.837之assignment 0
category: wheel
---

#Why I want to write this note?#

写这个笔记有几个原因吧：第一，我写blog效率太低，自2011年建立以来到现在三四年了，不过寥寥十来篇，实在对不住当年开github blog时候的豪情壮志，离每周一篇的速度相差太大，需要迎头赶上；第二，在下做技术（特指软件开发从业时间）已近十年，居然没有特别精深的钻研方向，实在惭愧，当然过去的若干年间也曾上下求索，找寻方向，比如曾经的模式识别，无线通信，由于种种原因都暂且搁置，当然最主要的原因还是自己的持久力很差，面对理解深一层的知识时候的挫败感，无力前行。然而如今我已过而立之年，痛定思痛，终于选择了集兴趣，数学，艺术，趣味性的图形学作为深入研究的方向。具体方法还是得Learning by doing, 之前看了几章PBRT，以及基础书籍，看到中部渐觉吃力，再后来看到了MIT OCW的Computer Graphic的assignments, 于是决定好好编程实现，并且写笔记记录，好好的实践Learning by doing的精神。

#Assignment 0: Iterated Function System#

第一个任务是要实现一个 Iterated Function Systems, 英文直翻译应该叫做函数迭代系统，严格来说是分形数学的一个东西。当然分形图形不是MIT这门课所包含的内容，这个习作工程难度不是很大，主要是让初学者熟悉C++的用法以及后续Ray Tracing主要用到的代码流程，即逐个像素颜色的生成，最后生成一个位图。也正如知乎上的图形大牛所言，图形学的研究与学习可以抛开OpenGL或者DirectX，直面数学原理。

##什么是Iterated Function System##

其实我也不太懂，我先抄一下维基百科吧：

在数学上，迭代系统是一种生成分形(Fractals)的方法，最后形成的结构往往是自相似的。分形很好的模拟了大自然中某些现象或事物，比如雪花，树叶。IFS分形可以以任意纬度绘制，但通常被绘制成2D图形，这也是assignment 0的要求。分形通常由他自身几个拷贝组成，而且每个拷贝都要被一个函数系统所转换，用数学语言来说应该是被一个函数系统所映射，而且这个函数不止一个，他们是一个有限集合：

$$
\{f_i : X \rightarrow X | i = 1, 2, ... , N \}, N \in \mathbb{N}
$$

'\(f_i\)' 是在X度量空间上的一个转换函数。最后主要的IFS计算公式为：

$$
S = \prod_{i = 1}^N f_i(S)
$$

装逼结束，以上的罗列公式除了装逼还有测试Latex的支持。

##代码实现##

上文中的函数系统放到图形学的语境当中，其实就是仿射变换(Affine Transformation)。算法流程其实已经在assignment的描述里被描述的很清楚了：在单位矩形(对角线为(0,0)到(1,1))，随机生成一定数量的2D的点，对每一个点再做若干次仿射变化，相当于上文中的IFS计算公式里的连乘，具体伪代码如下：

    for "lots" of random points (x0, y0)
       for k=0 to num_iters 
           pick a random transform fi
           (xk+1, yk+1) = fi(xk, yk)
       display a dot at (xk, yk)

具体到代码，我们首先读取配置文件，配置文件就是用来表达若干个函数系统的，即仿射变换。每一项包括用到这个变换的概率以及变换矩阵(Transformation Matrix)。比如树叶的配置文件fern.txt:

4
0.03
0.000000 0.000000 0.500000 
0.000000 0.200000 0.000000 
0.000000 0.000000 1.000000 
0.71
0.817377 0.065530 0.091311 
-0.065530 0.817377 0.232765 
0.000000 0.000000 1.000000 
0.13
0.108707 -0.372816 0.445646 
0.279612 0.144943 -0.059806 
0.000000 0.000000 1.000000 
0.13
-0.080250 0.385423 0.540125 
0.289067 0.107000 -0.004534 
0.000000 0.000000 1.000000

第一个行的数字为4，说明总共有四个函数系统，第一个函数被用到的概率是0.03，0.03后面是变换矩阵。而我的工作主要是解析程序命令行以及参数，比如随机点的数量，图片的分辨率还有迭代次数。然后实现一个ifs类，主要用来渲染分形图片, 这里代码细节就不详述了，列一些最关键的渲染图片的函数：

{% highlight cpp %}
void ifs::renderImage(Image &img, int numPoints, int numIters, Vec3f color)
{
    int i;
    int j;
    time_t t;
    srand((unsigned) time(&t));
    int height = img.Height();
    int width = img.Width();

    float coordinate_x = 0.0;
    float coordinate_y = 0.0;

    Vec3f iteratedPoint(0, 0, 0);
    for (i = 0; i < numPoints; i++)
    {
        coordinate_x = rand() / (float)(RAND_MAX);
        coordinate_y = rand() / (float)(RAND_MAX);
        iteratedPoint.Set(coordinate_x, coordinate_y, 0.0);

        for (j = 0; j < numIters; j++)
        {
            int indexProbability = 0;
            float probability =(float)(rand()) /(float)(RAND_MAX);
            int transIndex = getTransformationIndex(probability);
            Matrix m = mArrayMatrices[transIndex];
            m.Transform(iteratedPoint);

        }

        coordinate_x = iteratedPoint[0];
        coordinate_y = iteratedPoint[1];

        if (coordinate_x >= 0 && coordinate_x < 1 &&
                coordinate_y >= 0 && coordinate_y < 1 )
        {
            img.SetPixel(coordinate_x * width, coordinate_y * height, color);
        }
    }
}
{% endhighlight %}

最后生成的图片效果如下：

![alt text](/images/notes/mit_graphic/fern_100.png"fern_100.png") ![alt text](/images/notes/mit_graphic/fern_30.png"fern_30.png")

以及
![alt text](/images/notes/mit_graphic/sierpinski_triangle.png"sierpinski_triangle.png")


##参看资料##

1. [Wikipedia: Iterated function system](https://en.wikipedia.org/wiki/Iterated_function_system "Wikipedia: Iterated function system").
2. [MIT 6.837 Intro to Computer Graphics](http://groups.csail.mit.edu/graphics/classes/6.837/F04/assignments/assignment0/ "MIT 6.837 Intro to Computer Graphics").





