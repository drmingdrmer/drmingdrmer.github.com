
在上篇
[互联网中对象访问频率的91分布](/tech/2019/10/30/zipf)
我们通过
`90%的流量由10%的内容产生`
这句经验描述，
得出了访问频率的[zipf](https://en.wikipedia.org/wiki/Zipf%27s_law)模型:

<img src="https://www.zhihu.com/equation?tex=f%28k%29%20%3D%20c/k%5Es%5Ctag%7B1%7D%5C%5C" alt="f(k) = c/k^s\tag{1}\\" class="ee_img tr_noresize" eeimg="1">

CDN ([Content delivery network](https://en.wikipedia.org/wiki/Content_delivery_network))
就是一个典型的符合zipf分布的缓存系统:
将缓存服务部署在距离用户最近的上百个地区（CDN边缘机房），
当用户需要访问热门内容时，只需在边缘机房读取，以此达到性能优化。

因为符合zipf模型，
所以，如果一个边缘机房的容量对全量数据的占比变化1%，**会带来每年数十万元的成本变化**。
一个中等规模的CDN系统，调优前后，**可能会有每年几百万的成本差别**。

本文从原理到实践跟大家一起分析下CDN的成本构成，以及zipf如何影响成本。

## 本文结构:

![xxx](https://gitee.com/drdrxp/bed/raw/master-md2zhihu-asset/cdn/8f9f27966ffec647-bigmap.jpg)

<!--more-->

# 文件访问频率的分布规律

按照前文
[互联网中对象访问频率的91分布](/tech/2019/10/30/zipf)
中介绍，
下图是从真实业务中提取的1000个URL访问次数，
按照访问次数从高到低进行排序的zipf曲线:

![xxx](https://gitee.com/drdrxp/bed/raw/master-md2zhihu-asset/cdn/a55f31f9083e2851-1kfile.png)

<center>图1 访问频率分布</center>

基于上图所呈现的访问热度分布，CDN边缘机房会缓存最热的文件
(对应图中橘黄色部分)，
为用户提供更好的访问质量，
而剩下蓝色部分不太频繁被访问的文件，
则按照正常的方式回源站拉取，再返回给用户。

于是就有了由存储/回源构成的CDN的成本:

# CDN成本构成

构成CDN的成本主要由2部分组成:

-   边缘机房中用于存储热文件(橘黄色)的服务器开销;
-   拉取本地不存在的冷文件(蓝色)时，访问内容源站(存储全量数据)的带宽开销。

显然边缘存储量越大，可缓存的内容就越多，
回源率(回源下载带宽/业务总下载带宽)就会越低，因此:

-   边缘存储成本上升，会带来回源带宽成本下降。
-   反之边缘存储成本下调，则会带来回源成本的上升。

接下来我们通过3个步骤，找到一个合适的边缘容量，**让存储和带宽的成本总和达到一个极小值**:

-   量化业务参数;
-   确定业务的模型，也就是zipf中的参数;
-   根据单位成本找出业务场景中的最优边缘容量，以及其对应的最低成本。

# 确定业务参数

一个假想的业务场景和CDN服务配置如下:

-   每个CDN边缘机房对外提供的日常下载总带宽: `20Gbps`，
-   源站中所有文件的**总量**`n`，
-   边缘机房的存储容量占源站文件总量的比值: `e`(edge)，例如(图1)中，`e=10%`，每个边缘机房能存储1/10的内容。
-   回源率`b`(backsource) 是边缘机房消耗的回源带宽跟这个边缘机房下载带宽的比值，例如(图1)中，`e=10%`时，`b=6%`。

![xxx](https://gitee.com/drdrxp/bed/raw/master-md2zhihu-asset/cdn/83f8dfb288d235e5-cdn-arch.jpg)

其中下载带宽和文件总量`n`是业务属性;
`e`和`b`是CDN的属性，不取决于业务，是由CDN提供商选择，并最终决定成本的因素。

> `e`，边缘容量
> 对应(图1)中橘黄色部分的x方向的**宽度**，图中`e`的值是10%。
> 
> `b` 对应(图1)中的蓝色部分，
> 是需要回源的请求，在(图1)中，`b`的值是蓝色部分的**访问次数之和**(不同于`e`，不是宽度)，
> 跟所有文件总访问次数之和的比值。


# 确定业务模型-拟合zipf曲线

业务场景确定后，找出最佳成本的第一步是为这个热度分布曲线建立模型，
以便后面的计算。
根据我们上一篇中介绍过的
[转换成对应的zipf分布的公式](https://drmingdrmer.github.io/tech/2019/10/30/zipf#%E5%A6%82%E4%BD%95%E5%B0%86%E8%AE%BF%E9%97%AE%E8%AE%A1%E6%95%B0%E8%BD%AC%E6%8D%A2%E6%88%90%E5%AF%B9%E5%BA%94%E7%9A%84zipf%E5%88%86%E5%B8%83%E7%9A%84%E5%85%AC%E5%BC%8F)
的方法，找出zipf曲线的2个参数:

把x，y轴分别取对数后，图像会呈现一个线性关系:

<img src="https://www.zhihu.com/equation?tex=log%28f%28k%29%29%20%3D%20log%28c%29%20-%20s%20%5Ctimes%20log%28k%29%5C%5C" alt="log(f(k)) = log(c) - s \times log(k)\\" class="ee_img tr_noresize" eeimg="1">

![xxx](https://gitee.com/drdrxp/bed/raw/master-md2zhihu-asset/cdn/29ff04ae01d0e13e-1kloglog-regression.png)

<center>(图2) 取对数后呈现线性关系</center>

然后通过[多项式回归](https://zh.wikipedia.org/wiki/%E5%A4%9A%E9%A1%B9%E5%BC%8F%E5%9B%9E%E5%BD%92)，对(图2)中的点拟合一条直线，就可以得到`c`和`s`。

例如我们使用预先准备好的url计数文件
[file-access-count.txt](/post-res/cache-hit/file-access-count.txt)
作为例子，
使用这个
[find-zipf.py](/post-res/cache-hit/find-zipf.py)
脚本来拟合可以得到:

```sh
> python2 find-zipf.py
6.796073 - 0.708331x
```

再分别对两边取指数，就从日志中得到了zipf 曲线的参数:
**第k热的对象和它被访问次数的关系**:

<img src="https://www.zhihu.com/equation?tex=f%28k%29%20%3D894/k%5E%7B0.708331%7D%5C%5C" alt="f(k) =894/k^{0.708331}\\" class="ee_img tr_noresize" eeimg="1">

其中`s`的值是我们后面计算需要用到的。

> zipf中的`c`参数在我们的分析中没有被使用到，
> 因为`c`只决定了文件被访问频率的分布的绝对值，而不影响分布的相对值。
> 而我们要考察的回源率`b`，是一个比值，它只跟频率分布的相对值有关。


# 通过zipf建立CDN边缘容量和回源率的关系

确定了zipf方程后，
我们就可以准确的给出CDN系统中以下3个关键变量的关系:

-   全量文件`n`TB，
-   边缘机房容量`e*n`TB，
-   和回源率`b`。

<img src="https://www.zhihu.com/equation?tex=b%20%3D%5Cfrac%7B%E4%B8%8D%E5%9C%A8%E8%BE%B9%E7%BC%98%E7%9A%84%E6%96%87%E4%BB%B6%E7%9A%84%E8%AF%B7%E6%B1%82%E6%95%B0%7D%7B%E5%85%A8%E9%83%A8%E8%AF%B7%E6%B1%82%E6%95%B0%7D%3D%5Cfrac%7B%5Cint_%7Ben%7D%5En%20c/k%5Es%20dk%7D%7B%5Cint_1%5En%20c/k%5Es%20dk%7D%3D%5Cfrac%7Bn%5E%7B1-s%7D-%28en%29%5E%7B1-s%7D%7D%7Bn%5E%7B1-s%7D%20-%201%7D%5Ctag%7B2%7D%5C%5C" alt="b =\frac{不在边缘的文件的请求数}{全部请求数}=\frac{\int_{en}^n c/k^s dk}{\int_1^n c/k^s dk}=\frac{n^{1-s}-(en)^{1-s}}{n^{1-s} - 1}\tag{2}\\" class="ee_img tr_noresize" eeimg="1">

因为`s` 已经通过直线拟合得到了，而`e`和`b`可以从边缘机房的具体配置和访问日志得到。
于是我们可以确定`n`的值，
也就是说(划重点):

**CDN可以根据访问模型估算源站的总文件量**。

> 虽然源站信息一般不会直接透漏给CDN提供商，
> 但实际上这个信息对CDN来说是可估算的。
> 
> 同样，对于源站来说，他知道自己的全量数据`n`的值，以及回源率`b`的值，
> 通过这个公式也可以知道CDN的 **一个边缘机房的存储容量`e`**。


# 确定IT设施单位成本

有了`b`，`e`的关系，接下来给出基础**成本数据**，就可以确定最优成本对应的`e`的值了:

从网上搜一些基础数据(如服务器价格，硬盘价格，各家云大厂提供的带宽单价等)，
折算成单位成本为后面计算时使用:

-   边缘机房的存储的成本是 `1 元/TB/天`，包括服务器和租机房的费用，三年均摊。
-   边缘机房拉取源站内容的带宽成本是 `600 元/Gbps/天`。

> 价格等数值因为是从网上搜来的，可能跟实际运营中的数据相差很多，这里只为用来说明原理，
> 确定方法，不保证数值上跟真实环境完全吻合。


根据以上设定，我们可以得到一个边缘机房中:

-   存储成本是: `n * e * 1元/TB/天`
-   回源带宽成本: `20Gbps * b * 600元/Gbps/天`

# 找出最优配置

现在所有的模型，场景和数据都准备好了，
最后我们把它们放在一起得到最终结论: **成本对边缘容量的变化**:

1.  选择不同的边缘容量`e`的值((图1)橘黄色部分的宽度)，
1.  根据公式`(2)`计算对应的回源率`b`((图1)蓝色部分积分/蓝色黄色部分积分)，
1.  再结合单位成本，得出每个`e`对应的总成本，

这里我们选择一个更大的样本,
其中`s=1.10138726235`, `n=8,000,000`,

<img src="https://www.zhihu.com/equation?tex=f%28k%29%20%3Dc/k%5E%7B1.10138726235%7D%5C%5C" alt="f(k) =c/k^{1.10138726235}\\" class="ee_img tr_noresize" eeimg="1">

得到如下一个边缘容量`e`决定回源率`b`和成本的表格:

<table>
<tr class="header">
<th style="text-align: right;">e:边缘存储容量占比%</th>
<th style="text-align: right;">b:回源率%</th>
<th style="text-align: right;">存储成本</th>
<th style="text-align: right;">回源带宽成本</th>
<th style="text-align: right;">总成本</th>
</tr>
<tr class="odd">
<td style="text-align: right;">2.00%</td>
<td style="text-align: right;">12.14%</td>
<td style="text-align: right;">144</td>
<td style="text-align: right;">1456.8</td>
<td style="text-align: right;">1600.8</td>
</tr>
<tr class="even">
<td style="text-align: right;">3.00%</td>
<td style="text-align: right;">10.64%</td>
<td style="text-align: right;">216</td>
<td style="text-align: right;">1276.8</td>
<td style="text-align: right;">1492.8</td>
</tr>
<tr class="odd">
<td style="text-align: right;">4.00%</td>
<td style="text-align: right;">9.62%</td>
<td style="text-align: right;">288</td>
<td style="text-align: right;">1154.4</td>
<td style="text-align: right;">1442.4</td>
</tr>
<tr class="even">
<td style="text-align: right;">5.00%</td>
<td style="text-align: right;">8.85%</td>
<td style="text-align: right;">360</td>
<td style="text-align: right;">1062.0</td>
<td style="text-align: right;">1422.0</td>
</tr>
<tr class="odd">
<td style="text-align: right;"><code>min ------&gt; 6.00%</code></td>
<td style="text-align: right;">8.23%</td>
<td style="text-align: right;">432</td>
<td style="text-align: right;">987.6</td>
<td style="text-align: right;"><strong>1419.6</strong></td>
</tr>
<tr class="even">
<td style="text-align: right;">7.00%</td>
<td style="text-align: right;">7.72%</td>
<td style="text-align: right;">504</td>
<td style="text-align: right;">926.4</td>
<td style="text-align: right;">1430.4</td>
</tr>
<tr class="odd">
<td style="text-align: right;">8.00%</td>
<td style="text-align: right;">7.28%</td>
<td style="text-align: right;">576</td>
<td style="text-align: right;">873.6</td>
<td style="text-align: right;">1449.6</td>
</tr>
<tr class="even">
<td style="text-align: right;">9.00%</td>
<td style="text-align: right;">6.89%</td>
<td style="text-align: right;">648</td>
<td style="text-align: right;">826.8</td>
<td style="text-align: right;">1474.8</td>
</tr>
<tr class="odd">
<td style="text-align: right;">10.00%</td>
<td style="text-align: right;">6.56%</td>
<td style="text-align: right;">720</td>
<td style="text-align: right;">787.2</td>
<td style="text-align: right;">1507.2</td>
</tr>
<tr class="even">
<td style="text-align: right;">…</td>
<td style="text-align: right;">…</td>
<td style="text-align: right;">…</td>
<td style="text-align: right;">…</td>
<td style="text-align: right;">…</td>
</tr>
</table>

我们看到，在这个业务场景中，**边缘容量取到6%时，总成本会达到最低: 1419.6 元/天**。

把`e`变化作为x轴，画出边缘存储成本和回源带宽成本随`e`变化的曲线如下图，
趋势看起来就更清楚了:

![xxx](https://gitee.com/drdrxp/bed/raw/master-md2zhihu-asset/cdn/310f872d60c8787b-edge-backsource-cost.png)

我们看到总成本(红色曲线)在这个图上有一个极小值，大概在`e=6%`的位置。

而如果边缘机房的容量配置不在`6%`这个位置，譬如达到**`10%`**，
一个边缘机房每天就会多花掉`1507-1419= 88` 元，
如果整个CDN系统有`100`个边缘机房，

**一年多花的钱就是320万** (`88 * 100 * 365`)。

> 当然，在实际运营一个CDN这种庞大的系统的时候，还有很多因素需要考虑:
> 
> -   如边缘机房的存储容量带来的性能提升，
> -   低回源率带来的整体延迟降低等。
> 
> 因此我们的结论尚不能作为优化的标准，而只能作为优化的参考:
> 
> -   知道理论上限在哪，
> -   不做无效的尝试，
> -   制定有依据的目标，
> -   了解运营策略调整的成本等等。


# 总结

算完了，休息一下。

互联网的各种服务之间都是相互关联的。
虽然目前看来，源站和CDN之间的协作还是单方面的，一方去适应另一方，
或一方服务另一方的模式。
其实源站和边缘之间，本应是一个整体，但市场的划分导致了技术和设施层面上的分离，
让这个本应完整的框架分离成了2个或多个部分。

这里我们看到的源站和边缘之间的关系，
还只是开始，这些只不过是数学结论上的互相了解，
源站和边缘之间应该有比这更大的的协作空间。

试想一下，边缘和源站之间如果能在单向数据下载的基础上建立双向数据通讯，
在中心源站数据发生变化要通知边缘缓存的基础上，
如果边缘发生的事情能很快推送到中心，
才是真正的互联网模式。

市场一直在变化，
每次在这些变化背后，
一定是有一个什么新的东西出现，是让用户使用到了更方便或更高效的东西，
才让技术的格局发生了变化(google之于yahoo，微信之于邮箱，twitter之于博客)。

出色的支持和适配可以产生优质的产品，但我相信协作和互通更能带来质的变化。
作为一个技术从业人员，发现这些改变的可能，实现这些改变，并把这种改变服务于可受益的人，
也许就是最开心的事情了: **Discover, Design, Distribute**...



Reference:

- file-access-count.txt : [/post-res/cache-hit/file-access-count.txt](/post-res/cache-hit/file-access-count.txt)

- find-zipf.py : [/post-res/cache-hit/find-zipf.py](/post-res/cache-hit/find-zipf.py)

- zipf : [https://en.wikipedia.org/wiki/Zipf%27s_law](https://en.wikipedia.org/wiki/Zipf%27s_law)

- zipf-blog : [/tech/2019/10/30/zipf](/tech/2019/10/30/zipf)

- 多项式回归 : [https://zh.wikipedia.org/wiki/%E5%A4%9A%E9%A1%B9%E5%BC%8F%E5%9B%9E%E5%BD%92](https://zh.wikipedia.org/wiki/%E5%A4%9A%E9%A1%B9%E5%BC%8F%E5%9B%9E%E5%BD%92)


[file-access-count.txt]: /post-res/cache-hit/file-access-count.txt
[find-zipf.py]: /post-res/cache-hit/find-zipf.py
[zipf]:  https://en.wikipedia.org/wiki/Zipf%27s_law
[zipf-blog]:  /tech/2019/10/30/zipf
[多项式回归]:  https://zh.wikipedia.org/wiki/%E5%A4%9A%E9%A1%B9%E5%BC%8F%E5%9B%9E%E5%BD%92