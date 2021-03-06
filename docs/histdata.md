### 概述
---
不管是回测还是实盘，数据的重要性不言而喻。回测和实盘不同的是：回测只需要历史数据，是处理静态数据的，而实盘除了历史数据，还需要实时数据。


### 回测历史数据的处理
---
回测的时候，支持两种数据格式，一种是csv格式的，这种数据格式是从`Multicharts`导出的数据的标准格式。回测框架处理的时候，根据列的id识别，列名不需要相同。
```csv
<Date>, <Time>, <Open>, <High>, <Low>, <Close>, <Volume>
2014/1/2,09:35:00,2342.2,2343.4,2339.2,2343.2,10226
2014/1/2,09:40:00,2343.4,2343.8,2336.6,2338.0,12114
2014/1/2,09:45:00,2338.2,2344.6,2338.2,2343.2,13029
2014/1/2,09:50:00,2343.4,2344.4,2339.2,2339.2,10843
2014/1/2,09:55:00,2339.2,2339.8,2335.2,2335.2,11218
2014/1/2,10:00:00,2335.0,2337.8,2335.0,2336.2,8748
2014/1/2,10:05:00,2336.0,2341.8,2335.0,2341.0,11790
2014/1/2,10:10:00,2341.0,2341.2,2332.0,2332.6,13139
2014/1/2,10:15:00,2332.6,2332.6,2327.4,2330.2,18912
2014/1/2,10:20:00,2330.2,2333.4,2328.6,2332.8,12306
2014/1/2,10:25:00,2332.8,2334.4,2329.2,2331.2,10002
2014/1/2,10:30:00,2331.2,2334.0,2330.2,2332.8,8564
2014/1/2,10:35:00,2333.0,2335.6,2332.2,2332.8,9589
2014/1/2,10:40:00,2332.6,2333.8,2328.6,2330.0,9738
2014/1/2,10:45:00,2330.0,2334.2,2330.0,2331.6,9055
```
配置文件使用csv数据的设置项
```json
{
    "replayer":{
        "mode":"csv",           //数据类型，csv或者bin
        "path":"./storage/",    //数据存储路径
    }
}
```
csv数据格式的目录结构如下，其中`./csv/`是csv数据文件直接存储的子目录，csv数据文件第一次读取的时候，会转换成`WonderTrader`内部数据格式`.dsb`，第二次读取的时候会先检查是否存在`.dsb`文件，如果存在的话，就可以直接加载，从而提升加载速度。
```
root
  |——csv
  |——bin
```
csv文件名采用固定的格式：合约代码_周期.csv，如股指期货主力合约5分钟历史K线对应的csv数据文件名为`CFFEX.IF.HOT_m5.csv`。
![alt  csv回测数据](http://wt.f-sailors.cn/snapshots/csv_hisdata.png)


另外一种，就是WonderTrader的数据组件落地的数据文件，不需要做任何修改，只需要配置要根目录，就可以直接使用。
配置文件使用bin数据的设置项
```json
{
    "replayer":{
        "mode":"bin",           //数据类型，csv或者bin
        "path":"./FUT_Data/",    //数据存储路径
    }
}
```

### 实盘历史数据的处理
---
实盘环境下，数据加载速度是比较重要的问题。所以我们不能再继续在实盘中使用csv格式的数据了。而且实盘数据还要使用实时数据，所以实盘数据的处理更为复杂一些。

* 期货的历史数据处理
    生产环境下，必须要处理静态数据(历史数据)和动态数据(实时数据)拼接的问题。对于分月合约，如`CFFEX.IF2005`来说，直接读取历史的K线和实时的K线进行无缝拼接就可以了。但是对于主力合约，如`CFFEX.IF.HOT`来说，就需要根据主力合约换月的规则，读取多个分月合约的历史数据进行拼接。所以WonderTrader专门建立了一个主力合约换月规则的机制来进行控制。
    ```json
    {
        "CFFEX": {
            "IF": [
                {
                    "date": 20191220,
                    "from": "IF1912",
                    "newclose": 4045.2,
                    "oldclose": 4028.8,
                    "to": "IF2001"
                },
                {
                    "date": 20200117,
                    "from": "IF2001",
                    "newclose": 4176.4,
                    "oldclose": 4157.4,
                    "to": "IF2003"
                },
                {
                    "date": 20200320,
                    "from": "IF2003",
                    "newclose": 3561.8,
                    "oldclose": 3584.0,
                    "to": "IF2004"
                }
            ]
        }
    }
    ```
    如上所示，是从2019年12月20日到2020年3月底的主力合约映射规则。在缓存IF主力合约历史数据的时候，就会根据这个切换规则表，读取对应的分月合约，然后缓存到一起，提供给系统使用。
    但是这个时候有一个新的问题产生了：分月合约的历史数据太多太繁琐，如果要一个一个合约去处理历史数据，会很繁琐，而且容易出错。那么有没有更简便的办法来处理更多的主力合约的历史数据呢？针对这个问题，`WonderTrader`提供了一种机制：用户可以从第三方软件中导出主力连续的历史数据，提供给`WonderTrader`使用。
    `WonderTrader`在处理主力合约数据加载的指令的时候，会优先检查是否有直接的主力合约连续数据，然后在这个数据的基础上，再根据换月规则加载分月合约的数据进行拼接。这样的话，用户只需要在第一次提供一下主力连续的历史数据，以后就只需要维护主力换月规则表，就能够完成主力合约的数据处理了。
    ![alt  期货主力数据](http://wt.f-sailors.cn/snapshots/fut_hot.png)
    
* 股票的历史数据处理
    股票和期货不同：期货有主力合约的概念，而股票有复权数据的概念。两个不同的机制，导致在历史数据的处理上也有所不同。
    一般来说，除权除息会导致股票数据的跳空，所以只有复权以后的数据才能减少这种因为除权除息导致的跳空。`WonderTrader`只考虑前复权的方式：即最新数据不变，只需要用除权因子修正历史数据。前复权的好处是：最新价格和实时行情一致，不需要修正，方便处理；坏处是：每次除权除息，历史数据必须要重新修正。
    不管前复权的优缺点，`WonderTrader`既然采用了这样的机制，也提供了相对简便的解决方案。`WonderTrader`中股票代码的格式为“市场ID.股票代码”，如`SSE.600000`，在获取复权数据的时候，我们只需要传递`SSE.600000Q`，就表示要读取复权数据，而如果直接传递`SSE.600000`则表示要读取不复权的数据。股票历史复权数据的拼接规则也是一样，先读取历史复权数据，数据文件名为`“600000Q.dsb”`，然后再读取未复权的最新数据文件`"600000.dsb"`。
    ![alt  股票复权数据](http://wt.f-sailors.cn/snapshots/stk_qfq.png)

### 如何获取历史数据
---
一个稳定、高质量的数据源，对于策略来说是非常重要的。除了我们的数据组件每天运行，进行实时数据落地以外，历史数据的获取也是一个比较重要的步骤。从某种角度来说，策略的研发都是建立在对数据的处理之上的。
对于成熟的团队来说，由很多渠道可以获取到高质量的数据。对于小团队和个人来说，则希望能够有免费的渠道获取历史数据。网络上也有比较多的文章介绍各家的数据获取的api，这里也做一些简单的介绍。

* `Multicharts`导出数据
    `Multicharts`是一款基于图表的交易软件，使用和TradeStation内置的PowerLanguage同源的EasyLanguage作为策略开发语言，策略计算结果可以直接在图表上反馈出来，信号点的展示也非常直观，同时集成了很多有用的算法，比较受策略研发人员的欢迎。
    `Multicharts`不是免费软件，需要向Broker申请开通MC交易，而MC则从客户的佣金上加成25%作为使用费。
    由于`Multicharts`运营比较成熟，所以MC里面提供的行情数据质量还是非常高的。所以这里推荐需要历史数据的用户可以考虑从MC里导出历史数据。
    导出期货主力合约数据
    ![alt  股票复权数据](http://wt.f-sailors.cn/snapshots/mc_export_fut.jpg)
    导出股票复权数据
    ![alt  导出股票复权数据](http://wt.f-sailors.cn/snapshots/mc_export_stk.jpg)

* `tushare`库获取数据
    tushare提供了很多类型的行情数据接口，包括股票列表、历史行情、财务数据、板块数据等。新版本的tusahre还提供了期货、期权、债券等更多市场更多品种的相关数据。老版本的tushare数据是免费的，但是已经停止维护接口了；新版本的tushare，设置了积分的门槛。
    > 老版本tushare接口文档：<http://tushare.org/>
    > 新版本tushare接口文档：<https://tushare.pro/document/2>

    `tushare`相对于`Multicharts`来说，更适合于量化交易：`tushare`全部都是api调用，获取数据非常方便，而MC需要手动导出，相对来说比较麻烦。虽然`tushare`现在有了一些限制，总的来说，数据质量和数据量都还是颇为不错的，不管是做回测还是实盘，应该都可以通过`tushare`拿到足够的数据。
    
* 其他数据源
    网络数据源主要指的是各大门户网站提供的数据api接口以及各大数据聚合平台。
    * 数据聚合平台有：
    > 聚合数据 <https://www.juhe.cn/docs/api/id/21>

    * 门户网站数据api有：
    > 新浪财经 <https://finance.sina.com.cn/>
    > 网易财经 <https://money.163.com/>

    参考文章：[数据接口-免费版（股票数据API）](https://blog.csdn.net/llingmiao/article/details/79941066)

* 购买专业全面的数据源
    很多数据公司如`大智慧`、`万得`、`聚源`等，都提供全面的数据服。购买这类数据源的好处就是，数据质量有保证，因为上游数据服务商有专职的人员来维护和保证数据质量。

### `wtpy`中`StockToolkit.py`模块
---
StockToolkit模块是为了方便用户拉取股票相关数据而增加的一个简单的工具模块。主要功能就是从互联网拉取相关的股票数据，包括：历史K线、股票列表、除权因子等。
StockToolkit模块目前功能相对简单，但是在后续的更新中，将会逐渐增加更多功能，希望能对广大用户有所帮助。
目前StockToolkit模块用的功能如下：
* StockToolkit.dmpStksFromTS
    从`tushare`拉取股票列表并保存到文件。

* StockToolkit.dmpStkDaysFromTS
    从`tushare`拉取股票历史K线数据(日线，5分钟线)。

* StockToolkit.dmpStkDaysFrom163
    从`163`拉取股票历史K线数据。只支持日线数据，而且成交量是以手为单位的。

* StockToolkit.dmpAdjFromSINA
    从`新浪`拉取股票的除权因子数据