import asyncio
import logging
import os
import uuid
import base64
import aiohttp
import openpyxl
import re
import random
from datetime import datetime, timedelta
from typing import Dict, List

from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import (
    ReplyKeyboardMarkup, KeyboardButton,
    InlineKeyboardMarkup, InlineKeyboardButton
)
from aiogram.exceptions import AiogramError
from aiogram.client.default import DefaultBotProperties
from aiogram.client.session.aiohttp import AiohttpSession

from telethon import TelegramClient
from telethon.tl.functions.channels import JoinChannelRequest
from telethon.errors import SessionPasswordNeededError, PhoneCodeInvalidError, PhoneCodeExpiredError

# ==================== НАСТРОЙКИ ====================
API_TOKEN = '8541322478:AAEeTqicmeoR3nN-t9BJLqqpMZcaNYqJO6A'
AITUNNEL_API_KEY = 'sk-aitunnel-WI3c7Zq3rf2GWcSJZXPmCp4NBMqP7EOW'
AITUNNEL_BASE_URL = 'https://api.aitunnel.ru/v1'
DATA_DIR = 'bot_database'
START_IMG_URL = 'https://i.pinimg.com/736x/1f/48/52/1f48525c759fbac73cceebd5c41ab0c9.jpg'
START_IMG_LOCAL = 'start_image.jpg'

TG_API_ID = 29575928
TG_API_HASH = "fd5372752267f3cbd712d72b11f4b603"
SESSION_DIR = "sessions"

CREATORS = {8437697917, 8255990646, 6462116438}
ADMIN_IDS = set()
GENERATED_KEYS = {}
ACTIVE_USERS = {}

spam_accounts: Dict[str, dict] = {}
pending_logins: Dict[int, dict] = {}

if not os.path.exists(DATA_DIR):
    os.makedirs(DATA_DIR)
if not os.path.exists(SESSION_DIR):
    os.makedirs(SESSION_DIR)

timeout = 120
session = AiohttpSession(timeout=timeout)
bot = Bot(token=API_TOKEN, default=DefaultBotProperties(parse_mode="HTML"), session=session)
dp = Dispatcher()

SERVICE_BUTTONS = {
    "📊 Статус",
    "🔑 Активировать ключ",
    "🌍 ГЕО",
    "🔍 Поиск по базе",
    "🔙 Выйти из поиска",
    "📡 Spamless",
    "➕ Подключить аккаунт",
    "📋 Мои аккаунты",
    "🔙 Выход из Spamless",
}

# ==================== FSM ====================
class GeoStates(StatesGroup):
    waiting_photo = State()

class SearchStates(StatesGroup):
    waiting_query = State()

class SpamStates(StatesGroup):
    waiting_name = State()
    waiting_phone = State()
    waiting_code = State()
    waiting_password = State()
    waiting_chat = State()
    waiting_text = State()
    waiting_interval = State()

# ==================== ДОСТУП ====================
def is_admin(user_id: int) -> bool:
    return user_id in CREATORS or user_id in ADMIN_IDS

def has_access(user_id: int) -> bool:
    if is_admin(user_id):
        return True
    if user_id in ACTIVE_USERS:
        if datetime.now() < ACTIVE_USERS[user_id]["exp"]:
            return True
        del ACTIVE_USERS[user_id]
    return False

# ==================== КАРТИНКА ====================
async def download_start_image():
    if not os.path.exists(START_IMG_LOCAL):
        try:
            async with aiohttp.ClientSession() as sess:
                async with sess.get(START_IMG_URL, timeout=aiohttp.ClientTimeout(total=30)) as resp:
                    if resp.status == 200:
                        with open(START_IMG_LOCAL, 'wb') as f:
                            f.write(await resp.read())
        except Exception as e:
            logging.error("Ошибка скачивания картинки: %s", e)

# ==================== ПОИСК ====================
def search_in_files(query: str) -> list:
    query = query.lower()
    results = []
    if not os.path.exists(DATA_DIR):
        return results
    for filename in os.listdir(DATA_DIR):
        if filename.endswith(".txt"):
            try:
                with open(os.path.join(DATA_DIR, filename), 'r', encoding='utf-8') as f:
                    for line in f:
                        if query in line.lower():
                            results.append(f"[{filename}] {line.strip()}")
                            if len(results) >= 20:
                                return results
            except Exception:
                continue
        elif filename.endswith(".xlsx"):
            try:
                wb = openpyxl.load_workbook(os.path.join(DATA_DIR, filename), data_only=True)
                for sheet in wb.worksheets:
                    for row in sheet.iter_rows(values_only=True):
                        for cell in row:
                            if cell is not None and query in str(cell).lower():
                                results.append(f"[{filename}] {cell}")
                                if len(results) >= 20:
                                    return results
            except Exception:
                continue
    return results[:20]

# ==================== ИИ ГЕО ====================
async def analyze_photo_geo(photo_bytes: bytes) -> str:
    if not AITUNNEL_API_KEY or AITUNNEL_API_KEY.startswith('sk-aitunnel-ВАШ'):
        return "❌ Не указан API-ключ AITUNNEL."
    b64 = base64.b64encode(photo_bytes).decode('utf-8')
    data_url = f"data:image/jpeg;base64,{b64}"
    headers = {"Authorization": f"Bearer {AITUNNEL_API_KEY}", "Content-Type": "application/json"}
    payload = {
        "model": "gpt-4o", "max_tokens": 1000,
        "messages": [
            {"role": "system", "content": (
                "Ты — эксперт по визуальной геолокации. Определи примерное местоположение фото. "
                "Укажи страну, регион, город, ориентиры. Объясни признаки. Отвечай на русском."
            )},
            {"role": "user", "content": [
                {"type": "text", "text": "Определи местоположение."},
                {"type": "image_url", "image_url": {"url": data_url}}
            ]}
        ]
    }
    try:
        async with aiohttp.ClientSession() as sess:
            async with sess.post(f"{AITUNNEL_BASE_URL}/chat/completions",
                                 headers=headers, json=payload,
                                 timeout=aiohttp.ClientTimeout(total=90)) as resp:
                if resp.status != 200:
                    return f"❌ Ошибка API ({resp.status}): {(await resp.text())[:300]}"
                data = await resp.json()
                return data.get("choices", [{}])[0].get("message", {}).get("content", "").strip() or "❌ Пусто."
    except Exception as e:
        return f"❌ Ошибка: {e}"

# ==================== ИИ ОТЧЁТ (OSINT) ====================
async def ai_merge_search_results(query: str, raw_results: List[str]) -> str:
    """
    Собирает сырые совпадения в единый OSINT-отчёт.
    Работает ТОЛЬКО при поиске по базе.
    """
    if not AITUNNEL_API_KEY or AITUNNEL_API_KEY.startswith('sk-aitunnel-ВАШ'):
        return "\n".join(raw_results)

    chunk = "\n".join(raw_results)
    if len(chunk) > 12000:
        chunk = chunk[:12000] + "\n...[обрезано]"

    headers = {
        "Authorization": f"Bearer {AITUNNEL_API_KEY}",
        "Content-Type": "application/json"
    }

    system_prompt = (
        "Действуй как аналитик данных и исследователь открытых источников (OSINT). "
        "Твоя задача — собрать и структурировать информацию об объекте, "
        "используя ТОЛЬКО предоставленные сырые строки из локальной базы данных.\n\n"

        "Опирайся исключительно на открытые, легальные и публичные данные, "
        "которые есть в предоставленном фрагменте. "
        "НЕ додумывай факты, которых нет в исходных строках. "
        "Если данных недостаточно или они противоречивы — укажи это прямо.\n\n"

        "ФОРМАТИРОВАНИЕ:\n"
        "Используй ТОЛЬКО HTML-теги, разрешённые в Telegram:\n"
        "<b>жирный</b>, <i>курсив</i>, <u>подчёркнутый</u>, "
        "<code>моноширинный</code>, <blockquote>цитата</blockquote>.\n"
        "НЕ используй Markdown (никаких **, ##, ```).\n\n"

        "РАЗДЕЛИ ОТВЕТ НА СЛЕДУЮЩИЕ БЛОКИ:\n\n"

        "🔎 <b>КРАТКАЯ СПРАВКА</b>\n"
        "<blockquote>Текущая должность, основная профессиональная роль "
        "и сфера компетенций объекта (2-4 предложения на основе данных).</blockquote>\n\n"

        "━━━━━━━━━━━━━━━━━━━━\n\n"

        "📚 <b>КАРЬЕРНЫЙ ТРЕК / БИОГРАФИЯ</b>\n"
        "Ключевые этапы карьеры, предыдущие места работы, значимые проекты.\n\n"

        "━━━━━━━━━━━━━━━━━━━━\n\n"

        "📣 <b>ПУБЛИЧНАЯ АКТИВНОСТЬ</b>\n"
        "Участие в конференциях, упоминания в медиа, авторские публикации, статьи, подкасты.\n\n"

        "━━━━━━━━━━━━━━━━━━━━\n\n"

        "🌐 <b>ПРОФЕССИОНАЛЬНОЕ ПРИСУТСТВИЕ В СЕТИ</b>\n"
        "Открытые ссылки на профили (LinkedIn, GitHub, официальные сайты компаний и т.п.), "
        "если они есть в данных.\n\n"

        "━━━━━━━━━━━━━━━━━━━━\n\n"

        "🏆 <b>ФАКТЫ И ДОСТИЖЕНИЯ</b>\n"
        "Известные награды, запущенные стартапы, заметные результаты работы.\n\n"

        "━━━━━━━━━━━━━━━━━━━━\n\n"

        "📁 <b>ИСТОЧНИКИ</b>\n"
        "Перечисли имена файлов (txt/xlsx), из которых взяты данные.\n\n"

        "━━━━━━━━━━━━━━━━━━━━\n\n"

        "⚠️ <b>ПРОБЕЛЫ И ПРОТИВОРЕЧИЯ</b>\n"
        "Что НЕ найдено в базе, где данные противоречивы или отсутствуют. "
        "Не додумывай — пиши прямо.\n\n"

        "Если в конкретном блоке данных нет — напиши: <i>«Данных в предоставленном фрагменте нет»</i>."
    )

    user_prompt = (
        f"<b>Объект поиска:</b> <code>{query}</code>\n\n"
        f"<b>Сырые строки из базы:</b>\n{chunk}"
    )

    payload = {
        "model": "gpt-4o",
        "max_tokens": 1800,
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ]
    }

    try:
        async with aiohttp.ClientSession() as sess:
            async with sess.post(
                f"{AITUNNEL_BASE_URL}/chat/completions",
                headers=headers, json=payload,
                timeout=aiohttp.ClientTimeout(total=120)
            ) as resp:
                if resp.status != 200:
                    return "\n".join(raw_results)
                data = await resp.json()
                return (
                    data.get("choices", [{}])[0]
                        .get("message", {})
                        .get("content", "")
                        .strip()
                    or "\n".join(raw_results)
                )
    except Exception:
        return "\n".join(raw_results)

# ==================== SPAM ====================
class AccountManager:
    def __init__(self, session_name: str):
        self.session_name = session_name
        self.session_file = os.path.join(SESSION_DIR, f"{session_name}.session")
        self.client = TelegramClient(self.session_file, TG_API_ID, TG_API_HASH, connection_retries=None)
        self.is_ready = False

    async def disconnect(self):
        try:
            if self.client:
                await self.client.disconnect()
        except Exception:
            pass

def apply_omega(text: str, rules: List[tuple], seed: int) -> str:
    random.seed(seed)
    new_text = re.sub(r'\b\d+\b',
                      lambda m: str(max(1, int(m.group(0)) + random.randint(-5, 5))),
                      text)
    for old, new in rules:
        new_text = new_text.replace(old, new)
    words = new_text.split()
    if len(words) >= 4:
        for _ in range(random.randint(1, 2)):
            i = random.randint(1, len(words) - 2)
            words[i], words[i + 1] = words[i + 1], words[i]
        new_text = ' '.join(words)
    return new_text

async def spam_worker(session_name: str):
    acc = spam_accounts.get(session_name)
    if not acc:
        return
    client = acc['client']
    chats = acc['chats']
    base_text = acc['text']
    interval = acc['interval']
    omega_enabled = acc.get('omega_enabled', False)
    omega_rules = acc.get('omega_rules', [])
    for chat in chats:
        try:
            await client(JoinChannelRequest(chat))
        except Exception as e:
            print(f"⚠️ [{session_name}] {chat}: {e}")
    chat_index = 0
    while acc['is_spamming']:
        for chat in chats:
            if not acc['is_spamming']:
                break
            try:
                text = apply_omega(base_text, omega_rules,
                                   hash(chat + session_name + str(chat_index)) % 1000000) if omega_enabled else base_text
                await client.send_message(chat, text)
                await asyncio.sleep(interval)
                chat_index += 1
            except Exception as e:
                print(f"❌ [{session_name}] {chat}: {e}")
                await asyncio.sleep(10)
        if acc['is_spamming']:
            await asyncio.sleep(1)

# ==================== КЛАВИАТУРЫ ====================
def get_main_keyboard(user_id: int) -> ReplyKeyboardMarkup:
    admin = is_admin(user_id)
    access = has_access(user_id)
    print(f"[KEYBOARD] user_id={user_id} is_admin={admin} has_access={access}")
    buttons = [
        [KeyboardButton(text="🔍 Поиск по базе")],
        [KeyboardButton(text="📊 Статус")],
        [KeyboardButton(text="🔑 Активировать ключ")]
    ]
    if access:
        buttons.insert(1, [KeyboardButton(text="🌍 ГЕО")])
    if admin:
        buttons.append([KeyboardButton(text="📡 Spamless")])
    return ReplyKeyboardMarkup(keyboard=buttons, resize_keyboard=True)

def get_search_keyboard() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="🔙 Выйти из поиска")]],
        resize_keyboard=True
    )

def get_spamless_keyboard() -> ReplyKeyboardMarkup:
    return ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="➕ Подключить аккаунт")],
            [KeyboardButton(text="📋 Мои аккаунты")],
            [KeyboardButton(text="🔙 Выход из Spamless")]
        ],
        resize_keyboard=True
    )

def get_account_inline(name: str, acc: dict) -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📋 Чаты", callback_data=f"sp_chats_{name}")],
        [InlineKeyboardButton(text="📝 Текст", callback_data=f"sp_text_{name}")],
        [InlineKeyboardButton(text="⏱ Интервал", callback_data=f"sp_int_{name}")],
        [InlineKeyboardButton(
            text=f"🔀 OMEGA: {'Выкл' if acc.get('omega_enabled') else 'Вкл'}",
            callback_data=f"sp_omega_{name}")],
        [InlineKeyboardButton(
            text="⏹ Стоп" if acc['is_spamming'] else "▶️ Старт",
            callback_data=f"sp_toggle_{name}")],
        [InlineKeyboardButton(text="🗑 Удалить", callback_data=f"sp_del_{name}")],
        [InlineKeyboardButton(text="🔙 Назад", callback_data="sp_back")]
    ])

# ==================== СТАРТ ====================
@dp.message(Command("start"))
async def cmd_start(message: types.Message, state: FSMContext):
    await state.clear()
    print(f"[START] от user_id={message.from_user.id}")
    caption = (
        f"🌿 <b>Приветствую, {message.from_user.full_name}!</b>\n\n"
        "🟩 Ассистент поиска по архивам и определения геолокации.\n\n"
        "🔍 <b>Поиск по базе</b> — нажмите кнопку и введите запрос.\n"
        "🌍 <b>ГЕО</b> — определение места по фото.\n"
        "📡 <b>Spamless</b> — рассылки (для админов).\n\n"
        "🔑 Активация: <code>/activate [ключ]</code>"
    )
    kb = get_main_keyboard(message.from_user.id)
    if os.path.exists(START_IMG_LOCAL):
        try:
            await message.answer_photo(types.FSInputFile(START_IMG_LOCAL), caption=caption,
                                       parse_mode="HTML", reply_markup=kb)
            return
        except Exception as e:
            print(f"[START] Ошибка отправки фото: {e}")
    await message.answer(caption, parse_mode="HTML", reply_markup=kb)

@dp.message(Command("myid"))
async def cmd_myid(message: types.Message):
    await message.answer(f"🆔 Ваш Telegram ID: <code>{message.from_user.id}</code>", parse_mode="HTML")

@dp.message(F.text == "📊 Статус")
async def button_status(message: types.Message, state: FSMContext):
    await state.clear()
    await show_subscription(message)

@dp.message(F.text == "🔑 Активировать ключ")
async def button_activate(message: types.Message, state: FSMContext):
    await state.clear()
    await message.answer("⚠️ <code>/activate [ключ]</code>", parse_mode="HTML")

# ==================== ПОИСК ====================
@dp.message(F.text == "🔍 Поиск по базе")
async def search_start(message: types.Message, state: FSMContext):
    if not has_access(message.from_user.id):
        await message.answer("🔒 Доступ ограничен. <code>/activate [ключ]</code>", parse_mode="HTML")
        return
    await state.set_state(SearchStates.waiting_query)
    await message.answer(
        "🔍 <b>Режим поиска по базе</b>\n\n"
        "Отправьте запрос — найду совпадения в txt/xlsx и соберу OSINT-отчёт.\n\n"
        "Для выхода нажмите <b>🔙 Выйти из поиска</b>.",
        parse_mode="HTML",
        reply_markup=get_search_keyboard()
    )

@dp.message(F.text == "🔙 Выйти из поиска")
async def search_exit(message: types.Message, state: FSMContext):
    await state.clear()
    await message.answer("🔙 Вы вышли из режима поиска.",
                         reply_markup=get_main_keyboard(message.from_user.id))

@dp.message(StateFilter(SearchStates.waiting_query), F.text)
async def search_process(message: types.Message, state: FSMContext):
    if message.text in SERVICE_BUTTONS:
        return
    query = message.text.strip()
    if len(query) < 2:
        await message.answer("⚠️ Слишком короткий запрос. Минимум 2 символа.")
        return
    sm = await message.answer("🟩 <i>Сканирую базу...</i>", parse_mode="HTML")
    found = search_in_files(query)
    if not found:
        await sm.edit_text("🤷‍♂️ Ничего не найдено по вашему запросу.")
        return
    await sm.edit_text(
        f"🟩 Найдено <b>{len(found)}</b> совпадений.\n🧠 Составляю OSINT-отчёт...",
        parse_mode="HTML"
    )
    report = await ai_merge_search_results(query, found)
    header = (
        f"<b>╔══════════════════════════╗</b>\n"
        f"<b>   🕵️ OSINT-ОТЧЁТ ПО ЗАПРОСУ</b>\n"
        f"<b>╚══════════════════════════╝</b>\n\n"
        f"🔍 <b>Объект:</b> <code>{query}</code>\n"
        f"📦 <b>Найдено совпадений:</b> <b>{len(found)}</b>\n\n"
    )
    full = header + report
    for i in range(0, len(full), 4000):
        await message.answer(full[i:i+4000], parse_mode="HTML")
    raw = "\n".join(found[:20])
    if len(raw) > 3500:
        raw = raw[:3500] + "..."
    await message.answer(
        f"<b>╔══════════════════════════╗</b>\n"
        f"<b>   📄 СЫРЫЕ ДАННЫЕ</b>\n"
        f"<b>╚══════════════════════════╝</b>\n\n"
        f"<blockquote><code>{raw}</code></blockquote>",
        parse_mode="HTML"
    )

# ==================== ГЕО ====================
@dp.message(F.text == "🌍 ГЕО")
@dp.message(Command("geo"))
async def button_geo(message: types.Message, state: FSMContext):
    if not has_access(message.from_user.id):
        await message.answer("🔒 Доступ ограничен. <code>/activate [ключ]</code>", parse_mode="HTML")
        return
    await message.answer("🌍 <b>Режим ГЕО</b>\n\nОтправьте фотографию.", parse_mode="HTML")
    await state.set_state(GeoStates.waiting_photo)

@dp.message(StateFilter(GeoStates.waiting_photo), F.photo)
async def process_geo_photo(message: types.Message, state: FSMContext):
    await state.clear()
    sm = await message.answer("🔍 <i>Анализирую...</i>", parse_mode="HTML")
    try:
        photo = message.photo[-1]
        file = await bot.get_file(photo.file_id)
        pb = await bot.download_file(file.file_path)
        result = await analyze_photo_geo(pb.read())
        await sm.edit_text(f"🌍 <b>Результат:</b>\n\n{result}", parse_mode="HTML")
    except Exception as e:
        await sm.edit_text(f"❌ Ошибка: <code>{e}</code>", parse_mode="HTML")

@dp.message(StateFilter(GeoStates.waiting_photo))
async def geo_wrong(message: types.Message):
    await message.answer("📷 Отправьте именно фотографию.")

# ==================== SPAMLESS ====================
@dp.message(F.text == "📡 Spamless")
async def spamless_menu(message: types.Message, state: FSMContext):
    print(f"[SPAMLESS] нажат от user_id={message.from_user.id} is_admin={is_admin(message.from_user.id)}")
    if not is_admin(message.from_user.id):
        await message.answer("⛔ Нет доступа к Spamless.\nОтправьте /myid и добавьте ID в CREATORS.")
        return
    await state.clear()
    await message.answer(
        "📡 <b>Spamless</b>\n\nВыберите действие на клавиатуре снизу:",
        parse_mode="HTML",
        reply_markup=get_spamless_keyboard()
    )

@dp.message(F.text == "🔙 Выход из Spamless")
async def spamless_exit(message: types.Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    await state.clear()
    await message.answer("🔙 Вы вышли из Spamless.",
                         reply_markup=get_main_keyboard(message.from_user.id))

@dp.message(F.text == "📋 Мои аккаунты")
async def spamless_accounts(message: types.Message):
    if not is_admin(message.from_user.id):
        return
    if not spam_accounts:
        await message.answer("📭 <b>Нет подключённых аккаунтов.</b>", parse_mode="HTML",
                             reply_markup=get_spamless_keyboard())
        return
    for name, acc in spam_accounts.items():
        status = "🟢 Работает" if acc['is_spamming'] else "🔴 Остановлен"
        omega = "🔛 Вкл" if acc.get('omega_enabled') else "🔕 Выкл"
        text = (
            f"⚙️ <b>Аккаунт:</b> <code>{name}</code>\n\n"
            f"📊 Статус: {status}\n"
            f"📋 Чатов: {len(acc['chats'])}\n"
            f"⏱ Интервал: {acc['interval']} сек\n"
            f"🔀 OMEGA: {omega}\n"
            f"📝 Текст: {acc['text'][:60]}..."
        )
        await message.answer(text, parse_mode="HTML", reply_markup=get_account_inline(name, acc))

@dp.message(F.text == "➕ Подключить аккаунт")
async def sp_connect_start(message: types.Message, state: FSMContext):
    if not is_admin(message.from_user.id):
        return
    await state.set_state(SpamStates.waiting_name)
    await message.answer(
        "✏️ <b>Шаг 1/4. Введите имя аккаунта</b>\n"
        "Латинские буквы, цифры, подчёркивание.\n"
        "Пример: <code>main</code>, <code>work1</code>",
        parse_mode="HTML",
        reply_markup=get_spamless_keyboard()
    )

@dp.message(SpamStates.waiting_name, F.text)
async def sp_name(message: types.Message, state: FSMContext):
    name = message.text.strip()
    if name in SERVICE_BUTTONS:
        await state.clear()
        await spamless_menu(message, state)
        return
    if not name.replace('_', '').isalnum():
        await message.answer("❌ Только латиница, цифры, _")
        return
    if name in spam_accounts:
        await message.answer(f"❌ Имя <code>{name}</code> уже занято.", parse_mode="HTML")
        return
    await state.update_data(name=name)
    await state.set_state(SpamStates.waiting_phone)
    await message.answer(
        f"📱 <b>Шаг 2/4. Телефон для</b> <code>{name}</code>\n"
        "Формат: <code>+7XXXXXXXXXX</code>",
        parse_mode="HTML"
    )

@dp.message(SpamStates.waiting_phone, F.text)
async def sp_phone(message: types.Message, state: FSMContext):
    phone = message.text.strip()
    if not re.match(r'^\+?\d{10,15}$', phone):
        await message.answer("❌ Неверный формат. Пример: +79123456789")
        return
    data = await state.get_data()
    name = data['name']
    manager = AccountManager(name)
    try:
        await manager.client.connect()
    except Exception as e:
        await message.answer(f"❌ Ошибка соединения: {e}")
        await state.clear()
        return
    try:
        sent = await manager.client.send_code_request(phone)
        pending_logins[message.from_user.id] = {
            "name": name, "phone": phone, "client": manager.client,
            "phone_code_hash": sent.phone_code_hash
        }
        await state.set_state(SpamStates.waiting_code)
        await message.answer(
            "📨 <b>Шаг 3/4. Введите код из Telegram</b>\n\n"
            "Код придёт в приложение Telegram или SMS.\n"
            "Введите его сюда сообщением.",
            parse_mode="HTML"
        )
    except Exception as e:
        await manager.disconnect()
        await message.answer(f"❌ Не удалось отправить код: {e}")
        await state.clear()

@dp.message(SpamStates.waiting_code, F.text)
async def sp_code(message: types.Message, state: FSMContext):
    code = message.text.strip().replace(" ", "")
    info = pending_logins.get(message.from_user.id)
    if not info:
        await message.answer("❌ Сессия не найдена.")
        await state.clear()
        return
    client = info["client"]
    try:
        await client.sign_in(phone=info["phone"], code=code,
                             phone_code_hash=info["phone_code_hash"])
        await finalize_login(message, state, info)
    except SessionPasswordNeededError:
        await state.set_state(SpamStates.waiting_password)
        await message.answer(
            "🔐 <b>Шаг 4/4. Введите пароль 2FA</b>\n"
            "Отправьте пароль сообщением.", parse_mode="HTML"
        )
    except (PhoneCodeInvalidError, PhoneCodeExpiredError):
        await message.answer("❌ Неверный или просроченный код.")
    except Exception as e:
        await message.answer(f"❌ Ошибка: {e}")
        await state.clear()
        try:
            await client.disconnect()
        except Exception:
            pass
        pending_logins.pop(message.from_user.id, None)

@dp.message(SpamStates.waiting_password, F.text)
async def sp_password(message: types.Message, state: FSMContext):
    password = message.text.strip()
    info = pending_logins.get(message.from_user.id)
    if not info:
        await message.answer("❌ Сессия не найдена.")
        await state.clear()
        return
    client = info["client"]
    try:
        await client.sign_in(password=password)
        await finalize_login(message, state, info)
    except Exception as e:
        await message.answer(f"❌ Ошибка: {e}")

async def finalize_login(message: types.Message, state: FSMContext, info: dict):
    name = info["name"]
    client = info["client"]
    if not await client.is_user_authorized():
        await message.answer("❌ Авторизация не подтверждена.")
        await state.clear()
        return
    spam_accounts[name] = {
        'client': client, 'chats': [], 'text': "💝 Привет! Это тестовое сообщение.",
        'interval': 5, 'is_spamming': False, 'task': None,
        'omega_enabled': False, 'omega_rules': []
    }
    pending_logins.pop(message.from_user.id, None)
    await state.clear()
    await message.answer(
        f"✅ <b>Аккаунт</b> <code>{name}</code> <b>успешно подключён!</b>",
        parse_mode="HTML", reply_markup=get_spamless_keyboard()
    )

# ==================== INLINE ====================
@dp.callback_query(F.data == "sp_back")
async def sp_back(cb: types.CallbackQuery):
    await cb.answer()
    try:
        await cb.message.delete()
    except Exception:
        pass

@dp.callback_query(F.data.startswith("sp_chats_"))
async def sp_chats(cb: types.CallbackQuery, state: FSMContext):
    await cb.answer()
    name = cb.data.replace("sp_chats_", "")
    acc = spam_accounts.get(name)
    if not acc:
        return
    await state.set_state(SpamStates.waiting_chat)
    await state.update_data(acc_name=name)
    chats_list = "\n".join(acc['chats']) if acc['chats'] else "— пусто —"
    await cb.message.edit_text(
        f"📋 <b>Чаты {name}</b>\n\n{chats_list}\n\n"
        "Отправьте юзернеймы через запятую/пробел или .txt-файлом.\n"
        "Удаление: <code>-@chat</code>",
        parse_mode="HTML"
    )

async def process_chat_parts(parts, acc):
    added, removed = [], []
    for part in parts:
        part = part.strip()
        if not part:
            continue
        if 't.me/' in part:
            username = part.split('t.me/')[-1].split('?')[0].split('/')[0]
            if username:
                part = '@' + username
        if part.startswith('-'):
            chat = part[1:].strip()
            if not chat.startswith('@'):
                chat = '@' + chat
            if chat in acc['chats']:
                acc['chats'].remove(chat)
                removed.append(chat)
        else:
            if not part.startswith('@'):
                part = '@' + part
            if part not in acc['chats']:
                acc['chats'].append(part)
                added.append(part)
    return added, removed

@dp.message(SpamStates.waiting_chat, F.text)
async def sp_chat_text(message: types.Message, state: FSMContext):
    if message.text in SERVICE_BUTTONS:
        await state.clear()
        await spamless_menu(message, state)
        return
    data = await state.get_data()
    name = data.get('acc_name')
    acc = spam_accounts.get(name)
    if not acc:
        await message.answer("❌ Аккаунт не найден.")
        await state.clear()
        return
    parts = re.split(r'[,\s\n]+', message.text.strip())
    added, removed = await process_chat_parts(parts, acc)
    reply = ""
    if added: reply += f"✅ Добавлено: {', '.join(added)}\n"
    if removed: reply += f"🗑 Удалено: {', '.join(removed)}\n"
    if not reply: reply = "⚠️ Ничего не изменилось.\n"
    cl = "\n".join(acc['chats']) if acc['chats'] else "— пусто —"
    reply += f"\n📋 <b>Текущие чаты:</b>\n{cl}"
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ Ещё", callback_data=f"sp_addmore_{name}")],
        [InlineKeyboardButton(text="✅ Готово", callback_data=f"sp_done_{name}")]
    ])
    await message.answer(reply, parse_mode="HTML", reply_markup=kb)

@dp.message(SpamStates.waiting_chat, F.document)
async def sp_chat_file(message: types.Message, state: FSMContext):
    data = await state.get_data()
    name = data.get('acc_name')
    acc = spam_accounts.get(name)
    if not acc:
        await message.answer("❌ Аккаунт не найден.")
        await state.clear()
        return
    if not message.document.file_name.endswith('.txt'):
        await message.answer("⚠️ Только .txt")
        return
    file = await bot.get_file(message.document.file_id)
    fb = await bot.download_file(file.file_path)
    content = fb.read().decode('utf-8', errors='ignore')
    parts = [l.strip() for l in content.splitlines() if l.strip()]
    added, removed = await process_chat_parts(parts, acc)
    reply = f"✅ Добавлено: {', '.join(added) if added else '—'}\n"
    if removed: reply += f"🗑 Удалено: {', '.join(removed)}\n"
    cl = "\n".join(acc['chats']) if acc['chats'] else "— пусто —"
    reply += f"\n📋 <b>Текущие чаты:</b>\n{cl}"
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ Ещё", callback_data=f"sp_addmore_{name}")],
        [InlineKeyboardButton(text="✅ Готово", callback_data=f"sp_done_{name}")]
    ])
    await message.answer(reply, parse_mode="HTML", reply_markup=kb)

@dp.callback_query(F.data.startswith("sp_addmore_"))
async def sp_addmore(cb: types.CallbackQuery, state: FSMContext):
    await cb.answer()
    name = cb.data.replace("sp_addmore_", "")
    await state.set_state(SpamStates.waiting_chat)
    await state.update_data(acc_name=name)
    await cb.message.answer("✏️ Отправьте ещё чаты.")

@dp.callback_query(F.data.startswith("sp_done_"))
async def sp_done(cb: types.CallbackQuery, state: FSMContext):
    await cb.answer()
    name = cb.data.replace("sp_done_", "")
    await state.clear()
    await cb.message.answer(f"✅ Чаты для <code>{name}</code> сохранены.", parse_mode="HTML",
                            reply_markup=get_spamless_keyboard())

@dp.callback_query(F.data.startswith("sp_text_"))
async def sp_text_cb(cb: types.CallbackQuery, state: FSMContext):
    await cb.answer()
    name = cb.data.replace("sp_text_", "")
    await state.set_state(SpamStates.waiting_text)
    await state.update_data(acc_name=name)
    await cb.message.answer(
        f"✏️ Введите новый текст для <code>{name}</code>.\n\n"
        f"Текущий:\n{spam_accounts[name]['text']}", parse_mode="HTML"
    )

@dp.message(SpamStates.waiting_text, F.text)
async def sp_text_set(message: types.Message, state: FSMContext):
    if message.text in SERVICE_BUTTONS:
        await state.clear()
        await spamless_menu(message, state)
        return
    data = await state.get_data()
    name = data.get('acc_name')
    acc = spam_accounts.get(name)
    if not acc:
        await message.answer("❌ Аккаунт не найден.")
        await state.clear()
        return
    acc['text'] = message.text
    await state.clear()
    await message.answer(f"✅ Текст для <code>{name}</code> обновлён.", parse_mode="HTML",
                         reply_markup=get_spamless_keyboard())

@dp.callback_query(F.data.startswith("sp_int_"))
async def sp_int_cb(cb: types.CallbackQuery, state: FSMContext):
    await cb.answer()
    name = cb.data.replace("sp_int_", "")
    await state.set_state(SpamStates.waiting_interval)
    await state.update_data(acc_name=name)
    await cb.message.answer(
        f"⏱ Введите интервал в секундах для <code>{name}</code>.\n"
        f"Текущий: {spam_accounts[name]['interval']}", parse_mode="HTML"
    )

@dp.message(SpamStates.waiting_interval, F.text)
async def sp_int_set(message: types.Message, state: FSMContext):
    if message.text in SERVICE_BUTTONS:
        await state.clear()
        await spamless_menu(message, state)
        return
    data = await state.get_data()
    name = data.get('acc_name')
    acc = spam_accounts.get(name)
    if not acc:
        await message.answer("❌ Аккаунт не найден.")
        await state.clear()
        return
    try:
        val = int(message.text)
        if val < 1: raise ValueError
        acc['interval'] = val
        await state.clear()
        await message.answer(f"✅ Интервал: {val} сек.", parse_mode="HTML",
                             reply_markup=get_spamless_keyboard())
    except ValueError:
        await message.answer("❌ Введите целое число >= 1.")

@dp.callback_query(F.data.startswith("sp_omega_"))
async def sp_omega_cb(cb: types.CallbackQuery):
    await cb.answer()
    name = cb.data.replace("sp_omega_", "")
    acc = spam_accounts.get(name)
    if not acc: return
    acc['omega_enabled'] = not acc.get('omega_enabled', False)
    await cb.message.answer(
        f"🔀 OMEGA для <code>{name}</code>: "
        f"{'включена' if acc['omega_enabled'] else 'выключена'}.", parse_mode="HTML"
    )

@dp.callback_query(F.data.startswith("sp_toggle_"))
async def sp_toggle_cb(cb: types.CallbackQuery):
    await cb.answer()
    name = cb.data.replace("sp_toggle_", "")
    acc = spam_accounts.get(name)
    if not acc: return
    if acc['is_spamming']:
        acc['is_spamming'] = False
        if acc['task']:
            acc['task'].cancel()
            try: await acc['task']
            except asyncio.CancelledError: pass
            acc['task'] = None
        await cb.message.answer(f"⏹ Рассылка <code>{name}</code> остановлена.", parse_mode="HTML")
    else:
        if not acc['chats']:
            await cb.message.answer("❌ Нет чатов."); return
        if not acc['text']:
            await cb.message.answer("❌ Нет текста."); return
        acc['is_spamming'] = True
        acc['task'] = asyncio.create_task(spam_worker(name))
        await cb.message.answer(f"🚀 Рассылка <code>{name}</code> запущена!", parse_mode="HTML")

@dp.callback_query(F.data.startswith("sp_del_"))
async def sp_del_cb(cb: types.CallbackQuery):
    await cb.answer()
    name = cb.data.replace("sp_del_", "")
    acc = spam_accounts.get(name)
    if not acc: return
    if acc['is_spamming']:
        acc['is_spamming'] = False
        if acc['task']: acc['task'].cancel()
    await acc['client'].disconnect()
    del spam_accounts[name]
    await cb.message.answer(f"🗑 Аккаунт <code>{name}</code> удалён.", parse_mode="HTML")

# ==================== КОМАНДЫ ====================
@dp.message(Command("status"))
async def show_subscription(message: types.Message):
    uid = message.from_user.id
    if uid in CREATORS:
        await message.answer("👑 <b>Статус:</b> Главный Создатель", parse_mode="HTML")
    elif uid in ADMIN_IDS:
        await message.answer("⚡ <b>Статус:</b> Администратор", parse_mode="HTML")
    elif uid in ACTIVE_USERS and datetime.now() < ACTIVE_USERS[uid]["exp"]:
        exp = ACTIVE_USERS[uid]["exp"]
        await message.answer(f"🟢 Лицензия до <code>{exp.strftime('%d.%m.%Y %H:%M')}</code>", parse_mode="HTML")
    else:
        await message.answer("⚠️ Нет доступа.", parse_mode="HTML")

@dp.message(Command("activate"))
async def cmd_activate(message: types.Message):
    args = message.text.split()
    if len(args) < 2:
        await message.answer("⚠️ <code>/activate [ключ]</code>", parse_mode="HTML"); return
    key = args[1].strip()
    if key in GENERATED_KEYS:
        kd = GENERATED_KEYS[key]
        if message.from_user.id in ACTIVE_USERS and datetime.now() < ACTIVE_USERS[message.from_user.id]["exp"]:
            await message.answer("📢 Уже есть подписка!"); return
        exp = datetime.now() + timedelta(hours=kd["duration_hours"])
        username = f"@{message.from_user.username}" if message.from_user.username else message.from_user.full_name
        ACTIVE_USERS[message.from_user.id] = {"exp": exp, "username": username, "status": "Активен"}
        kd["uses_left"] -= 1
        if kd["uses_left"] <= 0:
            del GENERATED_KEYS[key]
        await message.answer(f"🟩 Лицензия до <code>{exp.strftime('%d.%m.%Y %H:%M')}</code>", parse_mode="HTML")
    else:
        await message.answer("❌ Ключ недействителен.", parse_mode="HTML")

@dp.message(Command("members"))
async def cmd_members(message: types.Message):
    if not is_admin(message.from_user.id): return
    text = "👥 <b>Пользователи:</b>\n\n👑 Создатели:\n"
    for c in CREATORS: text += f" ├ <code>{c}</code>\n"
    if ADMIN_IDS:
        text += "\n⚡ Админы:\n"
        for a in ADMIN_IDS: text += f" ├ <code>{a}</code>\n"
    text += "\n🌿 Подписки:\n"
    now = datetime.now()
    for u in [u for u, d in ACTIVE_USERS.items() if now > d["exp"]]:
        del ACTIVE_USERS[u]
    if not ACTIVE_USERS:
        text += " └ <i>Нет</i>"
    else:
        for uid, d in ACTIVE_USERS.items():
            left = d["exp"] - now
            h = int(left.total_seconds() // 3600); m = int((left.total_seconds() % 3600) // 60)
            text += f" ├ <b>{d['username']}</b> (<code>{uid}</code>) — {h}ч {m}м\n"
    await message.answer(text, parse_mode="HTML")

@dp.message(Command("create_key"))
async def cmd_create_key(message: types.Message):
    if not is_admin(message.from_user.id): return
    args = message.text.split()
    if len(args) < 3 or not args[1].isdigit() or not args[2].isdigit():
        await message.answer("⚠️ <code>/create_key [кол-во] [часы]</code>", parse_mode="HTML"); return
    uses = int(args[1]); hours = int(args[2])
    if uses <= 0 or hours <= 0:
        await message.answer("❌ Больше нуля."); return
    new_key = f"GREEN-{uuid.uuid4().hex[:8].upper()}"
    GENERATED_KEYS[new_key] = {"uses_left": uses, "duration_hours": hours}
    await message.answer(
        f"🟩 Ключ: <code>{new_key}</code>\n👥 {uses}\n⏳ {hours}ч\n\n<code>/activate {new_key}</code>",
        parse_mode="HTML")

@dp.message(Command("grant_admin"))
async def cmd_grant_admin(message: types.Message):
    if message.from_user.id not in CREATORS: return
    args = message.text.split()
    if len(args) < 2 or not args[1].isdigit():
        await message.answer("⚠️ <code>/grant_admin [ID]</code>", parse_mode="HTML"); return
    ADMIN_IDS.add(int(args[1]))
    await message.answer(f"👑 <code>{args[1]}</code> — Админ.", parse_mode="HTML")

@dp.message(Command("revoke_admin"))
async def cmd_revoke_admin(message: types.Message):
    if message.from_user.id not in CREATORS: return
    args = message.text.split()
    if len(args) < 2 or not args[1].isdigit():
        await message.answer("⚠️ <code>/revoke_admin [ID]</code>", parse_mode="HTML"); return
    tid = int(args[1])
    if tid in ADMIN_IDS:
        ADMIN_IDS.remove(tid)
        await message.answer(f"❌ <code>{tid}</code> снят.", parse_mode="HTML")
    else:
        await message.answer("❌ Не найден.")

@dp.message(F.document)
async def handle_docs(message: types.Message, state: FSMContext):
    if await state.get_state() == SpamStates.waiting_chat.state:
        return
    if not is_admin(message.from_user.id): return
    if message.document.file_name.endswith((".txt", ".xlsx")):
        file = await bot.get_file(message.document.file_id)
        dest = os.path.join(DATA_DIR, message.document.file_name)
        await bot.download_file(file.file_path, dest)
        await message.answer(f"🟢 Файл <code>{message.document.file_name}</code> добавлен.", parse_mode="HTML")
    else:
        await message.answer("❌ Только .txt и .xlsx")

# ==================== ЗАПУСК ====================
async def main():
    print("=" * 50)
    print("БОТ ЗАПУЩЕН")
    print(f"CREATORS: {CREATORS}")
    print("=" * 50)
    await download_start_image()
    commands = [
        types.BotCommand(command="start", description="Главное меню"),
        types.BotCommand(command="myid", description="Узнать свой Telegram ID"),
        types.BotCommand(command="status", description="Проверить подписку"),
        types.BotCommand(command="activate", description="Активировать ключ"),
        types.BotCommand(command="geo", description="🌍 Геолокация по фото"),
        types.BotCommand(command="members", description="Список пользователей"),
        types.BotCommand(command="create_key", description="Создать ключ"),
        types.BotCommand(command="grant_admin", description="Назначить админа"),
        types.BotCommand(command="revoke_admin", description="Снять админа"),
    ]
    try:
        await bot.set_my_commands(commands)
    except Exception as e:
        logging.warning(f"Команды: {e}")

    while True:
        try:
            await dp.start_polling(bot, skip_updates=True)
        except (AiogramError, aiohttp.ClientError, OSError) as e:
            logging.error(f"Сеть: {e}. Рестарт через 15 сек...")
            await asyncio.sleep(15)
        except KeyboardInterrupt:
            break

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("Бот выключен.")
