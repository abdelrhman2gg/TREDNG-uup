# -*- coding: utf-8 -*-
"""
بوت تحليل فني احترافي للعملات الرقمية (عقود آجلة) على BingX V5.1 (مُعدَّل)
- يرسل تقارير دورية إلى Telegram (رسالة منفصلة لكل عملة).
- يحلل بيانات الشموع التاريخية (5 دقائق).
- يحسب مؤشرات متعددة (RSI, MACD, BBands, Stoch, SMAs, Ichimoku, ADX, ATR, CCI, PSAR, VWAP, OBV, CMF, MOM, ROC, SuperTrend,
  Fibonacci) وحجم التداول يدوياً.
- يوفر خيارات تنسيق متعددة للتقارير مع شروحات مبسطة.
- يقدم درجة تأكيد للإشارة بناءً على توافق المؤشرات الرئيسية.
- يوضح إشارات الشراء (Long) والبيع (Short).
- تصميم رسالة محسن مع رموز تعبيرية وتنسيق واضح.
- يتوقع حركة السعر لـ 6 شمعات قادمة (افتراضيًا).
- يحدد أفضل 5 عملات للتداول.
- يقدم تقييمًا لنسبة دقة قراءة المؤشرات (تقريبي).
"""
import os
import time
import requests
import datetime
import math
import traceback
from typing import Dict, List, Optional, Tuple, Any

# ==============================================================================
# ////////////////           إعدادات المستخدم الأساسية           ////////////////
# ==============================================================================

# --- إعدادات Telegram ---
BOT_TOKEN: str = "7842436262:AAHPMbzYO-Kzrl5k33Gdw4yc_6i2qcAnnik"  # <<-- ضع التوكن الخاص بك هنا
CHAT_ID: str = "5316104346"  # <<-- ضع معرف الدردشة الخاص بك هنا

# --- إعدادات BingX API ---
API_KEY: str = "BA0ipq2dFUZ7JiDTy41fkMdX2QX2ni7FrXQQ93ZCvo1Hsj7gSYrkrkQk4hvTGkeeu0ekrrBAZHDmJFNTZPIyA"  # <<-- ضع مفتاح API الخاص بك هنا

# --- إعدادات التحليل والبيانات ---
# تأكد من أن الرموز صحيحة للعقود الآجلة USDT-M على BingX
SYMBOLS: List[str] = [
    "BTC-USDT", "ETH-USDT", "BNB-USDT", "SOL-USDT",
    "XRP-USDT", "TAO-USDT",  # <<-- تم التغيير إلى TAO (تحقق من وجوده)
    "DOGE-USDT", "ADA-USDT"
]
KLINE_INTERVAL: str = "5m"  # **تم التعديل**: الفاصل الزمني 5 دقائق
KLINE_LIMIT: int = 400  # **تم التعديل**: زيادة الحد لتغطية فترة أطول (400 شمعة)
UPDATE_INTERVAL_SECONDS: int = 150  # **تم التعديل**: فاصل زمني للتحليل 2:30 دقيقة (150 ثانية)
DELAY_BETWEEN_SYMBOLS: float = 0.1 # تقليل التأخير لتحليل أسرع
DELAY_AFTER_TELEGRAM_SEND: float = 0.5 # تقليل التأخير

NUM_PREDICTED_CANDLES: int = 6  # عدد الشمعات المتوقعة
LEVERAGE: int = 20  # الرافعة المالية (20%)
NUM_TOP_SYMBOLS: int = 5 # عدد العملات الأعلى تقييما

# --- إعدادات التقرير ---
REPORT_FORMAT_STYLE: str = "simple"  # **تم التعديل**: تنسيق تقرير مبسط
# الخيارات: "simple", "detailed", "compact", "signal_only"

# --- إعدادات المؤشرات الفنية ---
# **تم التعديل**: الاحتفاظ بالفترات الافتراضية أو إضافة فترات جديدة
RSI_PERIOD: int = 14
RSI_OVERBOUGHT: int = 70
RSI_OVERSOLD: int = 30
MACD_FAST: int = 12
MACD_SLOW: int = 26
MACD_SIGNAL: int = 9
BBANDS_PERIOD: int = 20
BBANDS_STDDEV: float = 2.0
STOCH_K: int = 14
STOCH_D: int = 3
STOCH_SMOOTH_K: int = 3
SMA_PERIODS: List[int] = [5, 10, 20] # فترات المتوسطات المتحركة
ICHIMOKU_TENKAN: int = 9
ICHIMOKU_KIJUN: int = 26
ICHIMOKU_SENKOU_B: int = 52
FIBONACCI_PERIOD: int = 100
ADX_PERIOD: int = 14
ATR_PERIOD: int = 14
VOLUME_SMA_PERIOD: int = 20
CCI_PERIOD: int = 20
PSAR_STEP: float = 0.02
PSAR_MAX_STEP: float = 0.2
VWAP_PERIOD: int = 20  # فترة شائعة لـ VWAP على الأطر القصيرة
MOM_PERIOD: int = 10 # فترة مؤشر العزم
ROC_PERIOD: int = 12 # فترة معدل التغير
OBV_PERIOD: int = 20 # فترة OBV
CMF_PERIOD: int = 21 # فترة CMF
SUPER_TREND_PERIOD: int = 10
SUPER_TREND_MULTIPLIER: float = 3


# --- إعدادات درجة التأكيد ---
CONFIRMATION_INDICATORS = ['RSI_Trend', 'MACD_Hist', 'ADX_Trend', 'SMA_Align', 'CCI_Trend', 'PSAR_Trend', 'Stoch_Trend','OBV_Trend','Trend_SuperTrend'] # تم إضافة PSAR

# --- نقطة نهاية API ---
BINGX_API_BASE_URL: str = "https://open-api.bingx.com"
KLINE_ENDPOINT: str = "/openApi/swap/v3/quote/klines"

# ==============================================================================
# ////////////////          وظائف مساعدة وتنسيق الأرقام         ////////////////
# ==============================================================================
def safe_format_number(val: Optional[float], precision: int = 4, use_comma: bool = True) -> str:
    """ينسق الأرقام بأمان مع الفواصل والدقة المطلوبة."""
    if val is None:
        return "--"
    try:
        abs_val = abs(val)
        if abs_val == 0:
            current_precision = 2
        elif abs_val < 0.0001:
            current_precision = 8
        elif abs_val < 0.01:
            current_precision = 6
        elif abs_val < 1:
            current_precision = 5
        elif abs_val < 1000 and precision >= 4:
            current_precision = 4
        elif abs_val < 100000:
            current_precision = 2
        else:
            current_precision = 0
        format_string = f"{{:,.{current_precision}f}}" if use_comma else f"{{:.{current_precision}f}}"
        return format_string.format(val)
    except (ValueError, TypeError):
        return "--"

# ==============================================================================
# ////////////////          وظائف جلب البيانات من BingX          ////////////////
# ==============================================================================
def fetch_historical_klines(symbol: str, interval: str = KLINE_INTERVAL,
                            limit: int = KLINE_LIMIT) -> Optional[
    Tuple[List[float], List[float], List[float], List[float], List[float]]]:
    url = f"{BINGX_API_BASE_URL}{KLINE_ENDPOINT}"
    params = {"symbol": symbol, "interval": interval, "limit": limit}
    headers = {"X-BX-APIKEY": API_KEY}
    print(f"  [*] Fetching {limit} klines ({interval}) for {symbol} from {KLINE_ENDPOINT}...")
    try:
        response = requests.get(url, params=params, headers=headers, timeout=30)
        print(f"      HTTP Status Code: {response.status_code}")
        if response.status_code != 200:
            print(f"[-] API Error ({response.status_code}) for {symbol}. URL: {url}")
            try:
                print(f"[-] Response Body: {response.text}")
            except Exception:
                print("[-] Response Body: (Could not decode)")
            return None
        response.raise_for_status()
        try:
            data = response.json()
        except requests.exceptions.JSONDecodeError as json_err:
            print(f"[-] Failed JSON decode for {symbol}. Error: {json_err}")
            print(f"[-] Response Text: {response.text[:500]}...")
            return None
        if data and isinstance(data, dict) and data.get('code') == 0:
            if 'data' in data and isinstance(data['data'], list):
                klines_list_of_dicts = data['data']
                if not klines_list_of_dicts:
                    print(f" [!] API OK but kline list empty for {symbol}.");
                    return None
                o, h, l, c, v = [], [], [], [], []
                malformed = 0
                for i, k_dict in enumerate(klines_list_of_dicts):
                    if isinstance(k_dict, dict) and all(
                            k in k_dict for k in ['open', 'high', 'low', 'close', 'volume']):
                        try:
                            o.append(float(k_dict['open']));
                            h.append(float(k_dict['high']))
                            l.append(float(k_dict['low']));
                            c.append(float(k_dict['close']))
                            v.append(float(k_dict['volume']))
                        except (ValueError, TypeError) as e:
                            malformed += 1;
                            print(f" [!] Err converting kline #{i}: {k_dict}, E:{e}")
                    else:
                        malformed += 1;
                        print(f" [!] Bad kline format #{i}: {k_dict}")
                if malformed > 0:
                    print(f"  [!] Found {malformed} malformed klines for {symbol}.")
                    if len(c) < len(klines_list_of_dicts) * 0.8:
                        print(" [!] High malformed rate. Aborting.");
                        return None
                    else:
                        print("  [!] Continuing despite malformed klines.")
                if c:
                    print(f"  [+] OK: Fetched & Parsed {len(c)} OHLCV points for {symbol}.")
                    if len(c) < limit * 0.9:
                        print(f"  [!] Warn: Received {len(c)}/{limit} klines.")
                    return o[::-1], h[::-1], l[::-1], c[::-1], v[::-1]  # Reverse all
                else:
                    print(f"  [!] No valid close prices parsed for {symbol}.");
                    return None
            else:
                print(f"[-] API OK (code 0) but 'data' key missing/not list for {symbol}. R: {data}");
                return None
        else:
            code = data.get('code', 'N/A') if isinstance(data, dict) else 'N/A'
            msg = data.get('msg', 'No msg') if isinstance(data, dict) else f"Invalid: {str(data)[:200]}"
            print(f"[-] API Error for {symbol}. Code: {code}, Msg: {msg}");
            return None
    except requests.exceptions.Timeout:
        print(f"[-] Timeout fetching klines for {symbol}.");
        return None
    except requests.exceptions.RequestException as e:
        print(f"[-] Network Error fetching klines for {symbol}: {e}");
        return None
    except Exception as e:
        print(f"[-] Unexpected Error fetch/process for {symbol}: {e}");
        print(traceback.format_exc());
        return None

# ==============================================================================
# ////////////////        فئة حساب المؤشرات وتحليلها يدويًا        ////////////////
# ==============================================================================
class ManualTradingAnalyzer:
    """ يحسب المؤشرات الفنية وتفسرها وتولد إشارات بسيطة بدون مكتبات خارجية. """

    # --- دوال مساعدة (SMA, EMA, StdDev, TR, Wilder Smooth) ---
    @staticmethod
    def _calculate_sma(data: List[float], period: int) -> Optional[float]:
        if not data or len(data) < period:
            return None
        try:
            return sum(data[-period:]) / period
        except Exception as e:
            print(f"Err SMA ({period}): {e}");
            return None

    @staticmethod
    def _calculate_ema(data: List[float], period: int) -> Optional[float]:
        if not data or len(data) < period:
            return None
        try:
            sma = sum(data[:period]) / period
            ema_values = [sma]
            multiplier = 2 / (period + 1)
            for price in data[period:]:
                ema = (price * multiplier) + (ema_values[-1] * (1 - multiplier))
                ema_values.append(ema)
            return ema_values[-1] if ema_values else None
        except Exception as e:
            print(f"Err EMA ({period}): {e}");
            return None

    @staticmethod
    def _calculate_std_dev(data: List[float], period: int) -> Optional[float]:
        if not data or len(data) < period:
            return None
        subset = data[-period:]
        if len(subset) < 2:
            return 0.0
        try:
            mean = sum(subset) / len(subset)
            variance = sum([(price - mean) ** 2 for price in subset]) / len(subset)
            return math.sqrt(variance) if variance >= 0 else 0.0
        except Exception as e:
            print(f"Err StdDev ({period}): {e}");
            return None

    @staticmethod
    def _calculate_tr(high_prices: List[float], low_prices: List[float], close_prices: List[float]) -> List[float]:
        tr_list = []
        if not (high_prices and low_prices and close_prices and len(close_prices) > 1):
            return tr_list
        try:
            tr_list.append(high_prices[0] - low_prices[0])
            for i in range(1, len(close_prices)):
                tr = max(high_prices[i] - low_prices[i],
                         abs(high_prices[i] - close_prices[i - 1]),
                         abs(low_prices[i] - close_prices[i - 1]))
                tr_list.append(tr)
        except Exception as e:
            print(f"Err TR: {e}")
        return tr_list

    @staticmethod
    def _calculate_wilder_smooth(data: List[float], period: int) -> Optional[List[float]]:
        if not data or len(data) < period:
            return None
        smoothed_values = []
        try:
            sma = sum(data[:period]) / period
            smoothed_values.append(sma)
            for i in range(period, len(data)):
                new_val = (smoothed_values[-1] * (period - 1) + data[i]) / period
                smoothed_values.append(new_val)
        except Exception as e:
            print(f"Err Wilder ({period}): {e}");
            return None
        return smoothed_values
    
    @staticmethod
    def _calculate_obv(close_prices: List[float], volumes: List[float]) -> Optional[List[float]]:
        """ Calculates On Balance Volume (OBV) """
        if not (close_prices and volumes and len(close_prices) == len(volumes) and len(close_prices) > 1):
            return None
        try:
            obv_values = [0]  # Start with a zero OBV
            for i in range(1, len(close_prices)):
                if close_prices[i] > close_prices[i-1]:  # Uptick
                    obv_values.append(obv_values[-1] + volumes[i])
                elif close_prices[i] < close_prices[i-1]:  # Downtick
                    obv_values.append(obv_values[-1] - volumes[i])
                else:
                    obv_values.append(obv_values[-1])  # No change
            return obv_values
        except Exception as e:
            print(f"Err OBV: {e}")
            return None

    # --- دوال حساب المؤشرات الرئيسية ---
    def calculate_rsi(self, prices: List[float], period: int = RSI_PERIOD) -> Optional[float]:
        if not prices or len(prices) < period + 1:
            return None
        try:
            changes = [prices[i] - prices[i - 1] for i in range(1, len(prices))]
            relevant_changes = changes[-period:]
            gains = [c for c in relevant_changes if c >= 0]
            losses = [abs(c) for c in relevant_changes if c < 0]
            avg_gain = sum(gains) / period if gains else 0.0
            avg_loss = sum(losses) / period if losses else 0.0
            if avg_loss == 0:
                return 100.0
            rs = avg_gain / avg_loss
            rsi = 100.0 - (100.0 / (1.0 + rs))
            return round(rsi, 2)
        except Exception as e:
            print(f"Err RSI ({period}): {e}");
            return None

    def calculate_macd(self, prices: List[float], fast: int = MACD_FAST, slow: int = MACD_SLOW,
                      signal: int = MACD_SIGNAL) -> Tuple[Optional[float], Optional[float], Optional[float]]:
        if not prices or len(prices) < slow + signal - 1:
            return None, None, None
        try:
            ema_fast = self._calculate_ema(prices, fast)
            ema_slow = self._calculate_ema(prices, slow)
            if ema_fast is None or ema_slow is None:
                return None, None, None
            macd_line = ema_fast - ema_slow
            macd_values_for_signal = []
            temp_ema_fast_hist, temp_ema_slow_hist = [], []
            if len(prices) >= fast:
                sma_f = sum(prices[:fast]) / fast;
                temp_ema_fast_hist.append(sma_f)
                mult_f = 2 / (fast + 1);
                [temp_ema_fast_hist.append((prices[i] * mult_f) + (temp_ema_fast_hist[-1] * (1 - mult_f))) for i in
                 range(fast, len(prices))]
            if len(prices) >= slow:
                sma_s = sum(prices[:slow]) / slow;
                temp_ema_slow_hist.append(sma_s)
                mult_s = 2 / (slow + 1);
                [temp_ema_slow_hist.append((prices[i] * mult_s) + (temp_ema_slow_hist[-1] * (1 - mult_s))) for i in
                 range(slow, len(prices))]
            if len(temp_ema_fast_hist) > (slow - fast) and len(temp_ema_slow_hist) > 0:
                common_len = min(len(temp_ema_slow_hist), len(temp_ema_fast_hist) - (slow - fast))
                start_idx_fast = slow - fast
                for i in range(common_len):
                    macd_values_for_signal.append(
                        temp_ema_fast_hist[i + start_idx_fast] - temp_ema_slow_hist[i])
            signal_line = self._calculate_ema(macd_values_for_signal, signal)
            macd_hist = (macd_line - signal_line) if signal_line is not None else None
            return round(macd_line, 5), round(signal_line, 5) if signal_line else None, round(macd_hist,
                                                                                              5) if macd_hist else None
        except Exception as e:
            print(f"Err MACD: {e}");
            return None, None, None

    def calculate_bollinger_bands(self, prices: List[float], period: int = BBANDS_PERIOD,
                                  std_dev_multiplier: float = BBANDS_STDDEV) -> Tuple[
        Optional[float], Optional[float], Optional[float]]:
        if not prices or len(prices) < period:
            return None, None, None
        try:
            middle_band = self._calculate_sma(prices, period)
            if middle_band is None:
                return None, None, None
            std_dev = self._calculate_std_dev(prices, period)  # Use full list for std dev
            if std_dev is None:
                return None, middle_band, None
            upper_band = middle_band + (std_dev * std_dev_multiplier)
            lower_band = middle_band - (std_dev * std_dev_multiplier)
            return round(lower_band, 4), round(middle_band, 4), round(upper_band, 4)
        except Exception as e:
            print(f"Err BBands ({period}): {e}");
            return None, None, None

    def calculate_stochastic(self, high_prices: List[float], low_prices: List[float], close_prices: List[float],
                             k_period: int = STOCH_K, d_period: int = STOCH_D,
                             smooth_k_period: int = STOCH_SMOOTH_K) -> Tuple[Optional[float], Optional[float]]:
        required_length = k_period + smooth_k_period - 1 + d_period - 1
        if not (high_prices and low_prices and close_prices and len(close_prices) >= required_length):
            return None, None
        try:
            k_values_raw = []
            for i in range(k_period - 1, len(close_prices)):
                subset_h = high_prices[i - k_period + 1: i + 1];
                subset_l = low_prices[i - k_period + 1: i + 1]
                highest_high = max(subset_h);
                lowest_low = min(subset_l);
                current_close = close_prices[i]
                if highest_high == lowest_low:
                    k = 100.0 if current_close >= lowest_low else 0.0
                else:
                    k = 100.0 * (current_close - lowest_low) / (highest_high - lowest_low)
                k_values_raw.append(k)
            if not k_values_raw:
                return None, None
            k_values_smoothed = []
            if len(k_values_raw) >= smooth_k_period:
                for i in range(smooth_k_period - 1, len(k_values_raw)):
                    k_values_smoothed.append(
                        sum(k_values_raw[i - smooth_k_period + 1:i + 1]) / smooth_k_period)
            else:
                return None, None
            if not k_values_smoothed:
                return None, None
            d_values = []
            if len(k_values_smoothed) >= d_period:
                d_values = [sum(k_values_smoothed[i - d_period + 1:i + 1]) / d_period for i in
                            range(d_period - 1, len(k_values_smoothed))]
            else:
                return None, None
            if not d_values:
                return None, None
            return round(k_values_smoothed[-1], 2), round(d_values[-1], 2)
        except Exception as e:
            print(f"Err Stoch: {e}");
            return None, None

    def calculate_ichimoku(self, high_prices: List[float], low_prices: List[float], close_prices: List[float],
                           tenkan_p: int = ICHIMOKU_TENKAN, kijun_p: int = ICHIMOKU_KIJUN,
                           senkou_b_p: int = ICHIMOKU_SENKOU_B) -> Tuple[Optional[float], ...]:
        required_len = senkou_b_p
        if not (high_prices and low_prices and close_prices and len(close_prices) >= required_len):
            return (None,) * 5
        try:
            tenkan = (max(high_prices[-tenkan_p:]) + min(low_prices[-tenkan_p:])) / 2
            kijun = (max(high_prices[-kijun_p:]) + min(low_prices[-kijun_p:])) / 2
            senkou_a = (tenkan + kijun) / 2
            senkou_b = (max(high_prices[-senkou_b_p:]) + min(low_prices[-senkou_b_p:])) / 2
            chikou_span_value = close_prices[-1]
            return (round(tenkan, 4), round(kijun, 4), round(senkou_a, 4), round(senkou_b, 4),
                    round(chikou_span_value, 4))
        except Exception as e:
            print(f"Err Ichimoku: {e}");
            return (None,) * 5

    def calculate_fibonacci(self, high_prices: List[float], low_prices: List[float],
                            period: int = FIBONACCI_PERIOD) -> Dict[str, Optional[float]]:
        levels = {'fib_236': None, 'fib_382': None, 'fib_500': None, 'fib_618': None}
        if not (high_prices and low_prices and len(high_prices) >= period and len(low_prices) >= period):
            return levels
        try:
            recent_highs = high_prices[-period:];
            recent_lows = low_prices[-period:]
            high = max(recent_highs);
            low = min(recent_lows);
            diff = high - low
            if diff > 1e-9:
                levels['fib_236'] = round(high - diff * 0.236, 5)
                levels['fib_382'] = round(high - diff * 0.382, 5)
                levels['fib_500'] = round(high - diff * 0.5, 5)
                levels['fib_618'] = round(high - diff * 0.618, 5)
            return levels
        except Exception as e:
            print(f"Err Fibo: {e}");
            return levels

    def calculate_atr(self, high_prices: List[float], low_prices: List[float], close_prices: List[float],
                      period: int = ATR_PERIOD) -> Optional[float]:
        if not (high_prices and low_prices and close_prices and len(close_prices) >= period):
            return None
        try:
            tr_list = self._calculate_tr(high_prices, low_prices, close_prices)
            if len(tr_list) < period:
                return None
            # Use Wilder smoothing for ATR
            atr_list = self._calculate_wilder_smooth(tr_list[1:],
                                                     period)  # TR list is one element shorter than price lists
            return round(atr_list[-1], 5) if atr_list else None
        except Exception as e:
            print(f"Err ATR ({period}): {e}");
            return None

    def calculate_adx(self, high_prices: List[float], low_prices: List[float], close_prices: List[float],
                      period: int = ADX_PERIOD) -> Tuple[Optional[float], Optional[float], Optional[float]]:
        required_len = period * 2  # Needs more data due to double smoothing
        if not (high_prices and low_prices and close_prices and len(close_prices) >= required_len):
            return None, None, None
        try:
            plus_dm, minus_dm = [], []
            for i in range(1, len(high_prices)):
                move_up = high_prices[i] - high_prices[i - 1];
                move_down = low_prices[i - 1] - low_prices[i]
                plus_dm.append(move_up if move_up > move_down and move_up > 0 else 0.0)
                minus_dm.append(move_down if move_down > move_up and move_down > 0 else 0.0)
            tr = self._calculate_tr(high_prices, low_prices, close_prices)[1:]
            smooth_plus_dm = self._calculate_wilder_smooth(plus_dm, period)
            smooth_minus_dm = self._calculate_wilder_smooth(minus_dm, period)
            smooth_tr = self._calculate_wilder_smooth(tr, period)
            if not (smooth_plus_dm and smooth_minus_dm and smooth_tr):
                return None, None, None
            plus_di, minus_di, dx = [], [], []
            min_smooth_len = min(len(smooth_tr), len(smooth_plus_dm), len(smooth_minus_dm))
            for i in range(min_smooth_len):
                if smooth_tr[i] != 0:
                    pdi = 100 * (smooth_plus_dm[i] / smooth_tr[i])
                    mdi = 100 * (smooth_minus_dm[i] / smooth_tr[i])
                    plus_di.append(pdi);
                    minus_di.append(mdi)
                    di_sum = pdi + mdi
                    dx.append(100 * (abs(pdi - mdi) / di_sum) if di_sum != 0 else 0.0)
                else:
                    plus_di.append(0.0);
                    minus_di.append(0.0);
                    dx.append(0.0)
            if not dx:
                return None, None, None
            adx_list = self._calculate_wilder_smooth(dx, period)  # Smooth DX to get ADX
            if not adx_list:
                return None, None, None
            return round(adx_list[-1], 2), round(plus_di[-1], 2), round(minus_di[-1], 2)
        except Exception as e:
            print(f"Err ADX ({period}): {e}");
            return None, None, None

    def calculate_cci(self, high_prices: List[float], low_prices: List[float], close_prices: List[float],
                      period: int = CCI_PERIOD) -> Optional[float]:
        if not (high_prices and low_prices and close_prices and len(close_prices) >= period):
            return None
        try:
            tp_list = [(high_prices[i] + low_prices[i] + close_prices[i]) / 3 for i in range(len(close_prices))]
            tp_sma = self._calculate_sma(tp_list, period)
            if tp_sma is None:
                return None
            mean_deviation = sum([abs(tp - tp_sma) for tp in tp_list[-period:]]) / period
            if mean_deviation == 0:
                return 0.0
            last_tp = tp_list[-1]
            cci = (last_tp - tp_sma) / (0.015 * mean_deviation)
            return round(cci, 2)
        except Exception as e:
            print(f"Err CCI ({period}): {e}");
            return None

    def calculate_psar(self, high_prices: List[float], low_prices: List[float],
                       step: float = PSAR_STEP, max_step: float = PSAR_MAX_STEP) -> Optional[float]:
        if not high_prices or not low_prices or len(high_prices) < 2:
            return None
        try:
            psar = low_prices[0];
            ep = high_prices[0];
            af = step;
            is_rising = True
            psar_values = [psar]
            for i in range(1, len(high_prices)):
                prev_psar = psar_values[-1]
                if is_rising:
                    psar = prev_psar + af * (ep - prev_psar)
                    psar = min(psar, low_prices[i - 1], low_prices[i - 2] if i > 1 else low_prices[i - 1])
                    if low_prices[i] < psar:
                        is_rising = False;
                        psar = ep;
                        ep = low_prices[i];
                        af = step
                    else:
                        if high_prices[i] > ep:
                            ep = high_prices[i];
                            af = min(af + step, max_step)
                else:  # Falling
                    psar = prev_psar + af * (ep - prev_psar)
                    psar = max(psar, high_prices[i - 1], high_prices[i - 2] if i > 1 else high_prices[i - 1])
                    if high_prices[i] > psar:
                        is_rising = True;
                        psar = ep;
                        ep = high_prices[i];
                        af = step
                    else:
                        if low_prices[i] < ep:
                            ep = low_prices[i];
                            af = min(af + step, max_step)
                psar_values.append(psar)
            return round(psar_values[-1], 5) if psar_values else None
        except Exception as e:
            print(f"Err PSAR: {e}");
            return None

    def calculate_vwap(self, high_prices: List[float], low_prices: List[float], close_prices: List[float],
                       volumes: List[float],
                       period: int = VWAP_PERIOD) -> Optional[float]:
        """ Calculates Volume Weighted Average Price (VWAP) for a given period """
        required_len = period
        if not (high_prices and low_prices and close_prices and volumes and len(close_prices) >= required_len):
            return None
        try:
            relevant_high = high_prices[-period:]
            relevant_low = low_prices[-period:]
            relevant_close = close_prices[-period:]
            relevant_volume = volumes[-period:]

            typical_price_volume_sum = sum(
                ((relevant_high[i] + relevant_low[i] + relevant_close[i]) / 3) * relevant_volume[i] for i in
                range(period))
            total_volume_sum = sum(relevant_volume)

            if total_volume_sum == 0:
                return None  # Avoid division by zero

            vwap = typical_price_volume_sum / total_volume_sum
            return round(vwap, 4)
        except Exception as e:
            print(f"Err VWAP ({period}): {e}");
            return None

    def calculate_momentum(self, prices: List[float], period: int = MOM_PERIOD) -> Optional[float]:
        """Calculates Momentum (MOM)"""
        if not prices or len(prices) < period:
            return None
        try:
            return round(prices[-1] - prices[-period], 2)
        except Exception as e:
            print(f"Err MOM: {e}")
            return None

    def calculate_roc(self, prices: List[float], period: int = ROC_PERIOD) -> Optional[float]:
        """Calculates Rate of Change (ROC)"""
        if not prices or len(prices) < period or prices[period-1] == 0:
            return None
        try:
            return round(((prices[-1] - prices[period-1]) / prices[period-1]) * 100, 2)
        except Exception as e:
            print(f"Err ROC: {e}")
            return None
    
    def calculate_cmf(self, high_prices: List[float], low_prices: List[float], close_prices: List[float], volumes: List[float], period: int = CMF_PERIOD) -> Optional[float]:
        """Calculates Chaikin Money Flow (CMF)"""
        if not (high_prices and low_prices and close_prices and volumes and 
                len(high_prices) == len(low_prices) == len(close_prices) == len(volumes) and
                len(close_prices) >= period):
            return None
        try:
            money_flow_volume = []
            for i in range(len(close_prices)):
                money_flow_multiplier = ((close_prices[i] - low_prices[i]) - (high_prices[i] - close_prices[i])) / (high_prices[i] - low_prices[i]) if (high_prices[i] - low_prices[i]) != 0 else 0
                money_flow_volume.append(money_flow_multiplier * volumes[i])

            cmf_value = sum(money_flow_volume[-period:]) / sum(volumes[-period:]) if sum(volumes[-period:]) != 0 else 0
            return round(cmf_value, 4)
        except Exception as e:
            print(f"Err CMF: {e}")
            return None

    def calculate_supertrend(self, high_prices: List[float], low_prices: List[float], close_prices: List[float], period: int = SUPER_TREND_PERIOD, multiplier: float = SUPER_TREND_MULTIPLIER) -> Optional[float]:
        """Calculates Supertrend indicator"""
        if not (high_prices and low_prices and close_prices and len(close_prices) >= period):
            return None
        try:
            atr_values = self.calculate_atr(high_prices, low_prices, close_prices, period)
            if atr_values is None:
                return None
            
            basic_upperband = [(high_prices[i] + low_prices[i]) / 2 + multiplier * atr_values for i in range(len(close_prices))]
            basic_lowerband = [(high_prices[i] + low_prices[i]) / 2 - multiplier * atr_values for i in range(len(close_prices))]
            
            supertrend = [0.0] * len(close_prices)
            supertrend[0] = basic_upperband[0]  # Initialize supertrend

            for i in range(1, len(close_prices)):
                if supertrend[i-1] == basic_upperband[i-1]:
                    if close_prices[i] <= basic_lowerband[i]:
                        supertrend[i] = basic_lowerband[i]
                    else:
                        supertrend[i] = basic_upperband[i]
                elif supertrend[i-1] == basic_lowerband[i-1]:
                    if close_prices[i] >= basic_upperband[i]:
                        supertrend[i] = basic_upperband[i]
                    else:
                        supertrend[i] = basic_lowerband[i]

            return supertrend[-1] if supertrend else None
        except Exception as e:
            print(f"Err Supertrend: {e}")
            return None



    # --- تجميع حساب المؤشرات ---
    def calculate_all_indicators(self, ohlcv_data: Optional[Tuple[List[float], ...]],
                                 symbol: str) -> Dict[str, Optional[float]]:
        indicators = {}
        if not ohlcv_data or len(ohlcv_data) != 5:
            print(f"  [!] Invalid OHLCV data for {symbol} in calculate_all_indicators.")
            return indicators

        open_p, high_p, low_p, close_p, volumes = ohlcv_data
        n = len(close_p)
        print(f"  [*] Calculating indicators for {symbol} using {n} data points...")

        required_data_points = max(SMA_PERIODS[-1], ICHIMOKU_SENKOU_B, FIBONACCI_PERIOD,
                                   MACD_SLOW + MACD_SIGNAL - 1, BBANDS_PERIOD, RSI_PERIOD + 1,
                                   STOCH_K + STOCH_D + STOCH_SMOOTH_K - 2,
                                   ADX_PERIOD * 2, ATR_PERIOD, VOLUME_SMA_PERIOD, CCI_PERIOD, VWAP_PERIOD,
                                   MOM_PERIOD, ROC_PERIOD, CMF_PERIOD, SUPER_TREND_PERIOD)
        
        if n < required_data_points:
            print(f"  [!] Insufficient data for full calc {symbol}: {n} < {required_data_points}")
            # Return partially filled dict or empty dict if critical indicators can't be calculated
            if n < 100:  # Arbitrary threshold, adjust as needed
                print(f"  [!] Very low data points ({n}). Skipping indicator calculations.")
                return indicators

        # Calculate indicators with error handling for each
        indicators['rsi'] = self.calculate_rsi(close_p, RSI_PERIOD)
        indicators['macd'], indicators['macdsignal'], indicators['macdhist'] = self.calculate_macd(close_p, MACD_FAST,
                                                                                                   MACD_SLOW, MACD_SIGNAL)
        indicators['bbl'], indicators['bbm'], indicators['bbu'] = self.calculate_bollinger_bands(close_p, BBANDS_PERIOD,
                                                                                                 BBANDS_STDDEV)
        indicators['stoch_k'], indicators['stoch_d'] = self.calculate_stochastic(high_p, low_p, close_p, STOCH_K,
                                                                                STOCH_D, STOCH_SMOOTH_K)
        for period in SMA_PERIODS:
            indicators[f'sma_{period}'] = self._calculate_sma(close_p, period)
        (indicators['tenkan'], indicators['kijun'], indicators['senkou_a'],
         indicators['senkou_b'], indicators['chikou']) = self.calculate_ichimoku(high_p, low_p, close_p)
        fib_levels = self.calculate_fibonacci(high_p, low_p, FIBONACCI_PERIOD)
        indicators.update(fib_levels)
        indicators['atr'] = self.calculate_atr(high_p, low_p, close_p, ATR_PERIOD)
        indicators['adx'], indicators['plus_di'], indicators['minus_di'] = self.calculate_adx(high_p, low_p, close_p,
                                                                                              ADX_PERIOD)
        indicators['cci'] = self.calculate_cci(high_p, low_p, close_p, CCI_PERIOD)
        indicators['psar'] = self.calculate_psar(high_p, low_p, PSAR_STEP, PSAR_MAX_STEP)
        indicators['vwap'] = self.calculate_vwap(high_p, low_p, close_p, volumes, VWAP_PERIOD)
        indicators['volume_sma'] = self._calculate_sma(volumes, VOLUME_SMA_PERIOD)
        indicators['last_volume'] = volumes[-1] if volumes else None
        indicators['momentum'] = self.calculate_momentum(close_p, MOM_PERIOD)
        indicators['roc'] = self.calculate_roc(close_p, ROC_PERIOD)

        obv_values = self._calculate_obv(close_p, volumes)
        indicators['obv'] = obv_values[-1] if obv_values else None
        indicators['cmf'] = self.calculate_cmf(high_p, low_p, close_p, volumes, CMF_PERIOD)
        indicators['supertrend'] = self.calculate_supertrend(high_p, low_p, close_p)
        print(f"  [+] Indicator calculation completed for {symbol}.")
        return indicators
    
    def interpret_indicators(self, current_price: Optional[float], indicators: Dict[str, Optional[float]],
                             prices_history: List[float], volumes_history: List[float]) -> Dict[str, str]:
        """يفسر قيم المؤشرات إلى نص مفهوم."""
        interpretations = {}

        # --- RSI ---
        rsi = indicators.get('rsi')
        interpretations['rsi_status'] = "بيع" if rsi is not None and rsi > RSI_OVERBOUGHT else ("شراء" if rsi is not None and rsi < RSI_OVERSOLD else "محايد")
        interpretations['rsi_trend'] = 'sell' if rsi is not None and rsi > RSI_OVERBOUGHT else ('buy' if rsi is not None and rsi < RSI_OVERSOLD else None)

        # --- MACD ---
        macd, signal, hist = indicators.get('macd'), indicators.get('macdsignal'), indicators.get('macdhist')
        interpretations['macd_status'] = "إيجابي" if hist is not None and hist > 0 else ("سلبي" if hist is not None and hist < 0 else "محايد")

        # --- Stochastic ---
        stoch_k, stoch_d = indicators.get('stoch_k'), indicators.get('stoch_d')
        interpretations['stoch_status'] = "تشبع بيعي" if stoch_k is not None and stoch_d is not None and stoch_k < 20 and stoch_d < 20 else ("تشبع شرائي" if stoch_k is not None and stoch_d is not None and stoch_k > 80 and stoch_d > 80 else "محايد")
        interpretations['Stoch_Trend'] = "buy" if stoch_k is not None and stoch_d is not None and stoch_k > stoch_d else("sell" if stoch_k is not None and stoch_d is not None and stoch_k < stoch_d else None)

        # --- Bollinger Bands ---
        bbl, bbm, bbu = indicators.get('bbl'), indicators.get('bbm'), indicators.get('bbu')
        interpretations['bbands_status'] = "فوق العلوي" if current_price is not None and bbu is not None and current_price > bbu else ("تحت السفلي" if current_price is not None and bbl is not None and current_price < bbl else "داخل النطاقات")

        # --- ADX (Average Directional Index) ---
        adx = indicators.get('adx')
        plus_di = indicators.get('plus_di')
        minus_di = indicators.get('minus_di')
        interpretations['adx_status'] = "قوي" if adx is not None and adx > 25 else "ضعيف"
        interpretations['adx_trend_direction'] = 'buy' if plus_di is not None and minus_di is not None and plus_di > minus_di else ('sell' if plus_di is not None and minus_di is not None and plus_di < minus_di else None)

        # --- Volume ---
        last_volume, volume_sma = indicators.get('last_volume'), indicators.get('volume_sma')
        interpretations['volume_status'] = "مرتفع" if last_volume is not None and volume_sma is not None and last_volume > volume_sma * 1.2 else "منخفض"

        # --- CCI (Commodity Channel Index) ---
        cci = indicators.get('cci')
        interpretations['cci_status'] = "تشبع شراء" if cci is not None and cci > 100 else ("تشبع بيع" if cci is not None and cci < -100 else "محايد")
        interpretations['cci_trend'] = 'buy' if cci is not None and cci < -100 else ('sell' if cci is not None and cci > 100 else None)

        # --- PSAR (Parabolic SAR) ---
        psar = indicators.get('psar')
        interpretations['psar_status'] = "صاعد" if current_price is not None and psar is not None and current_price > psar else "هابط"
        interpretations['psar_trend'] = 'buy' if current_price is not None and psar is not None and current_price > psar else ('sell' if current_price is not None and psar is not None and current_price < psar else None)

        # --- SMAs ---
        sma_short = indicators.get('sma_5') # استخدام فترة SMA أقصر
        sma_long = indicators.get('sma_20')
        interpretations['sma_status'] = "صاعد" if sma_short is not None and sma_long is not None and sma_short > sma_long else "هابط"
        interpretations['sma_align'] = 'buy' if sma_short is not None and sma_long is not None and sma_short > sma_long else ('sell' if sma_short is not None and sma_long is not None and sma_short < sma_long else None)

        # --- Volume Based Indicators ---
        obv = indicators.get('obv')
        interpretations['obv_status'] = 'صاعد' if obv is not None and len(prices_history) > 1 and obv > self._calculate_obv(prices_history[:-1],volumes_history[:-1])[-1] else ("هابط" if obv is not None and len(prices_history) > 1 and obv < self._calculate_obv(prices_history[:-1],volumes_history[:-1])[-1] else "محايد")
        interpretations['OBV_Trend'] = 'buy' if obv is not None and len(prices_history) > 1 and obv > self._calculate_obv(prices_history[:-1],volumes_history[:-1])[-1] else ('sell' if obv is not None and len(prices_history) > 1 and obv < self._calculate_obv(prices_history[:-1],volumes_history[:-1])[-1] else None)

        # -- SuperTrend
        supertrend = indicators.get('supertrend')
        interpretations['supertrend_status'] = "صاعد" if current_price is not None and supertrend is not None and current_price > supertrend else "هابط"
        interpretations['Trend_SuperTrend'] = 'buy' if current_price is not None and supertrend is not None and current_price > supertrend else ('sell' if current_price is not None and supertrend is not None and current_price < supertrend else None)

        return interpretations

    def calculate_confirmation_score(self, indicators: Dict[str, Optional[float]], interpretations: Dict[str, str]) -> Tuple[str, float, float]:
        """Calculates confirmation score based on indicator alignments."""
        buy_conf_score = 0.0
        sell_conf_score = 0.0
        max_score = float(len(CONFIRMATION_INDICATORS))
        if max_score == 0:
            return "⚪ N/A", 0.0, 0.0

        # Check RSI Trend (using pre-calculated interpretation)
        if 'RSI_Trend' in CONFIRMATION_INDICATORS:
            if interpretations.get('rsi_trend') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('rsi_trend') == 'sell':
                sell_conf_score += 1.0

        # Check MACD Histogram
        if 'MACD_Hist' in CONFIRMATION_INDICATORS and indicators.get('macdhist') is not None:
            if indicators['macdhist'] > 0:
                buy_conf_score += 1.0
            elif indicators['macdhist'] < 0:
                sell_conf_score += 1.0

        # Check ADX Trend Direction (if trend is strong enough)
        if 'ADX_Trend' in CONFIRMATION_INDICATORS:
            if interpretations.get('adx_trend_direction') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('adx_trend_direction') == 'sell':
                sell_conf_score += 1.0

        # Check SMA Alignment
        if 'SMA_Align' in CONFIRMATION_INDICATORS:
            if interpretations.get('sma_align') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('sma_align') == 'sell':
                sell_conf_score += 1.0

        # Check CCI Trend
        if 'CCI_Trend' in CONFIRMATION_INDICATORS:
            if interpretations.get('cci_trend') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('cci_trend') == 'sell':
                sell_conf_score += 1.0

        # Check PSAR Trend
        if 'PSAR_Trend' in CONFIRMATION_INDICATORS:
            if interpretations.get('psar_trend') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('psar_trend') == 'sell':
                sell_conf_score += 1.0
        
        # Check Stochastic Trend
        if 'Stoch_Trend' in CONFIRMATION_INDICATORS:
            if interpretations.get('Stoch_Trend') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('Stoch_Trend') == 'sell':
                sell_conf_score += 1.0
        
        # Check OBV Trend
        if 'OBV_Trend' in CONFIRMATION_INDICATORS:
            if interpretations.get('OBV_Trend') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('OBV_Trend') == 'sell':
                sell_conf_score += 1.0
        # Check Supertrend
        if 'Trend_SuperTrend' in CONFIRMATION_INDICATORS:
            if interpretations.get('Trend_SuperTrend') == 'buy':
                buy_conf_score += 1.0
            elif interpretations.get('Trend_SuperTrend') == 'sell':
                sell_conf_score += 1.0
        # Determine confirmation text
        confirmation_text = "⚪ محايد" # افتراضي
        if max_score > 0:
            if buy_conf_score > sell_conf_score:
                ratio = buy_conf_score / max_score
                if ratio >= 0.8:
                    confirmation_text = "🟢 قوي جداً"
                elif ratio >= 0.6:
                    confirmation_text = "🟢 متوسط"
                else:
                    confirmation_text = "🟢 ضعيف"
            elif sell_conf_score > buy_conf_score:
                ratio = sell_conf_score / max_score
                if ratio >= 0.8:
                    confirmation_text = "🔴 قوي جداً"
                elif ratio >= 0.6:
                    confirmation_text = "🔴 متوسط"
                else:
                    confirmation_text = "🔴 ضعيف"
            elif buy_conf_score > 0:  # Equal non-zero scores
                confirmation_text = "🟡 متعارض"

        return f"{confirmation_text} [{buy_conf_score:.0f} شراء/{sell_conf_score:.0f} بيع]", buy_conf_score, sell_conf_score

    def generate_signals(self, current_price: Optional[float], indicators: Dict[str, Optional[float]], interpretations: Dict[str, str]) -> str:
        """Generates a simplified Buy/Sell signal based on interpretations."""
        if current_price is None:
            return "⚪ لا توجد بيانات سعر"

        confirmation_text, buy_score, sell_score = self.calculate_confirmation_score(indicators, interpretations)

        # --- تبسيط إشارات التداول ---
        signal = "⚪ محايد" # افتراضي
        if buy_score > sell_score:
            signal = "🟢 شراء (Long)"
        elif sell_score > buy_score:
            signal = "🔴 بيع (Short)"

        return signal # إرجاع الإشارة المبسطة فقط

    def predict_future_candles(self, close_prices: List[float], num_candles: int = NUM_PREDICTED_CANDLES) -> List[float]:
        """A very simplistic prediction - extend the linear trend."""
        if not close_prices or len(close_prices) < 2:
            return [None] * num_candles  # Not enough data
        
        # Linear extrapolation
        change = close_prices[-1] - close_prices[-2]
        predictions = [round(close_prices[-1] + change * (i + 1), 2) for i in range(num_candles)] # توقع الشموع المستقبلية
        return predictions # إرجاع قائمة التوقعات


    def evaluate_indicator_accuracy(self, interpretations: Dict[str, str]) -> float:
        """A very basic evaluation of indicator agreement (more sophisticated logic needed)."""
        trending_indicators = 0
        conflicting_indicators = 0

        if interpretations.get('rsi_trend') == 'buy': trending_indicators += 1
        elif interpretations.get('rsi_trend') == 'sell': trending_indicators += 1
        else: conflicting_indicators += 1 # محايد

        if interpretations.get('macd_status') == 'إيجابي': trending_indicators += 1
        elif interpretations.get('macd_status') == 'سلبي': trending_indicators += 1
        else: conflicting_indicators += 1 # محايد

        if interpretations.get('stoch_status') == 'تشبع بيعي': trending_indicators += 1 # عكسي
        elif interpretations.get('stoch_status') == 'تشبع شرائي': trending_indicators += 1
        else: conflicting_indicators += 1

        # Simple Momentum Check
        if interpretations.get('obv_status') == 'صاعد': trending_indicators += 1
        elif interpretations.get('obv_status') == 'هابط': trending_indicators += 1
        else: conflicting_indicators += 1 # محايد

        total_indicators = trending_indicators + conflicting_indicators
        accuracy = (trending_indicators / total_indicators) * 100 if total_indicators > 0 else 50  # 50% إذا لم تتوفر بيانات
        return round(accuracy, 1) # إرجاع النسبة المئوية

# ==============================================================================
# ////////////////        وظائف مساعدة (Telegram والتنسيق)        ////////////////
# ==============================================================================
# (send_telegram_alert as before)
def send_telegram_alert(message: str) -> None:
    if not BOT_TOKEN or not CHAT_ID:
        print("[!] Telegram not configured.");
        return
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    max_length = 4096
    message_parts = [message[i:i + max_length] for i in range(0, len(message), max_length)]
    for i, part in enumerate(message_parts):
        payload = {"chat_id": CHAT_ID, "text": part, "parse_mode": "HTML", "disable_web_page_preview": True}
        try:
            response = requests.post(url, json=payload, timeout=20)
            response.raise_for_status()
            print(f"[+] TG alert part {i + 1}/{len(message_parts)} sent.")
            if len(message_parts) > 1 and i < len(message_parts) - 1:
                time.sleep(1)
        except requests.exceptions.HTTPError as http_err:
            error_details = f"HTTP Error {http_err.response.status_code}"
            try:
                error_details += f": {http_err.response.json()}"
            except requests.exceptions.JSONDecodeError:
                error_details += f" - Body: {http_err.response.text[:200]}"
            print(f"[-] Failed TG send part {i + 1}. Error: {error_details}")
        except requests.exceptions.RequestException as e:
            print(f"[-] Failed TG send part {i + 1}. Network Error: {e}")
        except Exception as e:
            print(f"[-] Unexpected error sending TG part {i + 1}: {e}");
            print(traceback.format_exc())

def format_report_simple(symbol: str, current_price: Optional[float], indicators: Dict[str, Optional[float]], interpretations: Dict[str, str], signal: str, predicted_candles: List[Optional[float]], accuracy: float, leverage: int = LEVERAGE) -> str:
    """Formats a simple, trading-focused report."""
    price_str = f"<code>{safe_format_number(current_price, 4)}</code>" if current_price is not None else "N/A"
    current_time_str = datetime.datetime.now().strftime('%H:%M:%S') # الوقت الفعلي
    report = f"""
🪙 {symbol} ({KLINE_INTERVAL}) | ⏰{current_time_str} | 💲 {price_str}
⭐ الإشارة: <b>{signal}</b> (رافعة: {leverage}%)
🔮 توقع 6 شمعات قادمة: {', '.join([safe_format_number(p, 2) if p is not None else 'N/A' for p in predicted_candles])}
📊 دقة المؤشرات: {accuracy}%
"""
    return report.strip()

# --- وظائف تنسيق التقارير (تم التبسيط) ---
# (تذكر حذف أو تعديل وظائف التنسيق الأخرى إذا كنت تستخدم هذا الأسلوب فقط)

# ==============================================================================
# ////////////////                الحلقة الرئيسية للتشغيل                ////////////////
# ==============================================================================
def run_analysis_loop():
    analyzer = ManualTradingAnalyzer()
    print("===================================================")
    print("[*] Starting BingX Bot (Manual TA V5.1 - Professional)...")
    print(f"[*] Symbols: {', '.join(SYMBOLS)}")
    print(f"[*] Interval: {KLINE_INTERVAL}, Limit: {KLINE_LIMIT}")
    print(f"[*] Update Interval: {UPDATE_INTERVAL_SECONDS}s (Every 2.5 minutes)")
    print(f"[*] Report Style: simple")
    print(f"[*] Confirmation Indicators: {', '.join(CONFIRMATION_INDICATORS)}")
    print("===================================================")

    if not all([BOT_TOKEN and len(BOT_TOKEN) > 20, CHAT_ID, API_KEY]):
        print("\n!!! CRITICAL ERROR: Bot credentials missing or invalid.")
        return

    # **تم التعديل**: استخدام تنسيق التقرير المبسط
    report_formatter = format_report_simple
    print(f"[*] Using report formatter: {report_formatter.__name__}")

    while True:
        start_time = time.time()
        try:
            current_time_str = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            print(f"\n[*] {current_time_str} - Starting analysis cycle...")
            symbol_data = [] # جمع بيانات العملات لترتيبها
            successful_analyses = 0

            for symbol in SYMBOLS:
                print(f"--- Processing {symbol} ---")
                ohlcv_data = fetch_historical_klines(symbol, KLINE_INTERVAL, KLINE_LIMIT)

                required_data_points = max(SMA_PERIODS[-1], ICHIMOKU_SENKOU_B, FIBONACCI_PERIOD,
                                           MACD_SLOW + MACD_SIGNAL - 1, BBANDS_PERIOD, RSI_PERIOD + 1,
                                           STOCH_K + STOCH_D + STOCH_SMOOTH_K - 2,
                                           ADX_PERIOD * 2, ATR_PERIOD, VOLUME_SMA_PERIOD, CCI_PERIOD, VWAP_PERIOD,
                                           MOM_PERIOD, ROC_PERIOD, CMF_PERIOD)

                current_price = None
                report_to_send = None
                historical_closes = []
                historical_volumes = []

                if ohlcv_data:
                    _, _, _, historical_closes, historical_volumes = ohlcv_data
                    if historical_closes:
                        current_price = historical_closes[-1]
                    else:
                        print(f"  [!] Empty close prices list for {symbol}.")
                else:
                    print(f"  [!] OHLCV data fetch failed for {symbol}.")

                # --- Execute analysis and reporting ---
                if ohlcv_data and len(historical_closes) >= required_data_points:
                    print(f"  [*] Current Price (Last Close): {current_price}")
                    try:
                        indicators = analyzer.calculate_all_indicators(ohlcv_data, symbol)
                        interpretations = analyzer.interpret_indicators(current_price, indicators, historical_closes,historical_volumes)
                        signal = analyzer.generate_signals(current_price, indicators, interpretations) # Get simplified signal
                        predicted_candles = analyzer.predict_future_candles(historical_closes)
                        accuracy = analyzer.evaluate_indicator_accuracy(interpretations) # نسبة تقريبية لدقة المؤشرات

                        # -- تخزين البيانات لتحديد أفضل العملات لاحقًا ---
                        symbol_data.append({
                            'symbol': symbol,
                            'signal': signal,
                            'accuracy': accuracy,
                            'current_price': current_price,
                            'indicators': indicators, # نحتاج إلى المؤشرات لتحديد الأفضل
                            'interpretations':interpretations,
                            'predicted_candles': predicted_candles
                            })
                        successful_analyses += 1 # الإبلاغ عن التحليل الناجح في هذه المرحلة

                    except Exception as analysis_err:
                        print(f"  [!] Error during analysis phase for {symbol}: {analysis_err}")
                        print(traceback.format_exc())
                        report_to_send = f"🪙 <b>{symbol} ({KLINE_INTERVAL})</b>\n   ⚠️ Error during analysis.\n"

                elif historical_closes:  # Insufficient data
                    print(
                        f"  [!] Insufficient data for {symbol} ({len(historical_closes)} < {required_data_points}).")
                    price_str = f"💲 <code>{safe_format_number(current_price, 4)}</code>" if current_price is not None else "💲 N/A"
                    report_to_send = f"🪙 <b>{symbol} ({KLINE_INTERVAL})</b> | {price_str}\n   ⏳ Insufficient data ({len(historical_closes)}/{required_data_points}).\n"
                else:  # Failed fetch or no close prices
                    print(f"  [!] Failed/No data for {symbol}.")
                    report_to_send = f"🪙 <b>{symbol} ({KLINE_INTERVAL})</b>\n   ⚠️ Failed to fetch or parse data.\n"

                if report_to_send:
                    send_telegram_alert(report_to_send) # إرسال رسالة الخطأ على الفور
                    time.sleep(DELAY_AFTER_TELEGRAM_SEND) # احترام التأخير

                print(f"--- Finished processing {symbol} ---")
                time.sleep(DELAY_BETWEEN_SYMBOLS)


            # --- تحديد أفضل 5 عملات ---
            sorted_symbols = sorted(symbol_data, key=lambda x: x['accuracy'], reverse=True)[:NUM_TOP_SYMBOLS]

            # --- إرسال تقرير Telegram واحد لأفضل العملات ---
            top_symbols_report = "\n🔥 **أفضل 5 عملات حسب دقة المؤشرات:** 🔥\n"
            for s in sorted_symbols:
                report_to_send = report_formatter(s['symbol'], s['current_price'], s['indicators'], s['interpretations'], s['signal'], s['predicted_candles'], s['accuracy'])
                top_symbols_report += report_to_send + "\n"

            if successful_analyses > 0: # فقط إرسال إذا كان هناك تحليل ناجح واحد على الأقل
                send_telegram_alert(top_symbols_report) # إرسال تقرير مجمع
                time.sleep(DELAY_AFTER_TELEGRAM_SEND) # احترام التأخير
            else:
                send_telegram_alert("\n[!] لم يتم إجراء تحليل ناجح لأي عملة في هذه الدورة.\n") # إبلاغ المستخدم


            cycle_duration = time.time() - start_time
            print(
                f"\n[*] Analysis cycle completed in {cycle_duration:.2f} seconds. Successful analyses: {successful_analyses}/{len(SYMBOLS)}")
            sleep_time = max(0, UPDATE_INTERVAL_SECONDS - cycle_duration)
            print(f"[*] Sleeping for {sleep_time:.2f} seconds...")
            time.sleep(sleep_time)

        except KeyboardInterrupt:
            print("\n[!] Bot stopped manually by user (KeyboardInterrupt).")
            break
        except Exception as e:
            print(f"\n[!!!] CRITICAL ERROR in main loop: {e}")
            print(traceback.format_exc())
            error_message = f"🚨 Bot encountered a critical error: {e}\n```\n{traceback.format_exc()}\n```\nAttempting to restart after 60 seconds."
            send_telegram_alert(error_message[:4000])
            print("[!] Attempting to recover after 60 seconds...")
            time.sleep(60)

# ==============================================================================
# ////////////////                 نقطة الدخول الرئيسية                 ////////////////
# ==============================================================================
if __name__ == "__main__":
    try:
        import requests
    except ImportError:
        print("=" * 50);
        print("!!! CRITICAL ERROR: 'requests' library is not installed.");
        exit()

    run_analysis_loop()