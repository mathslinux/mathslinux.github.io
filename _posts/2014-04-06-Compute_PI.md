---
layout: post
tag: Tools
date: '\[2014-04-06 日 19:40\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 简单计算 PI 的方法
---

这几天搞一个虚拟机自动迁移的测试, 为了满足在特定 CPU
条件下触发迁移的条件, 需要有进程不断的在消耗 CPU 资源,
这类程序当然可以随便乱写, 来个 for (;;;) 之类的就可以完全满足. 但是,
本着专业的精神, 我决定用计算
[π](http://zh.wikipedia.org/wiki/%E5%9C%93%E5%91%A8%E7%8E%87) 来达到消耗
CPU 资源的目的.

学过一点级数展开的都知道下面这个著名的 [π
的莱布尼茨公式](http://zh.wikipedia.org/wiki/%CE%A0%E7%9A%84%E8%8E%B1%E5%B8%83%E5%B0%BC%E8%8C%A8%E5%85%AC%E5%BC%8F):
![](/images/posts/Tools/PI_Leibniz.png)

下面是直接用这个代码的 C++ 测试程序, 当然,
由于我用一个浮点型来保存结果的, 精度不可能搞得离谱, 要获得更高的精度,
麻烦还是用一个大数组吧 :)

``` cpp
#include <math.h>
#include <iostream>
#include <iomanip>

using namespace std;

/**
 * 使用传说中的计算 PI 的莱布尼茨公式:
 * pi/4 = 1 - 1/3 + 1/5 - 1/7 + 1/9 - ... + (-1)^n/(2n + 1)
 * 
 */
#define END 7                   /* 精确到小数点后 7 位 */

int main(int argc, char *argv[])
{
    double ret = 0;
    long long n = 0;
    int mark = 1;
    double precision = pow(10, -END - 1);

    while (1) {
        double f = (double)mark / (2 * n + 1);
        ret += f;
        if ((mark * f) < precision) { /* 计算精度 */
            break;
        }
        n += 1;
        mark = -mark;
    }

    cout << "PI ≈ " << setprecision(9) << ret * 4 << endl;
    return 0;
}
```

另外, 高中时候学过的最简单的:

``` bash
tan(4/π) = 1
=> 4/π = arctan(1)
=> π = 4 * arctan(1)
```

所以可以利用 `glibc` 提供的 API 直接算 PI.

``` cpp
double pi = atan(1) * 4;
```

这还不是最简单, 最简单的算 π 的方法, 如果熟悉 linux 下 `bc`
的童鞋应该想到了, 在 bc 中, 用函数 a (x) 可以计算反正切, 设置变量
`scale` 为想要的精度, 于是用 bc 计算就如此之简单了:

``` bash
$ echo "scale=10;4*a(1)"|bc -l
3.1415926532
```

想要消耗 CPU 资源, 你把 `scale` 设置为 100000 试试.
