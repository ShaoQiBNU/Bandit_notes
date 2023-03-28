# Bandit算法总结

## 前言
> Exploration and Exploitation(EE问题，探索与开发)是计算广告和推荐系统里常见的一个问题，主要目的是为了平衡推荐系统的准确性和多样性。

- Exploitation：直译就是开采，利用已知的比较确定的用户的兴趣，然后推荐与之相关的内容；

- Exploration：直译就是探索，除了推荐已知的用户感兴趣的内容，还需要不断探索用户其他兴趣，否则推荐出的结果来来回回都差不多。

> 在计算广告和推荐系统中，常常会面临物品冷启动的问题，新入库的物品没有历史累计数据，直接与其他热门物品一起排序会很吃亏。所以业界常常切出一小部分流量对冷启物品做EE，度过冷启阶段后就与其他热门物品一起排序，EE的算法主要有Naive、UCB、Thompson sampling(汤普森采样)、Epsilon-Greedy(ε-Greedy)等。

## Bandit算法

### Naive算法

> Naive算法做法是为物品设定冷启期(设定发送/曝光阈值)，在冷启阶段的所有物品随机分发，度过冷启阶段后，统计物品的后验指标如ctr、人均观看时长等，基于该指标排序分发。

- 该方法简单易行，但对线上业务指标有负向影响，一般会设置冷启强插概率(常设10%)，对一次请求，以10%的概率做冷启物品的探索，从而尽量减少对线上指标的影响。

### Thompson sampling(汤普森采样)

#### 前置知识
- 需要理解离散变量和连续变量下的概率函数和概率分布函数，参考：https://www.jianshu.com/p/b570b1ba92bb

- Beta分布推导
参考：https://zhuanlan.zhihu.com/p/69606875

- Beta分布特点
> Beta分布的PDF可以是钟形曲线、严格递增或递减甚至是直线以及渐近末端的U形曲线。当改变 $\alpha和\beta$ 的取值时，PDF曲线的形状会发生变化，如图所示https://github.com/ShaoQiBNU/Bandit_notes/blob/main/bandit.ipynb：

- 当$\alpha <1, \beta < 1$时，Beta的pdf为U型曲线；

- 当$\alpha =1, \beta = 1$时，Beta的pdf为均匀分布；

- 当$\alpha 和 \beta$大致相等时，Beta的pdf近似正态分布；

- 当$\alpha + \beta$越大，Beta的pdf分布曲线越窄，即越集中；依据此概率分布产生的随机数越接近中心位置；

- 当$\frac{\alpha}{\alpha + \beta}$越大，Beta的pdf分布越靠近1，否则越靠近0；依据此概率分布产生的随机数越靠近1，反之则都靠近0；

**注意：当参数$\alpha和\beta$确定后，使用 Beta 分布生成的随机数有可能不一样，所以汤普森采样法是不确定算法**
> 基于五组$\alpha和\beta$参数，模拟了10次随机概率值，得到如下结果：
```shell
a = 1, b =  1
0.6937000107561102
0.7840366816435548
0.41235411161636654
0.14500315935872166
0.07991505398994446
0.9643541906785918
0.5748189410179044
0.8395408332143908
0.28660976716301595
0.10965960835195543
max_prob:0.9643541906785918  min_prob: 0.07991505398994446


a = 5, b =  5
0.5383382032495503
0.5473594677577375
0.5930023365427707
0.6044101368790439
0.5085466313517792
0.4488450388940757
0.7270110373290828
0.4453501384966415
0.5938435832642678
0.4102626786275707
max_prob:0.7270110373290828  min_prob: 0.4102626786275707


a = 20, b =  5
0.6211395171854128
0.8117609090463984
0.6928939526732747
0.9010505793385053
0.8768824862565948
0.8291180834117697
0.8914348903229489
0.6198952043947642
0.8668317783463259
0.765599042572489
max_prob:0.9010505793385053  min_prob: 0.6198952043947642


a = 50, b =  100
0.28512116274530674
0.333322074517305
0.3824232661163195
0.37274101879976523
0.3316405036549513
0.2965792285890082
0.2910518503095889
0.3994700513677794
0.297189333260232
0.3062678531805608
max_prob:0.3994700513677794  min_prob: 0.28512116274530674


a = 1000, b =  800
0.5483641721518319
0.5522566812988445
0.5362789289510086
0.5497234700211355
0.570352221025448
0.5365367098756668
0.563454081227833
0.5629657428657496
0.5627366300338008
0.5587835318483853
max_prob:0.570352221025448  min_prob: 0.5362789289510086
```

> 从结果可以看出，当$\alpha和\beta$ 较小时，产生的随机概率值方差较大，说明此时收益不确定，可能很好，可能很差；当$\alpha和\beta$较大时，产生的随机概率值方差较小，说明此时收益基本趋近于稳定；

scipy的beta教程：https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.beta.html

- 汤普森采样为什么有效呢？

1）如果一个物品被选中的次数很多，即$ \alpha+\beta$很大，其pdf分布会很窄，此时，物品的收益已经非常确定了，用它产生随机数，基本上就在中心位置附近，接近平均收益。

2）如果一个物品$ \alpha + \beta$很大，而且$\frac{\alpha}{\alpha + \beta}$也很大，接近于1，那就确定这是个好的候选项，平均收益很好，每次选择很占优势，就进入利用阶段；反之则平均分布比较接近于0，几乎再无出头之日。

3）如果一个物品的$\alpha + \beta$很小，分布很宽，也就是没有被选择太多次，说明这个物品是好是坏还不太确定，那么分布就是跳跃的。此时用它产生随机数有可能很大、也可能很小；如果得到一个较大的随机数，在排序时会被优先输出，这就起到了探索作用；如果得到一个较小的随机数，在排序时会被pass。

#### 业界应用

> 从redis里获取物品的 click 和 disClick，然后得到$ \alpha = click + 1, \beta = disClick + 1$，基于$ Beta(\alpha, \beta)$得到随机概率值，按照概率值倒排，进行分发。

> 对于流量较大的场景，可以为每个用户保留一套汤普森参数: $\alpha_(i,j) 和 \beta_(i,j)，i代表用户，j代表物品$，共计 2 * M * N 个参数，M是用户数，N是物品数；

> 对于流量较小的场景，所有用户用一套汤普森参数即可：$\alpha和\beta$

### UCB
> 设定发送/曝光阈值，在冷启阶段的所有物品随机分发，度过冷启阶段后，按照如下公式计算得到物品的收益，基于该指标排序分发。

$ x_{j} + \sqrt{\frac{2lnt}{T_{j}}} $
$ x_{j}是物品的平均收益，t是目前试验次数，T_{j}是该物品试验次数 $











