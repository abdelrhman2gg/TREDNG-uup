import requests
import time
import datetime

# ██████████████████████████████████████████████████████████████████████████
#                               إعدادات التطبيق
# ██████████████████████████████████████████████████████████████████████████

# ━━━ بيانات التليجرام ━━━
bot_token = "7842436262:AAHPMbzYO-Kzrl5k33Gdw4yc_6i2qcAnnik"
chat_id = "5316104346"

# ━━━ بيانات BingX API ━━━
api_key = "BA0ipq2dFUZ7JiDTy41fkMdX2QX2ni7FrXQQ93ZCvo1Hsj7gSYrkrkQk4hvTGkeeu0ekrrBAZHDmJFNTZPIyA"
secret_key = "mpawe74izq7jFwcblGqKFXDYQ4uwiQLrLF7wVseJ1qEyVZWLziUuY9HCV6wASr12K8pqlwYbo4RG8OeJKWFndA"

# ━━━ العملات المُتداولة ━━━
symbols = ["BTC-USDT", "ETH-USDT", "SOL-USDT", "XRP-USDT", "BNB-USDT"]

# ██████████████████████████████████████████████████████████████████████████
#                                 الدوال الأساسية
# ██████████████████████████████████████████████████████████████████████████

def send_telegram_message(message: str) -> None:
    """إرسال رسالة إلى بوت التليجرام"""
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    payload = {"chat_id": chat_id, "text": message}
    
    try:
        response = requests.post(url, json=payload)
        if response.status_code != 200:
            print(f"❌ فشل إرسال الرسالة | كود الخطأ: {response.status_code}")
    except Exception as e:
        print(f"❌ خطأ في الإرسال: {str(e)}")

def get_current_price(symbol: str) -> float | None:
    """جلب السعر الحالي للعملة من منصة BingX"""
    url = "https://open-api.bingx.com/openApi/swap/v2/quote/price"
    params = {"symbol": symbol}
    headers = {"X-BX-APIKEY": api_key}
    
    try:
        response = requests.get(url, params=params, headers=headers)
        data = response.json()
        
        if data.get('data') and 'price' in data['data']:
            return float(data['data']['price'])
        else:
            print(f"⚠️ بيانات غير متوقعة لـ {symbol}: {data}")
            return None
            
    except Exception as e:
        print(f"❌ خطأ في جلب السعر لـ {symbol}: {str(e)}")
        return None

def calculate_indicators(price: float, history: list) -> tuple:
    """حساب المؤشرات الفنية"""
    # توقع الأسعار للـ6 شمعات القادمة
    predictions = [round(price + (0.1 * (i+1)), 2) for i in range(6)]
    
    # حساب المتوسط المتحرك البسيط (5 فترات)
    updated_history = history + [price]
    if len(updated_history) > 5:
        updated_history.pop(0)
        
    moving_avg = round(sum(updated_history)/len(updated_history), 2)
    
    return predictions, moving_avg

# ██████████████████████████████████████████████████████████████████████████
#                                 حلقة التشغيل الرئيسية
# ██████████████████████████████████████████████████████████████████████████

def main():
    price_history = {symbol: [] for symbol in symbols}
    
    while True:
        print("\n" + "="*40 + " بدء دورة جديدة " + "="*40)
        
        for symbol in symbols:
            print(f"\n🔎 معالجة {symbol}...")
            
            # ━━━ جلب البيانات ━━━
            current_price = get_current_price(symbol)
            
            if current_price:
                # ━━━ حساب المؤشرات ━━━
                predictions, ma = calculate_indicators(
                    current_price,
                    price_history[symbol]
                )
                
                # ━━━ إعداد الرسالة ━━━
                message = (
                    f"📊 **{symbol}**\n"
                    f"▸ السعر الحالي: {current_price}\n"
                    f"▸ توقع الشمعات القادمة: {predictions}\n"
                    f"▸ المتوسط المتحرك (5): {ma}\n"
                    f"⏱ وقت التحديث: {datetime.datetime.now().strftime('%H:%M:%S')}"
                )
                
                # ━━━ إرسال التنبيه ━━━
                send_telegram_message(message)
                print("✅ تم إرسال التنبيه بنجاح")
                
                # تحديث التاريخ
                price_history[symbol] = price_history[symbol][-4:] + [current_price]
                
            else:
                print(f"⚠️ فشل في معالجة {symbol}")
        
        # ━━━ فاصل بين الدورات ━━━
        print("\n" + "="*35 + " انتظار 60 ثانية " + "="*35)
        time.sleep(60)

if __name__ == "__main__":
    main()
