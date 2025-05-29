# ==========================
# TELEGRAM BOT ALL-IN-ONE FULL
# (Tín hiệu tự động + Funding + Tra giá + Compare + Buy/Sell + Trend)
# ==========================

import logging
import httpx
import asyncio
import pandas as pd
from datetime import datetime
from dateutil import tz
from telegram import Update, constants
from telegram.ext import (
    Application, CommandHandler, MessageHandler, filters,
    CallbackContext, ContextTypes
)

TOKEN = "8199310023:AAEoFyIGUFYCsutqiqd0D2jQxWV4znMcD6U"
OWNER_ID = 1191586860
ALLOWED_GROUP_IDS = [-1001749552228, -1001813759468, -1001901072347]
SIGNAL_CHANNEL_ID = -1002127588410
VALID_INTERVALS = ["1m", "5m", "15m", "30m", "60m", "120m", "180m", "1h", "4h", "1d"]

def convert_to_local_time(timestamp):
    from_zone = tz.gettz("UTC")
    to_zone = tz.gettz("Asia/Ho_Chi_Minh")
    utc = datetime.fromtimestamp(timestamp)
    utc = utc.replace(tzinfo=from_zone)
    return utc.astimezone(to_zone).strftime("%H:%M")

def pretty_price(val):
    val = float(val)
    if val >= 1:
        return f"{val:,.2f}"
    elif val >= 0.01:
        return f"{val:.4f}".rstrip('0').rstrip('.')
    else:
        return f"{val:.8f}".rstrip('0').rstrip('.')

async def fetch_api_data(url):
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp.json()

async def get_all_coins():
    try:
        url = "https://api.mexc.com/api/v3/exchangeInfo"
        data = await fetch_api_data(url)
        coins = set()
        for item in data["symbols"]:
            if (
                item["quoteAsset"] == "USDT"
                and item["status"] == "ENABLED"
                and item["isSpotTradingAllowed"]
            ):
                coins.add(item["symbol"])  # lấy nguyên symbol như BTCUSDT
        return coins
    except Exception as e:
        logging.error(f"Error fetching all coins: {e}")
        return set()

# =========== FUNDING ===========
async def fetch_funding_data():
    try:
        funding_url = "https://contract.mexc.com/api/v1/contract/funding_rate"
        detail_url = "https://contract.mexc.com/api/v1/contract/detail"
        funding_data = await fetch_api_data(funding_url)
        detail_data = await fetch_api_data(detail_url)
        details_dict = {token["symbol"]: token for token in detail_data["data"]}
        top_funding = sorted(
            [x for x in funding_data["data"] if abs(x["fundingRate"]) >= 0.005],
            key=lambda x: x["fundingRate"],
        )
        return top_funding, details_dict
    except Exception as e:
        logging.error(f"Error fetching funding data: {e}")
        return None, None

async def funding_handler(update: Update, context: CallbackContext):
    if not update.message:
        return
    # Chặn riêng tư trừ OWNER
    if update.message.chat.type == "private" and update.message.from_user.id != OWNER_ID:
        return
    if update.message.chat.type in ["group", "supergroup", "channel"] and update.message.chat_id not in ALLOWED_GROUP_IDS:
        return
    top_funding, details_dict = await fetch_funding_data()
    if not top_funding:
        await update.message.reply_text(
            "🔥 <b>TOP FUNDING FEE 🔥</b>\nHiện không có funding rate đáng chú ý.",
            parse_mode=constants.ParseMode.HTML,
        )
        return
    response_text = "<b>🔥 TOP FUNDING FEE 🔥</b>\n"
    for index, item in enumerate(top_funding, start=1):
        token = details_dict.get(item["symbol"])
        if token:
            funding_rate = item["fundingRate"] * 100
            leverage = token["maxLeverage"]
            settlement_time = convert_to_local_time(item["nextSettleTime"] / 1000.0)
            response_text += f"\n<b>{index}. {item['symbol']} | x{leverage} | {settlement_time} | {round(funding_rate, 2)}%</b>\n"
    response_text += (
        "\n<b>Lưu ý:</b>\n"
        "- <i>Phí Funding âm => Long 🟢</i>\n"
        "- <i>Phí Funding dương => Short 🔴</i>\n\n/funding"
    )
    await update.message.reply_text(response_text, parse_mode=constants.ParseMode.HTML)

# =========== TÍN HIỆU TỰ ĐỘNG ===========
import numpy as np
from datetime import datetime, timedelta

def seconds_to_next_50min():
    now = datetime.now()
    if now.minute >= 50:
        # Chuyển sang giờ sau, phút 50
        next_hour = (now + timedelta(hours=1)).replace(minute=50, second=0, microsecond=0)
        wait_seconds = (next_hour - now).total_seconds()
    else:
        next_50 = now.replace(minute=50, second=0, microsecond=0)
        wait_seconds = (next_50 - now).total_seconds()
    return max(1, int(wait_seconds))

async def auto_signal_task(app: Application):
    import math

    async def get_klines(symbol, interval="1h", limit=50):
        url = f"https://api.mexc.com/api/v3/klines?symbol={symbol}&interval={interval}&limit={limit}"
        try:
            data = await fetch_api_data(url)
            if not data or len(data) < 21:
                return None
            df = pd.DataFrame(data, columns=[
                "timestamp", "open", "high", "low", "close", "volume", "close_time", "ignore"
            ])
            df["open"] = df["open"].astype(float)
            df["high"] = df["high"].astype(float)
            df["low"] = df["low"].astype(float)
            df["close"] = df["close"].astype(float)
            df["volume"] = df["volume"].astype(float)
            return df
        except Exception as e:
            logging.error(f"Lỗi fetch kline {symbol}: {e}")
            return None

    def calc_rsi(prices, period=14):
        delta = prices.diff()
        up = delta.clip(lower=0)
        down = -1 * delta.clip(upper=0)
        avg_gain = up.rolling(window=period, min_periods=period).mean()
        avg_loss = down.rolling(window=period, min_periods=period).mean()
        rs = avg_gain / (avg_loss + 1e-8)
        rsi = 100 - (100 / (1 + rs))
        return rsi

    def bollinger_bands(series, window=20, num_std=2):
        ma = series.rolling(window).mean()
        std = series.rolling(window).std()
        upper = ma + num_std * std
        lower = ma - num_std * std
        return upper, ma, lower

    def check_confirm_volume(df, threshold=1.1):
        latest_vol = df["volume"].iloc[-1]
        ma20 = df["volume"].rolling(20).mean().iloc[-1]
        return latest_vol > (ma20 * threshold)

    def find_rsi_bullish_divergence(df, threshold=25, lookback=20):
        rsi = df["RSI"]
        price = df["close"]
        for i in range(len(df) - 2, len(df) - lookback - 1, -1):
            if (price.iloc[-2] < price.iloc[i] and
                rsi.iloc[-2] > rsi.iloc[i] and
                rsi.iloc[-2] < threshold):
                return True
        return False

    def find_rsi_bearish_divergence(df, threshold=75, lookback=20):
        rsi = df["RSI"]
        price = df["close"]
        for i in range(len(df) - 2, len(df) - lookback - 1, -1):
            if (price.iloc[-2] > price.iloc[i] and
                rsi.iloc[-2] < rsi.iloc[i] and
                rsi.iloc[-2] > threshold):
                return True
        return False

    def calc_percent(entry, other):
        try:
            return round((other - entry) / entry * 100, 2)
        except:
            return 0.0

    while True:
        try:
            INTERVAL = "1h"
            coins = await get_all_coins()
            for symbol in coins:
                pair = f"{symbol}USDT"
                df = await get_klines(pair, interval=INTERVAL, limit=50)
                if df is None or len(df) < 21:
                    continue

                df["RSI"] = calc_rsi(df["close"])
                df["RSI"] = df["RSI"].round(2)
                upper, mid, lower = bollinger_bands(df["close"])
                df["BollUpper"] = upper
                df["BollMid"] = mid
                df["BollLower"] = lower

                confirm_vol = check_confirm_volume(df)

                # ==== LONG (Bullish) ====
                bullish_div = find_rsi_bullish_divergence(df)
                last_rsi = df["RSI"].iloc[-2]
                last_close = df["close"].iloc[-2]
                boll_support = last_close <= df["BollLower"].iloc[-2]
                if bullish_div and confirm_vol and last_rsi < 30 and boll_support:
                    entry = df["close"].iloc[-1]
                    sl = min(df["low"].iloc[-5:])  # SL là đáy gần nhất 5 nến
                    tp = entry + (entry - sl)     # TP lấy theo RR 1:1 (tùy chỉnh)
                    percent_sl = calc_percent(entry, sl)
                    percent_tp = calc_percent(entry, tp)
                    text = (
                        f"🟢 TÍN HIỆU LONG - {symbol}USDT 🟢\n\n"
                        f"🔎 Đủ điều kiện\n\n"
                        f"✨ ENTRY: {pretty_price(entry)} USDT\n"
                        f"🛡️ SL: {pretty_price(sl)} USDT ({percent_sl:+.2f}%)\n"
                        f"🎯 TP: {pretty_price(tp)} USDT ({percent_tp:+.2f}%) hoặc tuỳ mồm\n\n"
                        f"Lưu ý:\n"
                        f"- Chỉ vào lệnh khi đã kiểm tra lại chart tổng thể!\n"
                        f"- Quản lý vốn & kỷ luật."
                    )
                    await app.bot.send_message(chat_id=SIGNAL_CHANNEL_ID, text=text)
                    await asyncio.sleep(1)

                # ==== SHORT (Bearish) ====
                bearish_div = find_rsi_bearish_divergence(df)
                last_rsi_b = df["RSI"].iloc[-2]
                last_close_b = df["close"].iloc[-2]
                boll_resist = last_close_b >= df["BollUpper"].iloc[-2]
                if bearish_div and confirm_vol and last_rsi_b > 70 and boll_resist:
                    entry = df["close"].iloc[-1]
                    sl = max(df["high"].iloc[-5:])  # SL là đỉnh gần nhất 5 nến
                    tp = entry - (sl - entry)       # TP lấy theo RR 1:1
                    percent_sl = calc_percent(entry, sl)
                    percent_tp = calc_percent(entry, tp)
                    text = (
                        f"🔴 TÍN HIỆU SHORT - {symbol}USDT 🔴\n\n"
                        f"🔎 Đủ điều kiện\n\n"
                        f"✨ ENTRY: {pretty_price(entry)} USDT\n"
                        f"🛡️ SL: {pretty_price(sl)} USDT ({percent_sl:+.2f}%)\n"
                        f"🎯 TP: {pretty_price(tp)} USDT ({percent_tp:+.2f}%) hoặc tuỳ mồm\n\n"
                        f"Lưu ý:\n"
                        f"- Chỉ vào lệnh khi đã kiểm tra lại chart tổng thể!\n"
                        f"- Quản lý vốn & kỷ luật."
                    )
                    await app.bot.send_message(chat_id=SIGNAL_CHANNEL_ID, text=text)
                    await asyncio.sleep(1)

            # Đợi đúng đến phút 50 giờ tiếp theo
            wait_seconds = seconds_to_next_50min()
            print(f"[AutoSignal] Đợi {wait_seconds}s để đến phút 50.")
            await asyncio.sleep(wait_seconds)

        except Exception as e:
            logging.error(f"[AutoSignal] {e}")
            await asyncio.sleep(30)

# =========== LỆNH CHÍNH ===========
def is_valid_command_text(text):
    text = text.lower().strip()
    if text.startswith(("buy ", "sell ", "sosanh ", "compare ", "trend ")):
        return True
    if len(text.split()) == 1 and text.isalnum():
        return True
    return False

async def handle_message(update: Update, context: CallbackContext):
    if not update.message:
        return
    if update.message.chat.type == "private" and update.message.from_user.id != OWNER_ID:
        return
    if update.message.chat.type in ["group", "supergroup", "channel"] and update.message.chat_id not in ALLOWED_GROUP_IDS:
        return
    user_text = update.message.text.strip()
    if not is_valid_command_text(user_text):
        return
    args = user_text.split()
    # ==== LỆNH TRA GIÁ COIN ====
    if len(args) == 1 and args[0].isalnum():
        symbol = args[0].upper()
        url = f"https://api.mexc.com/api/v3/ticker/24hr?symbol={symbol}USDT"
        try:
            resp = await fetch_api_data(url)
            if "lastPrice" not in resp or "openPrice" not in resp:
                return
            last_price = float(resp["lastPrice"])
            open_price = float(resp["openPrice"])
            high_price = float(resp["highPrice"])
            low_price = float(resp["lowPrice"])
            price_change_percent = ((last_price - open_price) / open_price) * 100

            msg = (
                f"<b>{symbol}/USDT trên MEXC</b>\n\n"
                f"💰 <b>Giá:</b> {pretty_price(last_price)} USDT\n"
                f"📈 <b>Cao nhất (24h):</b> {pretty_price(high_price)} USDT\n"
                f"📉 <b>Thấp nhất (24h):</b> {pretty_price(low_price)} USDT\n"
                f"📊 <b>Thay đổi (24h):</b> {price_change_percent:+.2f}%"
            )
        except Exception:
            return
        await update.message.reply_text(msg, parse_mode="HTML")
        return

    # ==== LỆNH SO SÁNH ====
    if args and args[0].lower() in ["sosanh", "compare"] and len(args) >= 6:
        try:
            coin1 = args[1].upper()
            coin2 = args[2].upper()
            price1 = float(args[3].replace('$', '').replace(',', ''))
            price2 = float(args[4].replace('$', '').replace(',', ''))
            capital = float(args[5].replace('$', '').replace(',', ''))
            url1 = f"https://api.mexc.com/api/v3/ticker/24hr?symbol={coin1}USDT"
            url2 = f"https://api.mexc.com/api/v3/ticker/24hr?symbol={coin2}USDT"
            info1 = await fetch_api_data(url1)
            info2 = await fetch_api_data(url2)
            now1 = float(info1['lastPrice'])
            now2 = float(info2['lastPrice'])
            qty1 = capital / price1
            qty2 = capital / price2
            profit1 = (now1 - price1) * qty1
            percent1 = ((now1 - price1) / price1) * 100
            profit2 = (now2 - price2) * qty2
            percent2 = ((now2 - price2) / price2) * 100
            msg = (
                "```\n"
                f"Buy   #{coin1:<4}      ${price1:,.0f}\n"
                f"With            ${capital:,.0f}\n"
                f"Now             ${now1:,.0f}\n"
                f"Revenue    {percent1:+.2f}% {profit1:+,.1f}$\n"
                "---------------------------\n"
                f"Buy   #{coin2:<4}      ${price2:,.0f}\n"
                f"With            ${capital:,.0f}\n"
                f"Now             ${now2:,.0f}\n"
                f"Revenue    {percent2:+.2f}% {profit2:+,.1f}$\n"
                "```"
            )
            await update.message.reply_text(msg, parse_mode="Markdown")
            USDT2VND = 25500
            vnd1 = now1 * USDT2VND
            vnd2 = now2 * USDT2VND
            await update.message.reply_text(f"💵 {coin1}: {vnd1:,.0f} VNĐ\n💵 {coin2}: {vnd2:,.0f} VNĐ")
        except Exception:
            await update.message.reply_text("Không thể lấy giá coin.")
        return
    # ==== LỆNH BUY/SELL ====
    if args and args[0].lower() in ["buy", "sell"]:
        try:
            action = args[0].capitalize()
            symbol = args[1].upper()
            url = f"https://api.mexc.com/api/v3/ticker/24hr?symbol={symbol}USDT"
            coin_info = await fetch_api_data(url)
            if not coin_info or 'lastPrice' not in coin_info:
                return  # KHÔNG gửi trả lời gì cả nếu không có dữ liệu
            now_price = float(coin_info['lastPrice'])

            # Nếu chỉ có 3 tham số: buy/sell symbol price
            if len(args) == 3:
                buy_price = float(args[2].replace('$', '').replace(',', ''))
                icon = "🟢" if action == "Buy" else "🔴"
                title = f"{'LỆNH MUA' if action == 'Buy' else 'LỆNH BÁN'} {symbol}"
                percent = ((now_price - buy_price) / buy_price * 100) if action == "Buy" else ((buy_price - now_price) / buy_price * 100)
                msg = (
                    "```\n"
                    f"{icon} {title}\n"
                    f"Giá vào    : {pretty_price(buy_price)} USDT\n"
                    "------------------------\n"
                    f"Giá hiện tại: {pretty_price(now_price)} USDT\n"
                    f"Lãi/Lỗ     : {percent:+.2f}%\n"
                    "```"
                )
                await update.message.reply_text(msg, parse_mode="Markdown")
                return

            # Nếu nhập thêm số vốn (tức là đủ >= 4 tham số) thì xử lý như cũ
            buy_price = float(args[2].replace('$', '').replace(',', ''))
            capital = float(args[3].replace('$', '').replace(',', ''))
            leverage = 1
            for arg in args:
                if arg.lower().startswith('x'):
                    try:
                        leverage = int(arg.lower().replace('x', ''))
                    except:
                        pass
            fee_rate = 0.0005 if symbol == "BTC" else 0.0002
            amount = capital * leverage / buy_price
            def parse_stop_tp(val, buy_price, side, tp_or_sl):
                try:
                    if isinstance(val, str) and val.endswith("%"):
                        pct = float(val[:-1]) / 100
                        if side == "Buy":
                            if tp_or_sl == "tp":
                                return buy_price * (1 + pct)
                            else:
                                return buy_price * (1 - pct)
                        else:
                            if tp_or_sl == "tp":
                                return buy_price * (1 - pct)
                            else:
                                return buy_price * (1 + pct)
                    return float(val.replace('$', '').replace(',', ''))
                except:
                    return None
            stoploss = None
            takeprofit = None
            for idx, arg in enumerate(args):
                if arg.lower() in ["stop", "sl"] and idx + 1 < len(args):
                    stoploss = parse_stop_tp(args[idx+1], buy_price, action, "sl")
                if arg.lower() == "tp" and idx + 1 < len(args):
                    takeprofit = parse_stop_tp(args[idx+1], buy_price, action, "tp")
            def calc_result(entry, exit_price, qty, side, leverage, fee_rate):
                if side == "Buy":
                    raw = (exit_price - entry) * qty * leverage
                else:
                    raw = (entry - exit_price) * qty * leverage
                fee = (entry + exit_price) * abs(qty) * fee_rate / 2
                percent = ((exit_price - entry) / entry * 100) * leverage if side == "Buy" else ((entry - exit_price) / entry * 100) * leverage
                return raw - fee, percent, fee
            revenue_now, percent_now, fee_now = calc_result(buy_price, now_price, amount, action, leverage, fee_rate)
            if stoploss:
                revenue_stop, percent_stop, _ = calc_result(buy_price, stoploss, amount, action, leverage, fee_rate)
            else:
                revenue_stop = percent_stop = None
            if takeprofit:
                revenue_tp, percent_tp, _ = calc_result(buy_price, takeprofit, amount, action, leverage, fee_rate)
            else:
                revenue_tp = percent_tp = None
            icon = "🟢" if action == "Buy" else "🔴"
            title = f"LỆNH {'MUA' if action == 'Buy' else 'BÁN'} {symbol}"
            msg = (
                "```\n"
                f"{icon} {title}\n"
                f"Giá vào    : ${buy_price:,.0f}\n"
                f"Vốn        : ${capital:,.0f}\n"
                f"Đòn bẩy    : x{leverage}\n"
                "------------------------\n"
                f"Giá hiện tại: ${now_price:,.0f}\n"
                f"Lãi/Lỗ     : {percent_now:+.2f}% = {revenue_now:+,.0f}$\n"
                f"Fee        : ${fee_now:,.2f}\n"
            )
            if stoploss:
                msg += f"SL        {percent_stop:+.2f}% {revenue_stop:+,.1f}$\n"
            if takeprofit:
                msg += f"TP        {percent_tp:+.2f}% {revenue_tp:+,.1f}$\n"
            msg += "```"
            await update.message.reply_text(msg, parse_mode="Markdown")
            USDT2VND = 25500
            vnd_price = now_price * USDT2VND
            vnd_msg = f"💵 Giá hiện tại (VNĐ): {vnd_price:,.0f} VNĐ"
            await update.message.reply_text(vnd_msg)
        except Exception as e:
            await update.message.reply_text(f"Lỗi khi xử lý BUY/SELL: {e}")
        return

    # ==== LỆNH TREND ====
    if args and args[0].lower() == "trend" and len(args) > 1:
        try:
            symbol = args[1].upper()
            import numpy as np
            url = f"https://api.binance.com/api/v3/klines?symbol={symbol}USDT&interval=4h&limit=100"
            async with httpx.AsyncClient() as client:
                resp = await client.get(url)
                data = resp.json()
            closes = np.array([float(c[4]) for c in data if isinstance(c, (list, tuple)) and len(c) > 4 and c[4] != ''])
            ma20 = np.mean(closes[-20:])
            ma50 = np.mean(closes[-50:])
            uptrend = ma20 > ma50
            delta = closes[1:] - closes[:-1]
            up = delta.clip(min=0)
            down = -delta.clip(max=0)
            rs = np.mean(up[-14:]) / (np.mean(down[-14:]) + 1e-9)
            rsi = 100 - (100 / (1 + rs))
            trend_str = "Uptrend ⬆️" if uptrend else "Downtrend ⬇️"
            msg = (
                f"<b>{symbol} 4H Trend</b>\n"
                f"MA20: <b>{ma20:.2f}</b> | MA50: <b>{ma50:.2f}</b>\n"
                f"Trend: <b>{trend_str}</b>\n"
                f"RSI(14): <b>{rsi:.2f}</b>"
            )
            await update.message.reply_text(msg, parse_mode="HTML")
        except Exception as e:
            await update.message.reply_text(f"Lỗi khi tính trend: {e}")
        return

# =========== ERROR HANDLER ===========
async def error_handler(update, context):
    logging.error(f"Update {update} caused error {context.error}")

# =========== MAIN ===========
def main():
    logging.basicConfig(level=logging.INFO)
    application = Application.builder().token(TOKEN).build()
    application.add_handler(CommandHandler("funding", funding_handler))
    application.add_handler(MessageHandler(filters.TEXT, handle_message))
    application.add_error_handler(error_handler)
    # Start auto signal task khi bot start
    async def run_signal(ctx):
        await auto_signal_task(application)
    application.job_queue.run_once(lambda ctx: asyncio.create_task(run_signal(ctx)), when=1)
    application.run_polling()

if __name__ == "__main__":
    main()
