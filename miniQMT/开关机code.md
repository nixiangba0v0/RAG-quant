# 开机

```python
# ==== Cell A: 环境与全局配置 ====
import os, sys, time, subprocess
from threading import Thread
from xtquant import xtdata
from xtquant.xttrader import XtQuantTrader, XtQuantTraderCallback
from xtquant.xttype import StockAccount

# —— 路径与账号（按需修改）——
QMT_USERDATA_PATH = r"D:\QMT_dbzq\userdata_mini"
MINIQMT_EXE_PATH  = r"D:\QMT_dbzq\bin.x64\XtMiniQmt.exe"
ACCOUNT_ID        = "11100014"     # 示例
SESSION_ID        = 123455         # 同机并发策略需唯一

# —— 行情订阅参数（按需修改）——
SUB_CODES      = ["600000.SH"]
KLINE_PERIOD   = "1m"              # '1m'/'5m'/'1d' 等

# —— 控制项 —— 
AUTO_START_MINIQMT = True

# —— 运行期全局对象 —— 
process = None           # XtMiniQmt 进程对象（若由脚本启动）
xt_trader = None         # XtQuantTrader 实例
loop_thread = None       # 后台 run_forever 线程
acc = None               # 资金账号对象
quote_sub_ids = []       # 行情订阅号，用于反订阅

print("Python:", sys.version)
print("xtquant 载入 OK；路径存在：",
      os.path.exists(QMT_USERDATA_PATH), os.path.exists(MINIQMT_EXE_PATH))
# 可选：快速写权限测试（Windows上管理员/盘符权限易踩坑）
try:
    _testfile = os.path.join(QMT_USERDATA_PATH, "~perm_test.txt")
    with open(_testfile, "w") as f: f.write("ok")
    os.remove(_testfile)
    print("userdata_mini 写权限 OK")
except Exception as e:
    print("⚠️ userdata_mini 写权限异常：", e)


```

```python
# ==== Cell B: 启动 MiniQMT（可选） ====
if AUTO_START_MINIQMT:
    if not os.path.exists(MINIQMT_EXE_PATH):
        raise FileNotFoundError(f"未找到 XtMiniQmt.exe：{MINIQMT_EXE_PATH}")
    if process is None:
        process = subprocess.Popen(MINIQMT_EXE_PATH)
        print("已启动 XtMiniQmt.exe，请完成登录。")
    else:
        print("XtMiniQmt 已有进程对象，不重复启动。")
else:
    print("请手动启动并登录 XtMiniQMT，然后再继续。")

```

```python
# ==== Cell C: 定义回调类（轻量打印） ====
class MyCb(XtQuantTraderCallback):
    def on_disconnected(self): print("[回调] 连接断开")
    def on_stock_order(self, order): print("[回调] 委托", order.stock_code, order.order_status)
    def on_stock_trade(self, trade): print("[回调] 成交", trade.stock_code, trade.traded_volume)

print("回调类已就绪。")
```

```python
# ==== Cell D: 创建Trader并连接 ====
xt_trader = XtQuantTrader(QMT_USERDATA_PATH, SESSION_ID)
xt_trader.register_callback(MyCb())
xt_trader.start()

# 轻量重试，注意同一 session 两次 connect 需 ≥3 秒
max_retry = 10
for i in range(1, max_retry + 1):
    ret = xt_trader.connect()
    if ret == 0:
        print("✅ 连接成功。")
        break
    print(f"第 {i}/{max_retry} 次连接失败 ret={ret}，3秒后重试…")
    time.sleep(3)
else:
    raise SystemExit("❌ 连接失败，请检查：MiniQMT登录/路径/权限/session。")

```


```python
# ==== Cell E: 订阅账号 & 行情 ====
acc = StockAccount(ACCOUNT_ID)

# 订阅交易回推（资金/持仓/委托/成交）
sub_ret = xt_trader.subscribe(acc)
print("账号订阅返回：", sub_ret)

# 先（可选）补历史，再订阅实时；仅订阅实时时 count=0 即可
for code in SUB_CODES:
    try:
        # 可按需要注释掉下载历史
        from xtquant import xtdata
        xtdata.download_history_data(code, period=KLINE_PERIOD, start_time='20240101')
        sub_id = xtdata.subscribe_quote(code, period=KLINE_PERIOD, count=0)  # 仅实时
        quote_sub_ids.append(sub_id)
        print(f"{code} 订阅返回 sub_id={sub_id}")
    except Exception as e:
        print(f"{code} 订阅异常：", e)

# 拉一把最新K线确认
k = xtdata.get_market_data_ex([], SUB_CODES, period=KLINE_PERIOD)
print("get_market_data_ex 返回字段：", list(k.keys()) if isinstance(k, dict) else type(k))
print("取到的标的：", SUB_CODES)

```
```python
# ==== Cell F: 启动后台事件循环 ====
def _loop_forever():
    # run_forever 会阻塞该线程，直到 stop() 被调用
    xt_trader.run_forever()

if loop_thread is None or not loop_thread.is_alive():
    loop_thread = Thread(target=_loop_forever, daemon=True)
    loop_thread.start()
    time.sleep(0.5)
    print("✅ 后台事件循环启动：", loop_thread.is_alive())
else:
    print("后台事件循环已在运行。")
```


# 关机
```python
# ==== Cell G: 关机清理函数 ====
def graceful_shutdown():
    global xt_trader, loop_thread, process, acc, quote_sub_ids

    print("== 开始关机流程 ==")

    # 1) 行情反订阅（逐个 sub_id）
    if quote_sub_ids:
        for sid in list(quote_sub_ids):
            try:
                xtdata.unsubscribe_quote(sid)
                print("已反订阅行情 sub_id =", sid)
            except Exception as e:
                print("unsubscribe_quote 异常：", e)
        quote_sub_ids.clear()
    else:
        print("无行情订阅需要反订阅。")

    # 2) 交易反订阅账号
    try:
        if xt_trader is not None and acc is not None:
            ret = xt_trader.unsubscribe(acc)
            print("账号反订阅返回：", ret)
    except Exception as e:
        print("unsubscribe 异常：", e)

    # 3) 停止事件循环（让 run_forever 退出）
    try:
        if xt_trader is not None:
            xt_trader.stop()
            print("已请求停止 run_forever。")
    except Exception as e:
        print("stop 异常：", e)

    # 4) 等后台线程退出（最多等3秒）
    try:
        if loop_thread is not None:
            loop_thread.join(timeout=3)
            print("后台线程存活：", loop_thread.is_alive())
    except Exception as e:
        print("join 异常：", e)

    # 5) 断开交易连接
    try:
        if xt_trader is not None:
            xt_trader.disconnect()
            print("已断开交易连接。")
    except Exception as e:
        print("disconnect 异常：", e)

    # 6) 若由脚本启动 MiniQMT，则结束它
    try:
        if process is not None:
            process.terminate()
            print("已尝试终止 XtMiniQmt 进程。")
    except Exception as e:
        print("terminate 异常：", e)

    print("== 清理完成 ==")

print("graceful_shutdown() 已定义。")
```


```python
# ==== Cell H: 执行关机 ====
graceful_shutdown()
```
