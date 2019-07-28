# _*_ coding:utf-8 _*_
# 开发团队：淼洪科技
# 开发人员：16412
# 开发时间：2019-07-28 17:35
# 文件名称：147收益.py
# 开发工具：PyCharm


'''
二八小市值择时买卖

配置指定频率的调仓日，在调仓日每日指定时间，计算沪深300指数和中证500指数当前的20日涨
幅，如果2个指数的20日涨幅有一个为正，则进行选股调仓，之后如此循环往复。

止损策略：

    大盘止损：(可选)
        1. 每分钟取大盘前130日的最低价和最高价，如果最高大于最低的两倍则清仓，停止交易。
        2. 每分钟判断大盘是否呈现三只黑鸦止损，如果是则当天清仓并停止交易，第二天停止交
           易一天。

    个股止损：(可选)
        每分钟判断个股是否从持仓后的最高价回撤幅度，如果超过个股回撤阈值，则平掉该股持仓

    二八止损：(必需)
        每日指定时间，计算沪深300指数和中证500指数当前的20日涨幅，如果2个指数涨幅都为负，
        则清仓，重置调仓计数，待下次调仓条件满足再操作


'''

import tradestat


# from blacklist import *

# blacklist.py
# 建议在研究里建立文件blacklist.py，然后将这段代码拷贝进blacklist.py
# 模拟运行的时候只需要更新研究里的数据即可，这样可在不修改模拟运行代码的情况下
# 修改黑名单

# 配置股票黑名单
# 列出当且极不适宜购买的股票
# 注：1. 黑名单有时效性，回测的时候最好不使用，模拟交易建议使用
#     2. 用一模块或者大数据分析收集这类股票，定时更新
def get_blacklist():
    # 科恒股份、太空板业，一旦2016年继续亏损，直接面临暂停上市风险
    blacklist = ["600656.XSHG", "300372.XSHE", "600403.XSHG", "600421.XSHG", "600733.XSHG", "300399.XSHE",
                 "600145.XSHG", "002679.XSHE", "000020.XSHE", "002330.XSHE", "300117.XSHE", "300135.XSHE",
                 "002566.XSHE", "002119.XSHE", "300208.XSHE", "002237.XSHE", "002608.XSHE", "000691.XSHE",
                 "002694.XSHE", "002715.XSHE", "002211.XSHE", "000788.XSHE", "300380.XSHE", "300028.XSHE",
                 "000668.XSHE", "300033.XSHE", "300126.XSHE", "300340.XSHE", "300344.XSHE", "002473.XSHE"]
    return blacklist


def before_trading_start(context):
    log.info("---------------------------------------------")
    # log.info("==> before trading start @ %s", str(context.current_dt))

    # 盘前就判断三黑鸦状态，因为判断的数据为前4日
    g.is_last_day_3_black_crows = is_3_black_crows(g.index_4_stop_loss_by_3_black_crows)
    if g.is_last_day_3_black_crows:
        log.info("==> 前4日已经构成三黑鸦形态")
    pass


def after_trading_end(context):
    # log.info("==> after trading end @ %s", str(context.current_dt))
    g.trade_stat.report(context)

    reset_day_param()

    # 得到当前未完成订单
    orders = get_open_orders()
    for _order in orders.values():
        log.info("canceled uncompleted order: %s" % (_order.order_id))
    pass


def initialize(context):
    log.info("==> initialize @ %s", str(context.current_dt))

    # 设置手续费率
    set_commission(PerTrade(buy_cost=0.0003, sell_cost=0.0013, min_cost=5))
    # 设置基准指数：沪深300指数 '000300.XSHG'
    set_benchmark('000300.XSHG')
    # 设定滑点为百分比
    # 没有调用set_slippage函数, 系统默认的滑点是PriceRelatedSlippage(0.00246)
    set_slippage(PriceRelatedSlippage(0.01))
    # 使用真实价格回测(模拟盘推荐如此，回测请注释)
    set_option('use_real_price', True)

    # 加载统计模块
    g.trade_stat = tradestat.trade_stat()

    # 配置策略参数
    # 此配置主要为之前的小市值策略，保证之前的收益回撤
    # 如果想要更改，最好新建个函数，调整参数测试其他策略
    # 10日调仓
    # 关闭大盘三乌鸦及高低价止损
    # 关闭个股止盈止损
    # 关闭选股评分
    set_param()

    # 缓存股票持仓后的最高价
    g.last_high = {}

    # 如下参数不能更改
    if g.is_market_stop_loss_by_price:
        # 记录当日是否满足大盘价格止损条件，每日盘后重置
        g.is_day_stop_loss_by_price = False

    # 缓存三黑鸦判断状态
    g.is_last_day_3_black_crows = False
    if g.is_market_stop_loss_by_3_black_crows:
        g.cur_drop_minute_count = 0

    if g.is_rank_stock:
        if g.rank_stock_count > g.pick_stock_count:
            g.rank_stock_count = g.pick_stock_count

    if g.is_stock_stop_loss or g.is_stock_stop_profit:
        # 缓存当日个股250天内最大的3日涨幅，避免当日反复获取，每日盘后清空
        g.pct_change = {}

    # 打印策略参数
    log_param()


def set_param():
    # 调仓频率，单位：日
    g.period = 3
    # 调仓日计数器，单位：日
    g.day_count = 0
    # 配置调仓时间（24小时分钟制）
    g.adjust_position_hour = 14
    g.adjust_position_minute = 50

    # 配置选股参数

    # 备选股票数目
    g.pick_stock_count = 100

    # 配置选股参数
    # 是否根据PE选股
    g.pick_by_pe = False
    # 如果根据PE选股，则配置最大和最小PE值
    if g.pick_by_pe:
        g.max_pe = 200
        g.min_pe = 0

    # 是否根据EPS选股
    g.pick_by_eps = True
    # 配置选股最小EPS值
    if g.pick_by_eps:
        g.min_eps = 0

    # 配置是否过滤创业板股票
    g.filter_gem = True
    # 配置是否过滤黑名单股票，回测建议关闭，模拟运行时开启
    g.filter_blacklist = False

    # 是否对股票评分
    g.is_rank_stock = True
    if g.is_rank_stock:
        # 参与评分的股票数目
        g.rank_stock_count = 20

    # 买入股票数目
    g.buy_stock_count = 3

    # 配置二八指数
    # g.index2 = '000300.XSHG'  # 沪深300指数，表示二，大盘股
    # g.index8 = '000905.XSHG'  # 中证500指数，表示八，小盘股
    g.index2 = '000016.XSHG'  # 上证50指数
    g.index8 = '399333.XSHE'  # 中小板R指数
    g.gold = '518880.XSHG'
    # g.index8 = '399006.XSHE'  # 创业板指数

    # 判定调仓的二八指数20日增幅
    # g.index_growth_rate_20 = 0.00
    g.index_growth_rate_20 = 0.01

    # 配置是否根据大盘历史价格止损
    # 大盘指数前130日内最高价超过最低价2倍，则清仓止损
    # 注：关闭此止损，收益增加，但回撤会增加
    g.is_market_stop_loss_by_price = True
    if g.is_market_stop_loss_by_price:
        # 配置价格止损判定指数，默认为上证指数，可修改为其他指数
        g.index_4_stop_loss_by_price = '000001.XSHG'

    # 配置三黑鸦判定指数，默认为上证指数，可修改为其他指数
    g.index_4_stop_loss_by_3_black_crows = '000001.XSHG'

    # 配置是否开启大盘三黑鸦止损
    # 个人认为针对大盘判断三黑鸦效果并不好，首先有效三只乌鸦难以判断，准确率实际来看也不好，
    # 其次，分析历史行情看一般大盘出现三只乌鸦的时候，已经严重滞后了，使用其他止损方式可能会更好
    g.is_market_stop_loss_by_3_black_crows = False
    if g.is_market_stop_loss_by_3_black_crows:
        g.dst_drop_minute_count = 10

    # 配置是否个股止损
    g.is_stock_stop_loss = False
    # 配置是否个股止盈
    g.is_stock_stop_profit = False


def log_param():
    log.info("调仓日频率: %d日" % (g.period))
    log.info("调仓时间: %s:%s" % (g.adjust_position_hour, g.adjust_position_minute))

    log.info("备选股票数目: %d" % (g.pick_stock_count))

    log.info("是否根据PE选股: %s" % (g.pick_by_pe))
    if g.pick_by_pe:
        log.info("选股最大PE: %s" % (g.max_pe))
        log.info("选股最小PE: %s" % (g.min_pe))

    log.info("是否根据EPS选股: %s" % (g.pick_by_eps))
    if g.pick_by_eps:
        log.info("选股最小EPS: %s" % (g.min_eps))

    log.info("是否过滤创业板股票: %s" % (g.filter_gem))
    log.info("是否过滤黑名单股票: %s" % (g.filter_blacklist))
    if g.filter_blacklist:
        log.info("当前股票黑名单：%s" % str(get_blacklist()))

    log.info("是否对股票评分选股: %s" % (g.is_rank_stock))
    if g.is_rank_stock:
        log.info("评分备选股票数目: %d" % (g.rank_stock_count))

    log.info("买入股票数目: %d" % (g.buy_stock_count))

    log.info("二八指数之二: %s - %s" % (g.index2, get_security_info(g.index2).display_name))
    log.info("二八指数之八: %s - %s" % (g.index8, get_security_info(g.index8).display_name))
    log.info("判定调仓的二八指数20日增幅: %.1f%%" % (g.index_growth_rate_20 * 100))

    log.info("是否开启大盘历史高低价格止损: %s" % (g.is_market_stop_loss_by_price))
    if g.is_market_stop_loss_by_price:
        log.info("大盘价格止损判定指数: %s - %s" % (
        g.index_4_stop_loss_by_price, get_security_info(g.index_4_stop_loss_by_price).display_name))

    log.info("大盘三黑鸦止损判定指数: %s - %s" % (
    g.index_4_stop_loss_by_3_black_crows, get_security_info(g.index_4_stop_loss_by_3_black_crows).display_name))
    log.info("是否开启大盘三黑鸦止损: %s" % (g.is_market_stop_loss_by_3_black_crows))
    if g.is_market_stop_loss_by_3_black_crows:
        log.info("三黑鸦止损开启需要当日大盘为跌的分钟计数达到: %d" % (g.dst_drop_minute_count))

    log.info("是否开启个股止损: %s" % (g.is_stock_stop_loss))
    log.info("是否开启个股止盈: %s" % (g.is_stock_stop_profit))


# 重置当日参数，仅针对需要当日需要重置的参数
def reset_day_param():
    if g.is_market_stop_loss_by_price:
        # 重置当日大盘价格止损状态
        g.is_day_stop_loss_by_price = False

    if g.is_stock_stop_loss or g.is_stock_stop_profit:
        # 清空当日个股250天内最大的3日涨幅的缓存
        g.pct_change.clear()

    # 重置三黑鸦状态
    g.is_last_day_3_black_crows = False
    if g.is_market_stop_loss_by_3_black_crows:
        g.cur_drop_minute_count = 0


# 按分钟回测
def handle_data(context, data):
    if g.is_market_stop_loss_by_price:
        if market_stop_loss_by_price(context, g.index_4_stop_loss_by_price):
            return

    if g.is_market_stop_loss_by_3_black_crows:
        if market_stop_loss_by_3_black_crows(context, g.index_4_stop_loss_by_3_black_crows, g.dst_drop_minute_count):
            return

    if g.is_stock_stop_loss:
        stock_stop_loss(context, data)

    if g.is_stock_stop_profit:
        stock_stop_profit(context, data)

    # 获得当前时间
    hour = context.current_dt.hour
    minute = context.current_dt.minute

    # 每天下午14:50调仓
    if hour == g.adjust_position_hour and minute == g.adjust_position_minute:
        do_handle_data(context, data)


def do_handle_data(context, data):
    log.info("调仓日计数 [%d]" % (g.day_count))

    # 回看指数前20天的涨幅
    gr_index2 = get_growth_rate(g.index2)
    gr_gold = get_growth_rate(g.gold)
    gr_index8 = get_growth_rate(g.index8)
    gr_index20 = get_growth_rate5(g.index2)
    gr_index80 = get_growth_rate5(g.index8)
    log.info("当前%s指数的20日涨幅 [%.2f%%]" % (get_security_info(g.index2).display_name, gr_index2 * 100))
    log.info("当前%s指数的20日涨幅 [%.2f%%]" % (get_security_info(g.index8).display_name, gr_index8 * 100))
    log.info("当前%s指数的5日涨幅 [%.2f%%]" % (get_security_info(g.index2).display_name, gr_index20 * 100))
    log.info("当前%s指数的5日涨幅 [%.2f%%]" % (get_security_info(g.index8).display_name, gr_index80 * 100))
    gr_index2_1 = (attribute_history(g.index2, 21, '1d', ('close'), True)['close'][-2] -
                   attribute_history(g.index2, 21, '1d', ('close'), True)['close'][0]) / \
                  attribute_history(g.index2, 21, '1d', ('close'), True)['close'][0]
    gr_index8_1 = (attribute_history(g.index8, 21, '1d', ('close'), True)['close'][-2] -
                   attribute_history(g.index8, 21, '1d', ('close'), True)['close'][0]) / \
                  attribute_history(g.index8, 21, '1d', ('close'), True)['close'][0]
    avg_5_2 = data[g.index2].mavg(10, field='close') / attribute_history(g.index2, 11, '1d', ('close'), True)['close'][
        -10]
    avg_5_8 = data[g.index8].mavg(10, field='close') / attribute_history(g.index8, 11, '1d', ('close'), True)['close'][
        -10]
    log.info(avg_5_2, avg_5_8)
    if gr_index2 <= g.index_growth_rate_20 and gr_index8 <= g.index_growth_rate_20 and gr_gold <= g.index_growth_rate_20:
        clear_position(context)
        g.day_count = 0

    elif gr_index2 <= g.index_growth_rate_20 and gr_index8 <= g.index_growth_rate_20 and gr_gold > g.index_growth_rate_20:
        if g.day_count % g.period == 0 or (
                '518880.XSHG' not in context.portfolio.positions.keys() and len(context.portfolio.positions) > 0):
            if '518880.XSHG' not in context.portfolio.positions.keys() and len(context.portfolio.positions) > 0:
                g.day_count = 0
            log.info("==> 满足条件进行调仓")
            #  buy_stocks = pick_stocks2(context, data)
            # log.info("选股后可买股票: %s" %(buy_stocks))
            # close_short(context)
            adjust_position2(context, buy_stocks=('518880.XSHG', '518880.XSHG', '518880.XSHG'))
        g.day_count += 1
    # elif  gr_index2 > g.index_growth_rate_20 and (( (gr_index20 -gr_index80)/ abs(gr_index80)>0.5 and (gr_index2 -gr_index8)/abs(gr_index8)>0.5) or (gr_index20 -gr_index80)/abs(gr_index80)>2): #or ret_index8 > g.index_growth_rate_20:
    # elif  gr_index2 > g.index_growth_rate_20 and gr_index20 -gr_index2> gr_index80-gr_index8: #or ret_index8 > g.index_growth_rate_20:
    elif gr_index2 > g.index_growth_rate_20 and gr_index2 > 2 * gr_index8 and (
            avg_5_2 - avg_5_8 > 0.05 or (gr_index20 > 1.5 * gr_index80)):
        if g.day_count % g.period == 0 or (
                '150052.XSHE' not in context.portfolio.positions.keys() and len(context.portfolio.positions) > 0):
            if '150052.XSHE' not in context.portfolio.positions.keys() and len(context.portfolio.positions) > 0:
                g.day_count = 0
            log.info("==> 满足条件进行调仓")
            #  buy_stocks = pick_stocks2(context, data)
            # log.info("选股后可买股票: %s" %(buy_stocks))
            # close_short(context)
            adjust_position2(context, buy_stocks=('150052.XSHE', '150052.XSHE', '150052.XSHE'))
        g.day_count += 1
    elif gr_index20 < 0 and gr_index2 > g.index_growth_rate_20 and gr_index2 > 2 * gr_index8 and (
            avg_5_2 - avg_5_8 > 0.05 or (gr_index20 > 0.5 * gr_index80)):
        if g.day_count % g.period == 0 or (
                '150052.XSHE' not in context.portfolio.positions.keys() and len(context.portfolio.positions) > 0):
            if '150052.XSHE' not in context.portfolio.positions.keys() and len(context.portfolio.positions) > 0:
                g.day_count = 0
            log.info("==> 满足条件进行调仓")
            # close_short(context)
            #  buy_stocks = pick_stocks2(context, data)
            # log.info("选股后可买股票: %s" %(buy_stocks))
            adjust_position2(context, buy_stocks=('150052.XSHE', '150052.XSHE', '150052.XSHE'))
        g.day_count += 1
    else:
        if g.day_count % g.period == 0 or '150052.XSHE' in context.portfolio.positions.keys():
            if '150052.XSHE' in context.portfolio.positions.keys():
                g.day_count = 0
            log.info("==> 满足条件进行调仓")
            buy_stocks = pick_stocks(context, data)
            log.info("选股后可买股票: %s" % (buy_stocks))
            # close_short(context)
            adjust_position(context, buy_stocks)
        g.day_count += 1


def market_stop_loss_by_price(context, index):
    # 大盘指数前130日内最高价超过最低价2倍，则清仓止损
    # 基于历史数据判定，因此若状态满足，则当天都不会变化
    # 增加此止损，回撤降低，收益降低

    if not g.is_day_stop_loss_by_price:
        h = attribute_history(index, 160, unit='1d', fields=('close', 'high', 'low'), skip_paused=True)
        low_price_130 = h.low.min()
        high_price_130 = h.high.max()
        if high_price_130 > 2.2 * low_price_130 and h['close'][-1] < h['close'][-4] * 1 and h['close'][-1] > h['close'][
            -100]:
            # 当日第一次输出日志
            log.info("==> 大盘止损，%s指数前130日内最高价超过最低价2倍, 最高价: %f, 最低价: %f" % (
            get_security_info(index).display_name, high_price_130, low_price_130))
            g.is_day_stop_loss_by_price = True

    if g.is_day_stop_loss_by_price:
        clear_position(context)
        g.day_count = 0

    return g.is_day_stop_loss_by_price


def market_stop_loss_by_3_black_crows(context, index, n):
    # 前日三黑鸦，累计当日每分钟涨幅<0的分钟计数
    # 如果分钟计数超过一定值，则开始进行三黑鸦止损
    # 避免无效三黑鸦乱止损
    if g.is_last_day_3_black_crows:
        if get_growth_rate(index, 1) < 0:
            g.cur_drop_minute_count += 1

        if g.cur_drop_minute_count >= n:
            if g.cur_drop_minute_count == n:
                log.info("==> 超过三黑鸦止损开始")

            clear_position(context)
            g.day_count = 0
            return True

    return False


def is_3_black_crows(stock):
    # talib.CDL3BLACKCROWS

    # 三只乌鸦说明来自百度百科
    # 1. 连续出现三根阴线，每天的收盘价均低于上一日的收盘
    # 2. 三根阴线前一天的市场趋势应该为上涨
    # 3. 三根阴线必须为长的黑色实体，且长度应该大致相等
    # 4. 收盘价接近每日的最低价位
    # 5. 每日的开盘价都在上根K线的实体部分之内；
    # 6. 第一根阴线的实体部分，最好低于上日的最高价位
    #
    # 算法
    # 有效三只乌鸦描述众说纷纭，这里放宽条件，只考虑1和2
    # 根据前4日数据判断
    # 3根阴线跌幅超过4.5%

    h = attribute_history(stock, 4, '1d', ('close', 'open'), skip_paused=True, df=False)
    h_close = list(h['close'])
    h_open = list(h['open'])

    if len(h_close) < 4 or len(h_open) < 4:
        return False

    # 三根阴线
    if h_close[-4] > h_open[-4] \
            and (h_close[-1] < h_close[-2] and h_close[-2] < h_close[-3]) \
            and (h_close[-1] < h_open[-1] and h_close[-2] < h_open[-2] and h_close[-3] < h_open[-3]) \
            and h_close[-1] / h_close[-3] - 1 < -0.045 and get_current_data(stock) < h_close[-1] * 0.995:
        return True
    return False


'''
 # 一阳三阴
    if h_close[-4] > h_open[-4] \
        and (h_close[-1] < h_open[-1] and h_close[-2]< h_open[-2] and h_close[-3] < h_open[-3]):
        #and (h_close[-1] < h_close[-2] and h_close[-2] < h_close[-3]) \
        #and h_close[-1] / h_close[-3] - 1 < -0.045:
        return True
    return False
'''
'''
def is_3_black_crows(stock, data):
    # talib.CDL3BLACKCROWS
    his =  attribute_history(stock, 2, '1d', ('close','open'), skip_paused=True, df=False)
    closeArray = list(his['close'])
    closeArray.append(data[stock].close)
    openArray = list(his['open'])
    openArray.append(get_current_data()[stock].day_open)

    if closeArray[0]<openArray[0] and closeArray[1]<openArray[1] and closeArray[2]<openArray[2]:
        if closeArray[-1]/closeArray[0]-1>-0.045:
            his2 =  attribute_history(stock, 4, '1d', ('close','open'), skip_paused=True, df=False)
            closeArray1 = his2['close']
            if closeArray[0]/closeArray1[0]-1>0:
                return True
    return False
'''


# 个股止损
def stock_stop_loss(context, data):
    for stock in context.portfolio.positions.keys():
        cur_price = data[stock].close
        xi = attribute_history(stock, 2, '1d', 'high', skip_paused=True)
        ma = xi.max()
        if g.last_high[stock] < cur_price:
            g.last_high[stock] = cur_price

        threshold = get_stop_loss_threshold(stock, g.period)
        # log.debug("个股止损阈值, stock: %s, threshold: %f" %(stock, threshold))
        if cur_price < g.last_high[stock] * (1 - threshold):
            log.info("==> 个股止损, stock: %s, cur_price: %f, last_high: %f, threshold: %f"
                     % (stock, cur_price, g.last_high[stock], threshold))

            position = context.portfolio.positions[stock]
            if close_position(position):
                g.day_count = 0


# 个股止盈
def stock_stop_profit(context, data):
    for stock in context.portfolio.positions.keys():
        position = context.portfolio.positions[stock]
        cur_price = data[stock].close
        threshold = get_stop_profit_threshold(stock, g.period)
        # log.debug("个股止盈阈值, stock: %s, threshold: %f" %(stock, threshold))
        if cur_price > position.avg_cost * (1 + threshold):
            log.info("==> 个股止盈, stock: %s, cur_price: %f, avg_cost: %f, threshold: %f"
                     % (stock, cur_price, g.last_high[stock], threshold))

            position = context.portfolio.positions[stock]
            if close_position(position):
                g.day_count = 0


# 获取个股前n天的m日增幅值序列
# 增加缓存避免当日多次获取数据
def get_pct_change(security, n, m):
    pct_change = None
    if security in g.pct_change.keys():
        pct_change = g.pct_change[security]
    else:
        h = attribute_history(security, n, unit='1d', fields=('close'), skip_paused=True)
        pct_change = h['close'].pct_change(m)  # 3日的百分比变比（即3日涨跌幅）
        g.pct_change[security] = pct_change
    return pct_change


# 计算个股回撤止损阈值
# 即个股在持仓n天内能承受的最大跌幅
# 算法：(个股250天内最大的n日跌幅 + 个股250天内平均的n日跌幅)/2
# 返回正值
def get_stop_loss_threshold(security, n=3):
    pct_change = get_pct_change(security, 250, n)
    # log.debug("pct of security [%s]: %s", pct)
    maxd = pct_change.min()
    # maxd = pct[pct<0].min()
    avgd = pct_change.mean()
    # avgd = pct[pct<0].mean()
    # maxd和avgd可能为正，表示这段时间内一直在增长，比如新股
    bstd = (maxd + avgd) / 2

    # 数据不足时，计算的bstd为nan
    if not isnan(bstd):
        if bstd != 0:
            return abs(bstd)
        else:
            # bstd = 0，则 maxd <= 0
            if maxd < 0:
                # 此时取最大跌幅
                return abs(maxd)

    return 0.099  # 默认配置回测止损阈值最大跌幅为-9.9%，阈值高貌似回撤降低


# 计算个股止盈阈值
# 算法：个股250天内最大的n日涨幅
# 返回正值
def get_stop_profit_threshold(security, n=3):
    pct_change = get_pct_change(security, 250, n)
    maxr = pct_change.max()

    # 数据不足时，计算的maxr为nan
    # 理论上maxr可能为负
    if (not isnan(maxr)) and maxr != 0:
        return abs(maxr)
    return 0.30  # 默认配置止盈阈值最大涨幅为20%


# 获取股票n日以来涨幅，根据当前价计算
# n 默认20日
def get_growth_rate(security, n=20):
    lc = get_close_price(security, n)
    # c = data[security].close
    c = get_close_price(security, 1, '1m')

    if not isnan(lc) and not isnan(c) and lc != 0:
        return (c - lc) / lc
    else:
        log.error("数据非法, security: %s, %d日收盘价: %f, 当前价: %f" % (security, n, lc, c))
        return 0


def get_growth_rate5(security, n=5):
    lc = get_close_price(security, n)
    # c = data[security].close
    c = get_close_price(security, 1, '1m')

    if not isnan(lc) and not isnan(c) and lc != 0:
        return (c - lc) / lc
    else:
        log.error("数据非法, security: %s, %d日收盘价: %f, 当前价: %f" % (security, n, lc, c))
        return 0


# 获取前n个单位时间当时的收盘价
def get_close_price(security, n, unit='1d'):
    return attribute_history(security, n, unit, ('close'), True)['close'][0]


# 开仓，买入指定价值的证券
# 报单成功并成交（包括全部成交或部分成交，此时成交量大于0），返回True
# 报单失败或者报单成功但被取消（此时成交量等于0），返回False
def open_position(security, value):
    order = order_target_value_(security, value)
    if order != None and order.filled > 0:
        # 报单成功并有成交则初始化最高价
        cur_price = get_close_price(security, 1, '1m')
        g.last_high[security] = cur_price
        return True
    return False


# 平仓，卖出指定持仓
# 平仓成功并全部成交，返回True
# 报单失败或者报单成功但被取消（此时成交量等于0），或者报单非全部成交，返回False
def close_position(position):
    security = position.security
    order = order_target_value_(security, 0)  # 可能会因停牌失败
    if order != None:
        if order.filled > 0:
            # 只要有成交，无论全部成交还是部分成交，则统计盈亏
            g.trade_stat.watch(security, order.filled, position.avg_cost, position.price)

        if order.status == OrderStatus.held and order.filled == order.amount:
            # 全部成交则删除相关证券的最高价缓存
            if security in g.last_high:
                g.last_high.pop(security)
            else:
                log.warn("last high price of %s not found" % (security))
            return True

    return False


def open_short(context):
    futures_name = "IF"
    point_value = 300
    # 本月和下月合约代码
    codeThis = get_stock_index_futrue_code(context, futures_name)
    codeNext = get_stock_index_futrue_code(context, futures_name, 'next_month')
    # 进入本月的第3周，则换下月合约，否则都做本月合约
    weekofmonth = int(context.current_dt.strftime('%W')) - int(context.current_dt.replace(day=1).strftime('%W')) + 1
    if weekofmonth == 3:
        codeTarget = codeNext
    else:
        codeTarget = codeThis

    # 若当前合约不是codeTarget，则平仓
    if len(context.subportfolios[1].short_positions) > 0:
        codeNow = context.subportfolios[1].short_positions.keys()[0]
        if codeNow <> codeTarget:
            log.info('股指期货%s平仓' % codeNow)
            order_target(codeNow, 0, side='short', pindex=1)  # 平旧仓

    # 开新仓: 初略地，100万可用现金，开空单1手
    myCash = context.portfolio.transferable_cash
    if myCash > 1000000:
        transfer_cash(0, 1, context.subportfolios[0].transferable_cash)  # 从股票仓调入期货仓
        amount = int(myCash / 1000000.0)
        log.info('股指期货%s开仓' % codeTarget)
        order_target(codeTarget, amount, side='short', pindex=1)


def close_short(context):  # 股指期货空单平仓 Jiang
    # 平仓
    # if len(context.subportfolios[0].short_positions)==0:

    if len(context.subportfolios[1].short_positions) > 0:
        for codeNow in context.subportfolios[1].short_positions:
            if context.subportfolios[1].short_positions[codeNow].closeable_amount > 0:
                log.info('股指期货%s平仓' % codeNow)
                order_target(codeNow, 0, side='short', pindex=1)  # 平旧仓
        # 资金调入股票仓
        myCash = context.subportfolios[1].transferable_cash
        transfer_cash(1, 0, myCash)


def get_stock_index_futrue_code(context, symbol, month='this_month'):
    '''
    获取当天时间正在交易的股指期货合约。其中:
    symbol:
            'IF' #沪深300指数期货
            'IC' #中证500股指期货
            'IH' #上证50股指期货
    month:
            'this_month' #当月
            'next_month' #隔月
            'next_quart' #下季
            'skip_quart' #隔季
    '''
    display_name_dict = {'IC': '中证500股指期货', 'IF': '沪深300指数期货', 'IH': '上证50股指期货'}
    month_dict = {'this_month': 0, 'next_month': 1, 'next_quart': 2, 'skip_quart': 3}

    display_name = display_name_dict[symbol]
    n = month_dict[month]
    dt = context.current_dt.date()
    a = get_all_securities(types=['futures'], date=dt)
    try:
        df = a[(a.display_name == display_name) & (a.start_date <= dt) & (a.end_date >= dt)]
        if (len(df) > 4) and (month in ('next_quart', 'skip_quart')):
            return df.index[n + 1]
        else:
            return df.index[n]
    except:
        return 'WARRING: 无此合约'


# 清空卖出所有持仓
def clear_position(context):
    if context.portfolio.positions:
        log.info("==> 清仓，卖出所有股票")
        for stock in context.portfolio.positions.keys():
            position = context.portfolio.positions[stock]
            close_position(position)
    # open_short(context)


# 自定义下单
# 根据Joinquant文档，当前报单函数都是阻塞执行，报单函数（如order_target_value）返回即表示报单完成
# 报单成功返回报单（不代表一定会成交），否则返回None
def order_target_value_(security, value):
    if value == 0:
        log.debug("Selling out %s" % (security))
    else:
        log.debug("Order %s to value %f" % (security, value))

    # 如果股票停牌，创建报单会失败，order_target_value 返回None
    # 如果股票涨跌停，创建报单会成功，order_target_value 返回Order，但是报单会取消
    # 部成部撤的报单，聚宽状态是已撤，此时成交量>0，可通过成交量判断是否有成交
    return order_target_value(security, value)


# 过滤停牌股票
def filter_paused_stock(stock_list):
    current_data = get_current_data()
    return [stock for stock in stock_list if not current_data[stock].paused]


# 过滤ST及其他具有退市标签的股票
def filter_st_stock(stock_list):
    current_data = get_current_data()
    return [stock for stock in stock_list
            if not current_data[stock].is_st
            and 'ST' not in current_data[stock].name
            and '*' not in current_data[stock].name
            and '退' not in current_data[stock].name]


# 过滤涨停的股票
def filter_limitup_stock(context, stock_list):
    last_prices = history(1, unit='1m', field='close', security_list=stock_list)
    current_data = get_current_data()

    # 已存在于持仓的股票即使涨停也不过滤，避免此股票再次可买，但因被过滤而导致选择别的股票
    return [stock for stock in stock_list if stock in context.portfolio.positions.keys()
            or last_prices[stock][-1] < current_data[stock].high_limit]
    # return [stock for stock in stock_list if stock in context.portfolio.positions.keys()
    #    or last_prices[stock][-1] < current_data[stock].high_limit * 0.995]


# 过滤跌停的股票
def filter_limitdown_stock(context, stock_list):
    last_prices = history(1, unit='1m', field='close', security_list=stock_list)
    current_data = get_current_data()

    return [stock for stock in stock_list if stock in context.portfolio.positions.keys()
            or last_prices[stock][-1] > current_data[stock].low_limit]
    # return [stock for stock in stock_list if last_prices[stock][-1] > current_data[stock].low_limit]
    # return [stock for stock in stock_list if stock in context.portfolio.positions.keys()
    #    or last_prices[stock][-1] > current_data[stock].low_limit * 1.005]


# 过滤黑名单股票
def filter_blacklist_stock(context, stock_list):
    blacklist = get_blacklist()
    return [stock for stock in stock_list if stock not in blacklist]


# 过滤创业版股票
def filter_gem_stock(context, stock_list):
    return [stock for stock in stock_list if stock[0:3] != '300']


# 过滤20日增长率为负的股票
def filter_by_growth_rate(stock_list, n):
    return [stock for stock in stock_list if get_growth_rate(stock, n) > 0]


# 股票评分
def rank_stocks(data, stock_list):
    dst_stocks = {}
    for stock in stock_list:
        h = attribute_history(stock, 130, unit='1d', fields=('close', 'high', 'low'), skip_paused=True)
        low_price_130 = h.low.min()
        high_price_130 = h.high.max()

        avg_15 = data[stock].mavg(15, field='close')
        cur_price = data[stock].close

        # avg_15 = h['close'][-15:].mean()
        # cur_price = get_close_price(stock, 1, '1m')

        score = (cur_price - low_price_130) + 0.5 * (cur_price - high_price_130) + 1.1 * (cur_price - avg_15)
        # score = ((cur_price-low_price_130) + (cur_price-high_price_130) + (cur_price-avg_15)) / cur_price
        dst_stocks[stock] = score

    df = pd.DataFrame(dst_stocks.values(), index=dst_stocks.keys())
    df.columns = ['score']
    df = df.sort(columns='score', ascending=True)
    return df.index


'''
# 过滤新股
def filter_new_stock(stock_list):
    stocks = get_all_securities(['stock'])
    stocks = stocks[(context.current_dt.date() - stocks.start_date) > datetime.timedelta(60)].index
'''


def pick_stocks2(context, data):
    # get_security_info('000300.XSHG')
    q = None
    if g.pick_by_pe:
        if g.pick_by_eps:
            q = query(valuation.code).filter(
                valuation.code.in_(get_index_stocks('000300.XSHG')),
                indicator.eps > g.min_eps,
                valuation.pe_ratio > g.min_pe,
                valuation.pe_ratio < g.max_pe

            ).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )
        else:
            q = query(valuation.code).filter(
                valuation.code.in_(get_index_stocks('000300.XSHG')),
                valuation.pe_ratio > g.min_pe,
                valuation.pe_ratio < g.max_pe
            ).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )
    else:
        if g.pick_by_eps:
            q = query(valuation.code).filter(
                valuation.code.in_(get_index_stocks('000300.XSHG')),
                indicator.eps > g.min_eps
            ).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )
        else:
            q = query(valuation.code).order_by(
                # valuation.code in get_index_stocks('000300.XSHG'),
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )

    df = get_fundamentals(q)
    # stock_list = list(df['code'])
    stock_list = get_index_stocks('000016.XSHG')
    if g.filter_gem:
        stock_list = filter_gem_stock(context, stock_list)

    if g.filter_blacklist:
        stock_list = filter_blacklist_stock(context, stock_list)

    stock_list = filter_paused_stock(stock_list)
    stock_list = filter_st_stock(stock_list)
    stock_list = filter_limitup_stock(context, stock_list)
    stock_list = filter_limitdown_stock(context, stock_list)

    # 根据20日股票涨幅过滤效果不好，故注释
    # stock_list = filter_by_growth_rate(stock_list, 15)

    if g.is_rank_stock:
        # if len(stock_list) > g.rank_stock_count:
        #    stock_list = stock_list[:50]

        # log.debug("评分前备选股票: %s" %(stock_list))
        if len(stock_list) > 0:
            stock_list = rank_stocks(data, stock_list)
        # log.debug("评分后备选股票: %s" %(stock_list))

    # 选取指定可买数目的股票
    if len(stock_list) > g.buy_stock_count:
        stock_list = stock_list[:g.buy_stock_count]
    return stock_list


# 选股
# 选取指定数目的小市值股票，再进行过滤，最终挑选指定可买数目的股票
def pick_stocks(context, data):
    q = None
    if g.pick_by_pe:
        if g.pick_by_eps:
            q = query(valuation.code).filter(
                indicator.eps > g.min_eps,
                valuation.pe_ratio > g.min_pe,
                valuation.pe_ratio < g.max_pe
            ).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )
        else:
            q = query(valuation.code).filter(
                valuation.pe_ratio > g.min_pe,
                valuation.pe_ratio < g.max_pe
            ).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )
    else:
        if g.pick_by_eps:
            q = query(valuation.code).filter(
                indicator.eps > g.min_eps
            ).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )
        else:
            q = query(valuation.code).order_by(
                valuation.market_cap.asc()
            ).limit(
                g.pick_stock_count
            )

    df = get_fundamentals(q)
    stock_list = list(df['code'])

    if g.filter_gem:
        stock_list = filter_gem_stock(context, stock_list)

    if g.filter_blacklist:
        stock_list = filter_blacklist_stock(context, stock_list)

    stock_list = filter_paused_stock(stock_list)
    stock_list = filter_st_stock(stock_list)
    stock_list = filter_limitup_stock(context, stock_list)
    stock_list = filter_limitdown_stock(context, stock_list)

    # 根据20日股票涨幅过滤效果不好，故注释
    # stock_list = filter_by_growth_rate(stock_list, 15)

    if g.is_rank_stock:
        if len(stock_list) > g.rank_stock_count:
            stock_list = stock_list[:g.rank_stock_count]

        # log.debug("评分前备选股票: %s" %(stock_list))
        if len(stock_list) > 0:
            stock_list = rank_stocks(data, stock_list)
        # log.debug("评分后备选股票: %s" %(stock_list))

    # 选取指定可买数目的股票
    if len(stock_list) > g.buy_stock_count:
        stock_list = stock_list[:g.buy_stock_count]
    return stock_list


# 根据待买股票创建或调整仓位
# 对于因停牌等原因没有卖出的股票则继续持有
# 始终保持持仓数目为g.buy_stock_count
def adjust_position(context, buy_stocks):
    for stock in context.portfolio.positions.keys():
        if stock not in buy_stocks:
            log.info("stock [%s] in position is not buyable" % (stock))
            position = context.portfolio.positions[stock]
            close_position(position)
        else:
            log.info("stock [%s] is already in position" % (stock))

    # 根据股票数量分仓
    # 此处只根据可用金额平均分配购买，不能保证每个仓位平均分配
    position_count = len(context.portfolio.positions)
    if g.buy_stock_count > position_count:
        value = context.portfolio.cash / (g.buy_stock_count - position_count)

        for stock in buy_stocks:
            if context.portfolio.positions[stock].total_amount == 0:
                if open_position(stock, value):
                    if len(context.portfolio.positions) == g.buy_stock_count:
                        break


def adjust_position2(context, buy_stocks):
    for stock in context.portfolio.positions.keys():
        if stock not in buy_stocks:
            log.info("stock [%s] in position is not buyable" % (stock))
            position = context.portfolio.positions[stock]
            close_position(position)
        else:
            log.info("stock [%s] is already in position" % (stock))

    # 根据股票数量分仓
    # 此处只根据可用金额平均分配购买，不能保证每个仓位平均分配
    position_count = len(context.portfolio.positions)
    if g.buy_stock_count > position_count:
        value = context.portfolio.cash

        for stock in buy_stocks:
            if context.portfolio.positions[stock].total_amount == 0:
                if open_position(stock, value):
                    if len(context.portfolio.positions) == g.buy_stock_count:
                        break


'''
--------------------------------------------------------------------------------


