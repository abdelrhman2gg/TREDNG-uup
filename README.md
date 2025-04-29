import os
import requests
import time
import datetime

# قراءة بيانات البيئة
api_key = os.getenv("BA0ipq2dFUZ7JiDTy41fkMdX2QX2 ni7FrXQQ93ZCvo1Hsj7gSYrkrkQk4 hvTGkeeu0ekrrBAZHDmJFNTZPIyA")
secret_key = os.getenv("mpawe74izq7jFwcblGqKFXDYQ4uwiQLrLF7w VseJ1qEyVZWLziUuY9HCV6wASr12K8pqlwY bo4RG8OeJKWFndA")
bot_token = os.getenv("7842436262:AAHPMbzYO-Kzrl5k33Gdw4yc_6i2qcAnnik")
chat_id = os.getenv("5316104346")

# العملات اللي هنشتغل عليها
symbols = ["BTC-USDT", "ETH-USDT", "SOL-USDT", "XRP-USDT", "BNB-USDT"]

# دالة إرسال رسالة للتليجرام
def send_telegram_message(message):
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    payload = {
        "chat_id": chat_id,
        "text": message
    }
    try:
        response = requests.post(url, data=payload)
        if response.status_code != 200:
            print("Error sending message:", response.text)
    except Exception as e:
        print("Exception sending message:", e)

# دالة قراءة السعر من BingX
def get_price(symbol):
    url = "https://open-api.bingx.com/openApi/spot/v1/ticker/price"
    params = {
        "symbol": symbol
    }
    headers = {
        "X-BX-APIKEY": api_key
    }
    try:
        response = requests.get(url, params=params, headers=headers)
        data = response.json()
        return float(data['data']['price'])
    except Exception as e:
        print("Exception getting price:", e)
        return None

# دالة توقع ٦ شمعات قادمة
def predict_next_6_candles(current_price):
    prediction = []
    for i in range(6):
        delta = 0.1 * (i + 1)  # زياده وهمية بسيطة للشرح
        prediction.append(round(current_price + delta, 2))
    return prediction

# الحلقة الأساسية
def main():
    while True:
        for symbol in symbols:
            price = get_price(symbol)
            if price:
                prediction = predict_next_6_candles(price)
                message = f"**{symbol}**\n\n" \
                          f"Current Price: {price}\n\n" \
                          f"Next 6 Candles Prediction: {prediction}\n\n" \
                          f"Updated at: {datetime.datetime.now().strftime('%H:%M:%S')}"
                send_telegram_message(message)
                print(f"Sent update for {symbol}")
            time.sleep(2)  # 2 ثواني بين كل عملة عشان نرتاح شوية

        time.sleep(120)  # كل دقيقتين يعمل التحديث

if __name__ == "__main__":
    main()
