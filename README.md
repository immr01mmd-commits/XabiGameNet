from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler, CallbackContext
import random, json, os

# ذخیره داده‌ها
if os.path.exists("players.json"):
    with open("players.json", "r", encoding="utf-8") as f:
        players = json.load(f)
else:
    players = {}

# ======== ابزارها و اسکین‌ها ========
shop_items = {
    "🛠️ ابزار مسی": {"price":50, "power":2},
    "🛡️ ابزار فولادی": {"price":150, "power":4},
    "💎 ابزار الماسی": {"price":500, "power":8},
    "👷 اسکین معدنچی": {"price":100, "skin":"👷"},
    "🧑‍🚀 اسکین فضانورد": {"price":300, "skin":"🧑‍🚀"},
}

# ======== منوی اصلی ========
def start(update: Update, context: CallbackContext):
    user = update.effective_user
    uid = str(user.id)
    if uid not in players:
        players[uid] = {"coins":0, "level":1, "power":1, "skin":"👷"}
        save_players()
    update.message.reply_text(
        f"سلام {user.first_name}! ⛏️ خوش اومدی به ماینر حرفه‌ای!",
        reply_markup=main_menu()
    )

def main_menu():
    keyboard = [
        [InlineKeyboardButton("⛏️ ماین کن", callback_data="mine"),
         InlineKeyboardButton("🛒 شاپ", callback_data="shop")],
        [InlineKeyboardButton("📈 ارتقا", callback_data="upgrade"),
         InlineKeyboardButton("🏆 لیدربورد", callback_data="leaderboard")],
        [InlineKeyboardButton("📊 وضعیت", callback_data="status")]
    ]
    return InlineKeyboardMarkup(keyboard)

# ======== ماینینگ ========
def mine(update: Update, context: CallbackContext):
    query = update.callback_query
    uid = str(query.from_user.id)
    query.answer()
    power = players[uid]["power"]
    coins_earned = random.randint(5,15) * power
    players[uid]["coins"] += coins_earned
    # ارتقا سطح خودکار
    players[uid]["level"] = 1 + players[uid]["coins"] // 500
    save_players()
    query.edit_message_text(
        f"{players[uid]['skin']} شما ماین کردید و {coins_earned} سکه گرفتید!\nکل سکه‌ها: {players[uid]['coins']}\nسطح: {players[uid]['level']}",
        reply_markup=main_menu()
    )

# ======== شاپ ========
def shop(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    keyboard = [[InlineKeyboardButton(f"{item} - {info['price']} 💰", callback_data=f"buy_{item}")] for item, info in shop_items.items()]
    query.edit_message_text("🛒 شاپ حرفه‌ای ماینر:", reply_markup=InlineKeyboardMarkup(keyboard))

def buy(update: Update, context: CallbackContext):
    query = update.callback_query
    uid = str(query.from_user.id)
    query.answer()
    item_name = query.data.split("_",1)[1]
    item = shop_items[item_name]

    if players[uid]["coins"] >= item["price"]:
        players[uid]["coins"] -= item["price"]
        if "power" in item:
            players[uid]["power"] = item["power"]
        if "skin" in item:
            players[uid]["skin"] = item["skin"]
        save_players()
        text = f"🎉 خرید موفق! {item_name} به شما اضافه شد."
    else:
        text = "❌ سکه کافی نیست."
    query.edit_message_text(text=text, reply_markup=main_menu())

# ======== ارتقا ========
def upgrade(update: Update, context: CallbackContext):
    query = update.callback_query
    uid = str(query.from_user.id)
    query.answer()
    cost = players[uid]["level"] * 100
    if players[uid]["coins"] >= cost:
        players[uid]["coins"] -= cost
        players[uid]["power"] += 1
        save_players()
        text = f"💪 ارتقا موفق! قدرت شما اکنون: {players[uid]['power']}"
    else:
        text = f"❌ سکه کافی نیست! برای ارتقا نیاز به {cost} سکه دارید."
    query.edit_message_text(text=text, reply_markup=main_menu())

# ======== لیدربورد ========
def leaderboard(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    top_players = sorted(players.items(), key=lambda x: x[1]["coins"], reverse=True)[:10]
    text = "🏆 لیدربورد جهانی:\n"
    for i, (uid, data) in enumerate(top_players,1):
        text += f"{i}. {data['skin']} {uid} - {data['coins']} 💰\n"
    query.edit_message_text(text=text, reply_markup=main_menu())

# ======== وضعیت ========
def status(update: Update, context: CallbackContext):
    query = update.callback_query
    uid = str(query.from_user.id)
    data = players[uid]
    query.answer()
    query.edit_message_text(
        f"👤 شما: {data['skin']}\nسطح: {data['level']}\nقدرت: {data['power']}\nسکه‌ها: {data['coins']}",
        reply_markup=main_menu()
    )

# ======== ذخیره داده‌ها ========
def save_players():
    with open("players.json", "w", encoding="utf-8") as f:
        json.dump(players, f, ensure_ascii=False, indent=4)

# ======== هندلر دکمه‌ها ========
def button(update: Update, context: CallbackContext):
    query = update.callback_query
    data = query.data
    if data == "mine":
        mine(update, context)
    elif data == "shop":
        shop(update, context)
    elif data.startswith("buy_"):
        buy(update, context)
    elif data == "upgrade":
        upgrade(update, context)
    elif data == "leaderboard":
        leaderboard(update, context)
    elif data == "status":
        status(update, context)

# ======== main ========
def main():
    updater = Updater("YOUR_BOT_TOKEN", use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CallbackQueryHandler(button))

    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    main()
