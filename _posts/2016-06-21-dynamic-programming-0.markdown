---
layout: post
title: 初识动态规划(Dynamic Programming) 
category: algorithms
---

# 开头 #

因为鄙人不是计算机科学的科班出生，对数据结构与算法的学习力度不够。从大学本科时代的自学入门到现在已经十多年，途中断断续续的学习和实现《Introduction to Algorithms》里的内容，到现在也没彻底刷完，习题就更不用说了。最近开始刷Leetcode的题目，想彻底提高一下编码技艺，谁知碰到Dynamic Programming的题目彻底懵了，于是最近决心要把动态规划这个难啃的骨头啃下来。于是先啃《Introduction to Algorithms》里的DP章节，确实很难理解，硬着头皮看了一遍，刷一些LeetCode上的建议DP题目，确实小有收获。当然我这种中人之资还没有彻底理解DP，写这个就是为了增加博客内容以及加深理解。主要是从一道题目说起，做了几道DP的题目，这是唯一一道我没看相关解题思路就写出Accepted的代码，应该分析一下。

#题目#

A robot is located at the top-left corner of a m x n grid (marked 'Start' in the diagram below).

The robot can only move either down or right at any point in time. The robot is trying to reach the bottom-right corner of the grid (marked 'Finish' in the diagram below).

How many possible unique paths are there?

简单来讲，就是一个m乘以n的矩形格子，从左上角走到右下角总共多少个走法？

![robot_maze.png](http://leetcode.com/wp-content/uploads/2014/12/robot_maze.png  "robot_maze.png")

# 递归解法 #

经过九宫格的列举各个路径的笨办法，一个递归解法很容易被想出来，其实动态规划的第一步也是先写出递归式子。规律还是很容易找到，因为在格子里只能向下相右移动，所以任意一个位置的来路只有两个方向，他上面的格子，以及他左面的格子，上面格子的解加上左面格子的解就是当前位置的解，我们设dp为从(0, 0)当前位置的路径(i, j)的数量，得到一个递归解:

$$
dp[i,j]=dp[i-1, j]+dp[i, j-1]
$$

代码写起来也很容易:

{% highlight cpp %}
int uniquePaths(int m, int n) {
    if (m == 1 || n == 1)
        return 1; 
    else
        return uniquePaths(m - 1, n) + uniquePaths(m, n - 1);
}       
{% endhighlight %}

正如算法导论里说得，直接按递归写出来的代码性能及其低下，因为有很多冗余计算，也就是所谓的Subproblem Overlapping。具体的时间复杂度的计算，可以画出递归树，因为递归到最后，不管第一列还是第一行都是1，所以递归树分裂到有1出现就会停止，所以我们可以得出来，总共有`\(2^(n+1)+2^(n-1)\)`个节点，也可以说是那么多次调用，所以我们可以推出递归方案的时间复杂度为:

$$
T(n)=2^{\max(m, n)}
$$

这个代码当然不能被LeetCode通过。

# 动态规划解法 #

在算法导论里，说到一般的动态规划分四步：

* Characterize the structure of an optimal solution.
* Recursively define the value of an optimal solution.
* Compute the value of an optimal solution.
* Construct an optimal soltion from computed information. 

第一步应该是搞清楚需要优化什么东西，也就是问题的理解，第二步就是写出递归解，就像上文的内容，第三步我的理解是怎么能把重合的子问题计算出来单独处理，最后一步就是通过空间换时间的方式改写递归解。所以我对这个问题的解法就是，分配一个和网格同样大小的二维数组，所谓俄空间换时间，每一个元素都代表从(0, 0)到这个位置(i, j)的解，这样就只需要将二维数组遍历一次，没有冗余的将这个数组每个元素计算出来，数组最后一个元素即是问题的答案。还是利用上面的递归式子`\(dp[i,j]=dp[i-1, j]+dp[i, j-1]\)`来实现代码:

{% highlight cpp %}
int uniquePaths(int m, int n) {
    int * table = (int *)malloc(m*n*sizeof(int));
    memset(table, 0, m*n*sizeof(int));

    int i;
    int j;
    
    for (j = 0; j < m; j++)
    for (i = 0; i < n; i++) {
        if (i == 0 || j == 0) {
            if (i == 0 && j == 0)
                table[0] = 1;
            else if (i == 0)
                table[i + j*n] = table[i + (j - 1)*n];
            else if (j == 0)
                table[i + j*n] = table[i - 1 + j*n]; 
        } else
            table[i + j*n] = table[i - 1 + j*n] + table[i + (j - 1)*n];
    }

    int ret = table[n*m - 1];
    free(table);
    return ret;
}
{% endhighlight %}

成功的将`\(O(2^{max(m,n)})\)`的复杂度降到了`\(O(mn)\)`

# 尾声 #

虽然这是一道在LeetCode里算是中等难度的DP题目，但却是鄙人动态规划之路的第一个里程碑，而且这种明显的分配空间的方法还是比较低级的，如果以后对动态规划(DP)有了更深刻的认识，我将继续写文记录。

#参考引用#

1. [LeetCode](https://leetcode.com/).
2. [Introduction to Algorithms, Third Edition](https://mitpress.mit.edu/books/introduction-algorithms) 