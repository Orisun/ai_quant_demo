这是一个demo程序，旨在帮大家快速入门基于机器学习的量化投资方法。适于人群：有一定python基础，对炒股和量化投资一无所知的人。  
所谓量化投资，指根据资产的各方面数据表现，通过<b>程序化</b>的策略或模型来判断在什么时间进行买入或卖出，可以取得最大的收益。这里我们以A股市场为例。
量化投资是跟随趋势，而非创建趋势，尽可能在趋势的早期发现它，在趋势中获利。有些庄家通过连续地下大单买入，让散户跟随，从而拉升股份，这是在创造趋势。
# 运行程序
首先，通过以下命令安装必需的python依赖库。
```shell
pip install -r requirements.txt
```
1. 调用tusare，获取股票的历史价格数据。每支股票的数据会保存在file/data目录下，所有股票的代码保存在文件file/stock_list.txt中，作为demo我只往里面放了10支股票，实战中应该尽可能地多放一些。  
   `python data.py`
2. 根据基础的价格数据，生成机器学习模型所需要的高级特征。最终的特征文件是pickle二进制格式，存放在目录file/feature下。  
   `python feature.py`
3. 训练lightGBM模型。模型文件是file/model.lgb.txt。  
   `python model.py`
4. 根据历史数据回测模型效果。每天应该买入哪支股票以及对应的收益会存储在文件file/record.csv中，同时会计算量化投资最经常关注的三个指标：累积收益、最大回撤、夏普率。  
   `python backtest.py`

以上4步也可以通过执行`python main.py`一次性完成。
# 数据
我们需要搜集哪些信息来及时地发现趋势？影响股价的因素是多方面的，财报、公司新闻、市场情绪等等，作为demo这里只使用股票最近一段时间的OHLC，即Open开盘价、High最高价、Low最低价、Close收盘价。股价在一天之内是不断变化的，其最高点和最低点分别是High和Low，9:30开盘时的价格是Open，15:00收盘时的价格是Close。把OHLC画成一个柱状图，把多天的柱状图从左到右依次排列在画布上就是k线图。  
![柱状图.png](img/柱状图.png)  
Close大于Open说明当天是上涨的，反之代表下跌。国外用绿色的柱状图表示上涨，红色的柱状图表示下跌（红灯停绿灯行）；国内红色表示喜庆，所以用红色表示上涨，绿色表示下跌。  
![k_chart.png](img/k_chart.png)  
获取股票数据的python库有很多，经过各种踩坑我最终还是选择了<a href=https://tushare.pro>TuShare</a>。注册TuShare后会得到一个token，系统根据token对调用方进行身份的识别，有免费版，付费版也划分了多个档次，显然付费越多可调用的接口就越多，调用频率就越高。  
![tushare_token.png](img/tushare_token.png)  
如果在T时刻出现了送股、配股等情况，为保证公司的总市值不变，股价要调低。前复权指调整T时刻之前的价格使k线连续。后复权指调整T时刻之后的价格使k线连续。后复权不需要改动历史数据，便于操作，所以量化投资都使用后复权数据。（使用不复权的数据行不行？不行）
# 特征
使用哪些特征有助于机器学习模型更准确地预测股价的涨跌呢？随便打开一个炒股软件，k线图上会叠加各种各样的图表和曲线，这些指标我们应该认真学习，把它们用到机器学习模型里面来。  
![炒股界面.png](img/炒股界面.png)
作为demo程序我们只使用3种基础的指标：移动平均(MA, Moving Average)、MACD(Moving Average Convergence/Divergence Oscillator)和布林带(Bollinger Bands)。  
MA(n)表示过去n天(包括当天)的平均值取代当前值，n取1时MA曲线跟原始的价格曲线重合。MA是对上下波动的价格曲线进行了平滑，通过MA更容易看清整体的趋势，同时也带来了滞后性。n越大曲线越平滑，滞后性也越严重，即转向越迟钝。n比较小的曲线称之为MA快线，n比较大的曲线称之为MA慢线，快线与慢线的差称之为<b>动量指标</b>，动量反应的是价格近期的变化速度。速度对于预测未来价格的走势是有意义的！速度为正，未来的股价应该会上涨，速度越大上涨得越多。我们要预测的不是股价上涨的绝对值，而是股价上涨的百分比，因为你手里的可用资金是固定的。所以我们会让速度再除以当前的价格。  
MACD线就是MA的快线减慢线，不同股票对比时应该再除以价格。代码中没有使用MACD，而是使用的PPO(Percentage Price Oscillator)，就是在MACD的基础上除了慢线。  
布林带由3条线构成：中线、上限和下限。中线就是近期的移动平均，再算出近期价格波动的标准差，中线加/减m倍的标准差就是得到了上限和下限。大部分情况下价格游走于上限和下限构成的边界之内，价格接近甚至超出边界是一个比较强的信号，对预测股价未来的走势是有帮助的。   
我们并不需要像<a href="https://baike.baidu.com/item/%E5%9B%BE%E8%A1%A8%E4%B8%93%E5%AE%B6/7171011">图表专家</a>那样把非常复杂的指标进行组合，然后用代码进行刻画，最终喂给机器学习模型。机器学习模型可以自动地进行特征的组合、阈值的选择。  
# 模型
根据我多年的从业经验，工业界最接近“万能模型”这个称号的就是lightGBM，遇到任何问题都可以先试一下这个模型。简单来说它是一个决策树模型，最终它会根据历史数据学习出下图所示的预测树。  
![决策树预测股价涨幅.png](img/决策树预测股价涨幅.png)
三个臭皮匠顶个诸葛亮，你可能觉得一棵决策树预测得不准，lightGBM允许你训练很多棵决策树共同参与预测。每棵树越深代表条件组合得越复杂。越复杂不代表就能预测得越准，所以你要反复地调整参数，找到尽可能最优的方案。对于lightGBM模型影响最大的三个参数就是：学习率、树的个数、每棵树的深度。
# 回测

