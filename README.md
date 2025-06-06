import requests
import threading
import time
import json
from datetime import datetime
from pytz import timezone

BOT_TOKEN = "7693973331:AAFv29gidku7fQDt9nmN_SIkNSRFhj7XgkI"
active_sessions = {}

def send_message(chat_id, message):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    payload = {
        "chat_id": chat_id,
        "text": message["text"],
        "parse_mode": "Markdown",
        "disable_web_page_preview": True
    }
    requests.post(url, data=payload)

def get_countdown(updated_at, interval_sec):
    now = int(time.time() * 1000)
    passed = int((now - updated_at) / 1000)
    remaining = max(interval_sec - passed, 0)
    h = remaining // 3600
    m = (remaining % 3600) // 60
    s = remaining % 60
    return remaining, (f"{h}h {m}m {s}s" if h > 0 else f"{m}m {s}s")

def get_honey_restock_countdown():
    now_ph = datetime.now(timezone('Asia/Manila'))
    m = 59 - now_ph.minute
    s = 60 - now_ph.second
    return m * 60 + s, f"{m:02d}m {s:02d}s"

def fetch_all(chat_id, session_data, single_category=None):
    try:
        gear_seed = requests.get("https://growagardenstock.com/api/stock?type=gear-seeds").json()
        egg = requests.get("https://growagardenstock.com/api/stock?type=egg").json()
        weather = requests.get("https://growagardenstock.com/api/stock/weather").json()
        honey = requests.get("http://65.108.103.151:22377/api/stocks?type=honeyStock").json()
        cosmetics = requests.get("https://growagardenstock.com/api/special-stock?type=cosmetics").json()
        emoji_seeds = requests.get("http://65.108.103.151:22377/api/stocks?type=seedsStock").json().get("seedsStock", [])

        gear_restock_sec, gear_restock = get_countdown(gear_seed.get("updatedAt", 0), 300)
        seed_restock_sec, seed_restock = get_countdown(gear_seed.get("updatedAt", 0), 300)
        egg_restock_sec, egg_restock = get_countdown(egg.get("updatedAt", 0), 600)
        cosmetics_restock_sec, cosmetics_restock = get_countdown(cosmetics.get("updatedAt", 0), 14400)
        honey_restock_sec, honey_restock = get_honey_restock_countdown()

        gear_list = "\n".join(f"- {item}" for item in gear_seed.get("gear", [])) or "No gear."
        seed_list = "\n".join(
            f"- {(next((s['emoji'] for s in emoji_seeds if s['name'].lower() == seed.split(' **')[0].lower()), '') + ' ' if emoji_seeds else '')}{seed}"
            for seed in gear_seed.get("seeds", [])
        ) or "No seeds."
        egg_list = "\n".join(f"- {item}" for item in egg.get("egg", [])) or "No eggs."
        cosmetics_list = "\n".join(f"- {item}" for item in cosmetics.get("cosmetics", [])) or "No cosmetics."
        honey_list = "\n".join(f"- {h['name']}: {h['value']}" for h in honey.get("honeyStock", [])) or "No honey stock."

        weather_icon = weather.get("icon", "🌦️")
        weather_current = weather.get("currentWeather", "Unknown")
        crop_bonus = weather.get("cropBonuses", "None")
        weather_text = f"{weather_icon} {weather_current}"

        def compose_message(category):
            if category == "gear":
                return f"🛠️ *Gear*:\n{gear_list}\n⏳ *Restock in*: {gear_restock}"
            if category == "seeds":
                return f"🌱 *Seeds*:\n{seed_list}\n⏳ *Restock in*: {seed_restock}"
            if category == "egg":
                return f"🥚 *Eggs*:\n{egg_list}\n⏳ *Restock in*: {egg_restock}"
            if category == "cosmetics":
                return f"🎨 *Cosmetics*:\n{cosmetics_list}\n⏳ *Restock in*: {cosmetics_restock}"
            if category == "honey":
                return f"🍯 *Honey Stock*:\n{honey_list}\n⏳ *Restock in*: {honey_restock}"
            if category == "weather":
                return f"🌤️ *Weather*: {weather_text}\n🪴 *Crop Bonus*: {crop_bonus}"
            return ""

        if single_category:
            msg_text = compose_message(single_category)
            send_message(chat_id, {"text": msg_text})
            return

        if not session_data.get("initialized"):
            parts = [
                compose_message("gear"),
                compose_message("seeds"),
                compose_message("egg"),
                compose_message("cosmetics"),
                compose_message("honey"),
                compose_message("weather")
            ]
            message = "🌾 *Grow A Garden Stock Tracker*\n\n" + "\n\n".join(parts)
            send_message(chat_id, {"text": message})
            session_data["initialized"] = True

        updated_categories = []

        if gear_seed.get("seeds") != session_data.get("last_seeds"):
            updated_categories.append(("Seeds Stock Updated", "seeds"))
        if gear_seed.get("gear") != session_data.get("last_gear"):
            updated_categories.append(("Gear Stock Updated", "gear"))
        if egg.get("egg") != session_data.get("last_egg"):
            updated_categories.append(("Egg Stock Updated", "egg"))
        if cosmetics.get("cosmetics") != session_data.get("last_cosmetics"):
            updated_categories.append(("Cosmetics Stock Updated", "cosmetics"))
        if honey.get("honeyStock") != session_data.get("last_honey"):
            updated_categories.append(("Honey Stock Updated", "honey"))
        if weather.get("updatedAt") != session_data.get("last_weather_updated"):
            updated_categories.append(("Weather Updated", "weather"))

        session_data["last_gear"] = gear_seed.get("gear")
        session_data["last_seeds"] = gear_seed.get("seeds")
        session_data["last_egg"] = egg.get("egg")
        session_data["last_cosmetics"] = cosmetics.get("cosmetics")
        session_data["last_honey"] = honey.get("honeyStock")
        session_data["last_weather_updated"] = weather.get("updatedAt")

        for title, cat in updated_categories:
            send_message(chat_id, {"text": f"🔔 *{title}*"})
            msg = compose_message(cat)
            send_message(chat_id, {"text": msg})

    except Exception as e:
        print(f"❌ Gagstock error for {chat_id}: {str(e)}")

def start_tracking(chat_id):
    if chat_id in active_sessions:
        send_message(chat_id, {"text": "📡 You're already tracking Gagstock. Use gagstockoff to stop."})
        return
    send_message(chat_id, {"text": "𝙂𝙧𝙤𝙬 𝘼 𝙂𝙖𝙧𝙙𝙚𝙣 𝙎𝙩𝙤𝙘𝙠 𝘾𝙝𝙚𝙘𝙠𝙚𝙧! 𝙔𝙤𝙪'𝙡𝙡 𝙗𝙚 𝙣𝙤𝙩𝙞𝙛𝙞𝙚𝙙 𝙬𝙝𝙚𝙣 𝙨𝙩𝙤𝙘𝙠𝙨 𝙘𝙝𝙖𝙣𝙜𝙚."})
    session_data = {
        "initialized": False,
        "last_gear": None,
        "last_seeds": None,
        "last_egg": None,
        "last_cosmetics": None,
        "last_honey": None,
        "last_weather_updated": None,
        "stop_flag": False
    }
    def tracking_loop():
        while not session_data["stop_flag"]:
            fetch_all(chat_id, session_data)
            time.sleep(10)
    thread = threading.Thread(target=tracking_loop, daemon=True)
    session_data["thread"] = thread
    active_sessions[chat_id] = session_data
    thread.start()

def stop_tracking(chat_id):
    session = active_sessions.get(chat_id)
    if session:
        session["stop_flag"] = True
        active_sessions.pop(chat_id, None)
        send_message(chat_id, {"text": "🛑 Gagstock tracking stopped."})
    else:
        send_message(chat_id, {"text": "⚠️ You don't have an active gagstock session."})

def gagstock_command(chat_id, command):
    cmd = command.lower().lstrip("/")
    if cmd == "gagstockon":
        start_tracking(chat_id)
    elif cmd == "gagstockoff":
        stop_tracking(chat_id)
    elif cmd in ["gagstockgear", "gagstockseeds", "gagstockeggs", "gagstockcosmetics", "gagstockhoney"]:
        category_map = {
            "gagstockgear": "gear",
            "gagstockseeds": "seeds",
            "gagstockeggs": "egg",
            "gagstockcosmetics": "cosmetics",
            "gagstockhoney": "honey"
        }
        fetch_all(chat_id, {
            "initialized": True,
            "stop_flag": True
        }, single_category=category_map[cmd])
    else:
        send_message(chat_id, {
            "text": (
                "📌 Commands:\n"
                "• gagstockon — Start continuous Gagstock tracking\n"
                "• gagstockoff — Stop Gagstock tracking\n"
                "• gagstockgear — Check current Gear stock\n"
                "• gagstockseeds — Check current Seeds stock\n"
                "• gagstockeggs — Check current Egg stock\n"
                "• gagstockcosmetics — Check current Cosmetics stock\n"
                "• gagstockhoney — Check current Honey stock"
            )
        })

def handle_updates():
    last_update_id = None
    while True:
        try:
            url = f"https://api.telegram.org/bot{BOT_TOKEN}/getUpdates"
            if last_update_id:
                url += f"?offset={last_update_id + 1}"
            response = requests.get(url)
            if response.ok:
                data = response.json()
                for update in data["result"]:
                    last_update_id = update["update_id"]
                    message = update.get("message")
                    if not message:
                        continue
                    chat_id = message["chat"]["id"]
                    text = message.get("text", "").strip()
                    if text.lower().lstrip("/") in [
                        "gagstockon", "gagstockoff", "gagstockgear",
                        "gagstockseeds", "gagstockeggs", "gagstockcosmetics", "gagstockhoney"
                    ]:
                        gagstock_command(chat_id, text.lower())
            time.sleep(2)
        except Exception as e:
            print(f"Update polling error: {e}")
            time.sleep(5)

if __name__ == "__main__":
    print("🤖 Bot is running and polling Telegram...")
    start_tracking(-1002718221143)
    handle_updates()
