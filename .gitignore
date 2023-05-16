from telegram import ParseMode
import json
import logging
import sqlite3
import requests
import time
from telegram.ext import ConversationHandler
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackQueryHandler, CallbackContext
from telegram.error import NetworkError
from telegram import Bot


API_KEY_TTS = "1cb58f61b6786aea0cf917afb8a923d4"
API_KEY_TELEGRAM = "6298052506:AAGlQAlN0T5RunBhkJhpGKEg-EYOHvw5wA0"
TTS_API_URL = "https://texttospeech.ru/api/v2/synthesize"
FREE_VOICE_CODES = ["ru-RU001"]
PAID_VOICE_CODES = ["ru-RU020", "ru-RU021"]
VOICE_CODES = FREE_VOICE_CODES + PAID_VOICE_CODES
PRICE_PER_1000_CHAR = 30
STARTING_BALANCE = 10
DATABASE_NAME = "user_balances.db"
ADMIN_USER_ID = 1864913930

bot = Bot(token=API_KEY_TELEGRAM)
bot_username = bot.get_me().username

logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)

logger = logging.getLogger(__name__)


def init_db():
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute(
        "CREATE TABLE IF NOT EXISTS user_balance (user_id INTEGER PRIMARY KEY, balance INTEGER)")
    cursor.execute(
        "CREATE TABLE IF NOT EXISTS referrals (referrer_id INTEGER, referred_id INTEGER, status INTEGER, PRIMARY KEY(referrer_id, referred_id))")
    connection.commit()
    connection.close()


def set_user_balance(user_id, balance):
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute("INSERT OR REPLACE INTO user_balance VALUES (?, ?)",
                   (user_id, balance))
    connection.commit()
    connection.close()


def get_user_balance(user_id):
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute("SELECT balance FROM user_balance WHERE user_id = ?",
                   (user_id,))
    result = cursor.fetchone()
    connection.close()
    return result[0] if result else None


def show_start_menu(update, context):
    keyboard = start_menu_keyboard()
    update.callback_query.edit_message_text(
        "Выберите опцию:", reply_markup=keyboard)


def start_menu_keyboard():
    keyboard = [
        [InlineKeyboardButton("Озвучка текста 🗣", callback_data="voice_text")],
        [InlineKeyboardButton("Подсказка 💡",
                              callback_data="how_to_use")],
        [InlineKeyboardButton("Мой аккаунт ⚙️", callback_data="my_account")],
        [InlineKeyboardButton("Получить подарок 🎁", callback_data="get_gift")],
        [InlineKeyboardButton("Служба поддержки 👨🏿‍💻",
                              url="https://t.me/pastmalevich")]
    ]
    return InlineKeyboardMarkup(keyboard)


def generate_referral_link(user_id):
    return f"https://t.me/{bot_username}?start={user_id}"


def get_referral_count(user_id):
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute(
        "SELECT COUNT(*) FROM referrals WHERE referrer_id = ? AND status = 1", (user_id,))
    result = cursor.fetchone()
    connection.close()
    return result[0] if result else 0


def create_referral(referrer_id, referred_id):
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute("INSERT OR IGNORE INTO referrals VALUES (?, ?, 0)",
                   (referrer_id, referred_id))
    connection.commit()
    connection.close()


def update_referral_status(referrer_id, referred_id):
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute("UPDATE referrals SET status = 1 WHERE referrer_id = ? AND referred_id = ?",
                   (referrer_id, referred_id))
    connection.commit()
    connection.close()


def how_to_use(update, context):
    how_to_use_text = "Для того, чтобы задать определенную интонацию голоса при синтезе речи, вы можете использовать знаки препинания. Чтобы поставить ударение, поставьте символ '+' перед гласной, например - з+амок или зам+ок. Для создания паузы в речи используйте две этих чёрточки: '--'."
    update.callback_query.edit_message_text(how_to_use_text)
    context.bot.send_message(chat_id=update.effective_chat.id,
                             text="Нажмите /start, чтобы вернуться в главное меню.")


def show_my_account(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()

    my_account_text = "Мой аккаунт"
    keyboard = InlineKeyboardMarkup([
        [InlineKeyboardButton("Баланс", callback_data="balance")],
        [InlineKeyboardButton("Пополнить баланс",
                              callback_data="replenish_balance")]
    ])

    query.edit_message_text(my_account_text, reply_markup=keyboard)


def start(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    user_name = update.effective_user.first_name

    referrer_id = update.message.text.split(
    )[-1] if len(update.message.text.split()) > 1 else None

    if get_user_balance(user_id) is None:
        set_user_balance(user_id, STARTING_BALANCE)

        if referrer_id and referrer_id.isdigit():
            create_referral(int(referrer_id), user_id)

    update.message.reply_text(
        f"Привет, {user_name}! Я бот для реалистичной озвучки текста.")
    keyboard = start_menu_keyboard()
    update.message.reply_text("Выберите опцию:", reply_markup=keyboard)


def text_to_speech(text, voice_code):
    url = "https://texttospeech.ru/api/v2/synthesize"
    headers = {"Api-Key": API_KEY_TTS}
    data = {"text": text, "code": voice_code, "format": "mp3"}

    response = requests.post(url, headers=headers, data=data)

    if response.status_code == 200:
        result = response.json()
        if result["status"] == "success":
            return {"status": "success", "file": result["file"]}
        else:
            return {"status": "error", "message": result["comment"]}
    else:
        return {"status": "error", "message": f"HTTP Error {response.status_code}"}


def generate_voice_choice_keyboard():
    keyboard = []
    for voice_code in FREE_VOICE_CODES:
        keyboard.append([InlineKeyboardButton(
            f"{voice_code} (Free)", callback_data=voice_code)])
    for voice_code in PAID_VOICE_CODES:
        keyboard.append([InlineKeyboardButton(
            f"{voice_code} (Paid)", callback_data=voice_code)])
    return InlineKeyboardMarkup(keyboard)


def send_message_with_retry(update, context, text, reply_markup=None, max_retries=3):
    retries = 0
    while retries < max_retries:
        try:
            if update.message is not None:
                update.message.reply_text(text=text, reply_markup=reply_markup)
                break
        except NetworkError:
            retries += 1
            time.sleep(1)
            if retries == max_retries:
                logging.error("Failed to send message after maximum retries")
                break


def handle_start_menu(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()

    choice = query.data

    if choice == "voice_text":
        query.edit_message_text(
            "Выберите голос из списка для озвучивания.")
        keyboard = generate_voice_choice_keyboard()
        query.edit_message_text("Выберите голос:", reply_markup=keyboard)

    elif choice == "how_to_use":
        how_to_use(update, context)
    elif choice == "my_account":
        show_my_account(update, context)
    elif choice == "balance":
        balance(update, context)
    elif choice == "replenish":
        pass  # Add the replenish functionality here
    elif query.data == "replenish_balance":
        replenish_balance(update, context)
    elif choice == "get_gift":
        handle_get_gift(update, context)


def handle_get_gift(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()

    user_id = update.effective_user.id
    referral_link = generate_referral_link(user_id)
    referral_count = get_referral_count(user_id)

    gift_text = f"Чтобы получить подарок, пригласите друга, отправив ему вашу персональную реферальную ссылку:\n\n`{referral_link}`\n\nЗа каждого приглашенного вами друга вы получите 10 рублей на ваш баланс после того, как он включит бота и сделает хотя бы одну озвучку текста.\n\nВы пригласили {referral_count}/3 друзей."

    query.edit_message_text(gift_text, parse_mode="Markdown")
    context.bot.send_message(chat_id=update.effective_chat.id,
                             text="Нажмите /start, чтобы вернуться в главное меню.")


def wait_for_text(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()

    voice_code = query.data
    context.user_data['voice_code'] = voice_code

    query.edit_message_text(
        "Пожалуйста, отправьте мне текст, и я преобразую его в речь.")


def calculate_cost(text, voice_code):
    if voice_code in PAID_VOICE_CODES:
        cost = (len(text) / 1000) * PRICE_PER_1000_CHAR
        return round(cost, 2)
    return 0


def add_to_balance(user_id, amount):
    balance = get_user_balance(user_id)
    new_balance = balance + amount
    set_user_balance(user_id, new_balance)


def deduct_from_balance(user_id, amount):
    balance = get_user_balance(user_id)
    if balance >= amount:
        new_balance = balance - amount
        set_user_balance(user_id, new_balance)
        return True
    return False


def text_input(update: Update, context: CallbackContext):
    text = update.message.text
    context.user_data['text'] = text
    # Call the function 'process_text' without returning
    process_text(update, context)


def process_text(update: Update, context: CallbackContext):
    text = context.user_data['text']

    user_id = update.effective_user.id

    # Default to a free voice if not specified
    voice_code = context.user_data.get('voice_code', 'ru-RU001')

    cost = calculate_cost(text, voice_code)

    if get_user_balance(user_id) < cost:
        insufficient_funds_text = f"Стоимость озвучивания: {cost} рублей. У вас недостаточно средств на балансе."
        update.message.reply_text(insufficient_funds_text)
        keyboard = InlineKeyboardMarkup([
            [InlineKeyboardButton("Пополнить баланс",
                                  callback_data="replenish_balance")]  # Change callback_data to "replenish_balance"
        ])
        update.message.reply_text(
            "Нажмите на кнопку ниже, чтобы пополнить баланс:", reply_markup=keyboard)
        return

    deduct_from_balance(user_id, cost)
    update.message.reply_text("Подождите секунду!")

    response = text_to_speech(text, voice_code)

    if response['status'] == "success":
        audio_url = response['file']
        context.bot.send_audio(
            chat_id=update.effective_chat.id, audio=audio_url)
        del context.user_data['text']
        update.message.reply_text(
            f"Озвучка текста стоила {cost} рублей. Нажмите /start, чтобы вернуться в главное меню.")

        # Check if the user was referred and update the referral status
        connection = sqlite3.connect(DATABASE_NAME)
        cursor = connection.cursor()
        cursor.execute("SELECT referrer_id FROM referrals WHERE referred_id = ? AND status = 0",
                       (user_id,))
        result = cursor.fetchone()
        if result:
            referrer_id = result[0]
            referral_count = get_referral_count(referrer_id)
            if referral_count < 3:
                update_referral_status(referrer_id, user_id)
                # Add 10 rubles to the referrer's balance
                add_to_balance(referrer_id, 10)
    else:
        update.message.reply_text(
            "Ошибка при синтезе речи: " + response['comment'])


def balance(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    balance = get_user_balance(user_id)
    update.callback_query.edit_message_text(f"Ваш баланс: {balance} рублей.")
    context.bot.send_message(chat_id=update.effective_chat.id,
                             text="Нажмите /start, чтобы вернуться в главное меню.")


def replenish_balance(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()

    payment_link = "https://my.qiwi.com/Vladyslav-DS4oAi20L4?noCache=true"

    query.edit_message_text(
        text=f"Пополните ваш баланс, перейдя по следующей ссылке: {payment_link}\n\n"
             "<b>После успешного пополнения, отправьте скриншот боту или номер транзакции для подтверждения платежа.</b>\n\n"
             "В случае каких либо проблем или ошибок - обратитесь в ПОДДЕРЖКУ @pastmalevich .",
        parse_mode=ParseMode.HTML
    )
    context.user_data['expecting_payment_proof'] = True  # Set the flag


def handle_payment_proof(update: Update, context: CallbackContext):
    # Check if the flag is not set
    if not context.user_data.get('expecting_payment_proof'):
        return  # If not, don't respond

    if update.message.photo or (update.message.text and update.message.text.isdigit()):
        update.message.reply_text(
            "Спасибо, ваше подтверждение платежа отправлено администратору для проверки. Проверка займёт 2 минуты!"
            "После проверки ваш баланс будет обновлен."
        )
        context.bot.forward_message(chat_id=ADMIN_USER_ID,
                                    from_chat_id=update.effective_chat.id,
                                    message_id=update.message.message_id)
        context.user_data['expecting_payment_proof'] = False  # Reset the flag
    else:
        update.message.reply_text(
            "Пожалуйста, отправьте скриншот или номер транзакции для подтверждения платежа.")


def confirm_payment(update: Update, context: CallbackContext):
    if update.effective_user.id != ADMIN_USER_ID:
        update.message.reply_text(
            "У вас нет прав для выполнения этой команды.")
        return

    args = context.args
    if len(args) != 2:
        update.message.reply_text(
            "Введите команду в формате: /confirm_payment user_id amount")
        return

    user_id = int(args[0])
    amount = int(args[1])

    add_to_balance(user_id, amount)
    update.message.reply_text(
        f"Баланс пользователя {user_id} успешно пополнен на {amount} рублей.")


def main():
    init_db()

    updater = Updater(API_KEY_TELEGRAM, use_context=True)
    dp = updater.dispatcher
    text_input_handler = ConversationHandler(
        entry_points=[CallbackQueryHandler(
            wait_for_text, pattern='^(' + '|'.join(VOICE_CODES) + ')$')],
        states={
            1: [MessageHandler(Filters.text & ~Filters.command, text_input)],
        },
        fallbacks=[CommandHandler('start', start)],
    )
    dp.add_handler(text_input_handler)

    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CallbackQueryHandler(handle_start_menu,
                                        pattern='^(voice_text|how_to_use|my_account|balance|replenish_balance)$'))
    dp.add_handler(MessageHandler(
        Filters.text | Filters.photo, handle_payment_proof))
    dp.add_handler(CommandHandler("balance", balance))
    dp.add_handler(CommandHandler("confirm_payment",
                   confirm_payment, filters=Filters.chat(ADMIN_USER_ID)))
    dp.add_handler(MessageHandler(Filters.text & ~Filters.command &
                   ~Filters.update.edited_message, text_input))
    dp.add_handler(CallbackQueryHandler(handle_get_gift,
                   pattern='^(' + '|'.join(VOICE_CODES) + ')$'))
    dp.add_handler(CallbackQueryHandler(handle_get_gift, pattern='^get_gift$'))

    updater.start_polling()
    updater.idle()


if __name__ == '__main__':
    main()
