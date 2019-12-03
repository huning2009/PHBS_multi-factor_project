# BARRA's multi-factor models

View the book with "<i class="fa fa-book fa-fw"></i> Book Mode".

寻找cross-section上有良好解释性的多因子结构，并寻找有效的估计截面回归残差的方法objective: max IC = $corr(E(e_t),e_t)$



## phase 1: 搭建检验因子有效性的框架

**架构目录**

[流程1：因子数据清洗](#流程1：因子数据清洗)
[流程2：单因子有效性回测：取得有效因子](#流程2：-单因子有效性回测：取得有效因子)
[流程3：測試多因子间相關性](#流程3：測試多因子间相關性)
[流程4：多因子回歸模型](#流程4：多因子回歸模型)

**ddl控制(通过制定日期分类)**
[Nov.29/2019](#ddl制定日期：Nov292019)
[Dec.2/2019](#ddl制定日期：Dec22019)

**discussion log(通过讨论日期分类)**
[Dec.2/2019](#dicussion-logDec22019)

---


### ddl(制定日期：Nov.29/2019):
- 练习一下 matlab（evan定制）
- 每人写10个因子（代码规范：camel命名法），ddl: **Dec.2，8am**
- 讨论数据清洗细节以及因子的code review， ddl: **Dec.2，3:30pm**

### ddl(制定日期：Dec.2/2019):
- 根据code review修改code，ddl:**Dec.6,8am**
- 完成清洗数据的代码,分工明细：
Zhangjie L.:行业因子+清洗数据
ddl:**Dec.6,8am**
- 完成风格因子和行业因子的编写,分工明细：
Yuting F.:market beta, momentum, volatility--std($\epsilon_i$)(*CFA ddl:Dec.9/2019*)；
Evan H.:CMRA,liquidity；
Zhihao C.:size,E/P,DASTD
ddl**Dec.6,8am**


---

### dicussion log(Dec.2/2019)
1. 书写因子规范和共识，命名：camel
- 规定alpha因子文件格式(如alpha31.m)
```{Matlab}
function [X,offsetSize] = alpha31(stock)
%main function
%要求注明alpha计算公式
% stock is a structure

% clean data module here

% get alpha module here
    [X,offsetSize] = getAlpha(close, high, ...);
    
%--------------------------
function [factor,offsetSize] = getAlpha(close, high, ...)
%function compute alpha
```
2. 书写style factor规范和共识，命名：camel
- 规定style factor文件格式(如ETOP.m)
```{Matlab}
function [X,offsetSize] = ETOP(stock)
% Returns the historical EP,which is the net revenue of the past 12 months 
% of single stocks divided by their current market capital, 
% earnings_ttm / mkt_freeshares
% stock is a structure

% clean data module here

% get alpha module here
    [X,offsetSize] = getETOP(stock.PE_TTM);
end

%--------------------------
function [exposure,offsetSize] = getETOP(PE_TTM)
%function compute factor exposure of style factor
    exposure = 1 ./ (PE_TTM + eps);
    offsetSize = 1;
end

```
- 计算过程中的consensus
-- rank计算要滚动
-- 计算要包含今天
-- 处理0 division(+eps) 

2. 数据清洗讨论
2.1. 交易所数据：OHLC,volume,amount
**Note:** 填充范式： 0表明没有数据，nan表示数据存在但没有拿到
- step1: tradeDay, count(0) $\geq n_1 = 360$,直接踢出股票池
- step2: stDay, MaxLength(1) $\geq n_2 = 180$,连续ST超过一定时间，直接踢出股票池
- step3: onList time,所有股票在上市交易之前的数据全部填充为0
- step4:如果一个股票的某个关键字段，如收盘价，满足sum(isnan(字段))$\geq n_3$，则踢出股票池，因为数据不足
- step5.1:中间NA: ffill, rollingMean, interploration
- step5.2:头NA： bfill, 0

2.2. 行业因子与风格因子

**行业因子**：
- step1: 踢除 sum(isnan(level_1))$\geq 180$的股票
- step2: fillna(argmax(level_1)),用出现最多的情况填补空值

**风格因子**
- market beta
- size
- E/P
- volatility:DASTD,std($\epsilon$),CMRA
- liquidity


---

### 流程1：因子数据清洗
#### a.缺失值填补
方法：
- i. 检查数据缺失原因决定是否填补或删去
- ii. 填补可使用前一天的数值进行填补
- iii. 使用历史数据进行插值填补
- iv. 待定其他方法

#### b.极端值处理
方法：
— i. 使用中位数或者均值（区别在于把中位数都换成均值）
![](https://i.imgur.com/GGfiTpR.jpg)

**Note:** 去除极端值应该在同一个横截面上（对同一个时刻的所有公司的某一个因子暴露/负载序列使用上面的方法）

#### c.标准化
方法：
- a. z-score
将去极值处理后的因子暴露度序列减去其现在的均值，除以其标准差（Note:这一步操作应在同一个截面的数据上进行），得到一个新的近似服从N(0,1)分布的序列，这样做可以让不同因子的暴露度之间具有可比性
如因子序列$X_1(t_1),...,X_k(t_1)$为时间$t_1$横截面上k个因子的暴露(或称为factor loading),$X_i \in R^{n \times 1},i=1,..,k$, 其中n是$t_1$横截面上的公司（观测值）数量,则对$t_1$数据的标准化过程为：
$$X_{i}(t_1) = \frac{X_{i}(t_1)-mean(X_{i}(t_1))}{std(X_{i}(t_1))}, i = 1,2,...,k$$

#### d.正交化
标准化后的因子序列，对行业因子以及主要风格/风险因子进行回归，取残差得到正交化后的因子序列

---

### 流程2： 单因子有效性回测：取得有效因子

<!-- **ii,iii判断因子对当期回归应变量收益率有显著性贡献的有效性和稳定性**

**iv.判断因子收益率序列本身的性质**

**v.为判断因子对未来预期能力的有效性，稳定性** -->

#### a.統計檢驗：稳定性,显著性,不相关性（见于多因子选择的情况）

**ii,iii判断因子收益率作为回归中系数的显著性，稳定性**

**iv.判断因子收益率序列本身的显著性，稳定性**

**v.为判断因子对未来预期能力的显著性，稳定性**

##### i.通过横截面回归计算单因子收益率序列$f_k(t)$

![](https://i.imgur.com/cFJBpmC.jpg)

---

##### ii. 用每一期截面回归时得到的$f_k(t)$对应的t统计量的绝对值$t_{f_k}(t)$计算显著性:

对于BARRA模型，任取一个时间点T，对于因子k的因子负载/暴露$X_k(T)$， 有一个从回归中得到的显著性统计量$t_{f_k}(T)$，对于显著性统计量的时间序列$\{t_{f_k}(T)\}_{T=1,...,m}$，取绝对值得到 $|t_{f_k}(T)|$，检验原假设$H_0$: $mean(|t_{f_k}(T)|)$=0

假设检验的方法： bootstrap抽样得到$\{|t_{f_k}(T)|\}_{T=1,...,m}$的sampling distribution，用观测到的$mean(|t_{f_k}(T)|)_{observed}$进行单边检验

---

##### iii. 用每一期截面回归时得到的$f_k(t)$对应的t统计量的绝对值$t_{f_k}(t)$计算平稳性（是否对收益率有稳定的显著的影响）

对于显著性统计量的时间序列$\{t_{f_k}(T)\}_{T=1,...,m}$，取绝对值得到 $|t_{f_k}(T)|$,对于设定的threshold（这里是2）,计算：
$$\frac{\#\{t\in T:|t_{f_k}(T)|>2 \}}{T}$$

- **方法1**：检验原假设：$|t_{f_k}(T)|$>2的比例在这一比例真分布中不显著
检验方法： bootstrap抽样得到$\frac{\#\{t\in T:|t_{f_k}(T)|>2 \}}{T}$的sampling distribution（刻画了序列中$|t_{f_k}(T)|$>2的比例的分布）,用观测到的$\frac{\#\{t\in T:|t_{f_k}(T)|>2 \}}{T}$进行单边检验（存在的问题：无法处理在比例的分布中显著，但是真实的观测值仍然很低的问题，可能可以引入empirical的观测值下界来处理这个问题）

- **方法2**：进行ADF检验，检测序列的平稳性($H_0$:序列不稳定)

- **方法3**：使用$t_{f_k}(T)$的标准差进行bootstrap单边检验或对标准差进行 $\chi^2$单边检验.

---

##### iv. 因子收益率序列$f_k(T)$的一致性

- **step1**: 检验因子收益率显著性
检验原假设：$mean(f_k(T))\leq0$（若检验$mean(|f_k(T)|)=0$则通过bootstrap方法进行单边检验）
-- **方法1**：假设$f_k(T)\sim^{i.i.d.}N(\mu, \sigma^2)$，则直接进行t检验，检验统计量$t = \frac{mean(f_k(T))}{std(f_k(T))/\sqrt{T}} \sim^{H_0} t_{T-1}$,单边检验(Note:分布假设合理性说明：继承自线性回归模型的残差iid独立同分布正态假设，自然有系数服从正态分布)
-- **方法2**：用bootstrap进行单边检验

- **step2**: 检验因子收益率序列的平稳性
-- **方法1**： 设置单向,$f_k(T)\geq a, a>0$,a是threshold，方法同iii.
-- **方法2**： 设置绝对值,$|f_k(T)|\geq a$,a是threshold，方法同iii.
-- **方法3**： 使用ADF检验，检测时间序列平稳性
-- **方法4**： 使用$f_k(T)$的标准差进行bootstrap或对标准差进行$\chi^2$检验

---

##### v.因子收益率序列IC(预测能力)

检验$IC_{T}(d)$ = $corr(f_k(T),R_{T+d})$的显著性和稳定性，方法同上，假设检验如下：

显著性：检验原假设$mean(IC)=$ 0(双边检验)或$mean(|IC|)=0$（单边检验），可设置threshold

稳定性：检验IC或|IC|大于某一个threshold的比例或检验时间序列的稳定，或检验序列的标准差

---

#### b.打分法分层回测
1. 依照因子值对股票进行打分，构建投资组合回测，是最直观的衡量指标优劣的手段。一般来 说，通过回归法和计算因子 IC 值都无法确定因子的单调性（例如，某因子值排名在中间 1/3 的个股表现比前 1/3 和后 1/3 的个股表现要好），但是分层回测法是可以确定因子单调性的。 分层回测法逻辑简单，结果清晰，操作方便，并且具有能区分因子单调性的独特优势，是接 受度非常高的一种单因子测试手段。
2. 按照因子收益率fk分成5组，分层回测，每天按市值加权，得到净值曲线，回测年化收益率(可选 年化波动率、夏普比率、最大回撤)
3. 畫圖

处理方法示例：
1. total return(close/close前一天-1)会出现NA值，把total return * not st * 是交易日得到一张新的table，不为0的值可以作为输入。
2. 对于SectorOne，SectorOne（34个行业） * not st得到新的table。

---

### 流程3：測試多因子间相關性
1. 先畫 corr 的圖
2. 矩阵、相关系数矩阵(if needed)
3. 因子合成(if needed 只保留最高的or 两个因子收益率fk之间取平均值，留待进一步讨论)

---

### 流程4：多因子回歸模型

![](https://i.imgur.com/pTzhvKo.jpg)

**Note**:若已经正交化过了，则可以舍去第一步

删除当日St的股票，当日不交易的股票。

**step1**:每一期形成[Xindustey,Xstyle]的巨大矩阵：

Xindustry是申万一级行业。一共是34个行业。

Xstyle根据barra的风险因子选取9大类，目前因子分为Beta，动量，规模，盈利性，波动性，成长性，价值，杠杆率，流动性这９大类。Xstyle因子的每个大类参照Barra_CNE5里，每个大类下小类因子可以进行合成，得到9个Style因子。

**step2**:计算回归系数矩阵的时候，用分块矩阵的办法 beta=(X’X)^-1(X’Y),X是稀疏矩阵
e.g.
$$\begin{bmatrix}y_{1,1}\\y_{1,2}\\...\\y_{1,n_1}\\ y_{2,1}\\...\\y_{2,n_2}\end{bmatrix} = \begin{bmatrix}X1&0\\0&X2\end{bmatrix}\begin{bmatrix}\beta_1 \\ \beta_2 \end{bmatrix} + \epsilon$$

**step3**:画IC的时序图，计算IC序列的显著性和平稳性

---

## phase 2:阅读文献，完善框架
![](https://i.imgur.com/sU5qwgX.jpg)
![](https://i.imgur.com/dh1Biz9.jpg)
![](https://i.imgur.com/n7Smtre.jpg)

## phase 3:alpha prediction
option 1: find new factors
option 2: optimize prediction methods (current choice)

## data

### phase 1:
数据时间区间：2010.10.30 -- 2019.10.30

| 名称                                 | 解释（数据范围）          | 频率 |
| ------------------------------------ | ------------------------- | ---- |
| **价量因子**                         |                           |      |
| 开盘价open                           | 全A股非ST                 | 日频 |
| 收盘价close                          | 全A股非ST                 | 日频 |
| 最低价low                            | 全A股非ST                 | 日频 |
| 最高价high                           | 全A股非ST                 | 日频 |
| 日内均价avg_price                    | 全A股非ST                 | 日频 |
| 成交量volume(VWAP，金额)             | 全A股非ST                 | 日频 |
| 换手率turnover                       | 全A股非ST                 | 日频 |
| 中证500开盘价benchmark open(全收益)  |                           | 日频 |
| 中证500收盘价benchmark close(全收益) |                           | 日频 |
| **行业分类和风格因子**               |                           |      |
| *行业分类*                           |                           |      |
| 申万行业分类（一级&三级分类）        | 全A股非ST                 |      |
| *风格因子*                           |                           |      |
| **size**                             |                           |      |
| 个股流通股票金额                     | 全A股非ST                 | 日频 |
| **earnings yields**                  |                           |      |
| 分析师EPS一致预期estimate EPS        | 全A股非ST                 |      |
| 个股现金收益cash earnings            | 全A股非ST                 | 季频 |
| 个股净利润earnings TTM               | 全A股非ST                 | 季频 |
| **growth**                           |                           |      |
| 营业总收入                           | 全A股非ST，从2005.1.1开始 | 季频 |
| 归属母公司净利润                     | 全A股非ST，从2005.1.1开始 | 季频 |
| 未来1年企业一致预期净利润增长率      | 全A股非ST                 |      |
| 未来3年企业一致预期净利润增长率      | 全A股非ST                 |      |
| **value**                            |                           |      |
| 全A市场总市值                        |                           | 日频 |
| **leverage**                         |                           |      |
| 企业长期负债                         | 全A股非ST                 | 季频 |
| 企业总资产                           | 全A股非ST                 | 季频 |
| 企业总负债                           | 全A股非ST                 | 季频 |
| 企业账面权益(book equity)            | 全A股非ST                 | 季频 |



## timeline
### phase 1:

Nov. 26.2019 - Dec.1.2019:
implement GTJA 20170615
group meeting on Friday morning(Nov.29.2019)

### phase 2:
Dec.1.2019 - Dec.14.2019:
read reference in groups
group meeting on Friday morning(Dec.6.2019, Dec.13.2019)
### phase 3:
Dec.14.2019 - :
follow option 2, use machine learning or other statistical methods to optimize prediction.
or/and follow option 1, find new factor the multi-factor model

## proposal reference
[1] https://mp.weixin.qq.com/s/F5wSuoeSbpD8YKdQWEr4hQ


