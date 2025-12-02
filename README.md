"""
Meta Freshman Bot — resilient version

This file contains two modes:
1) Real bot mode — uses aiogram and runs as a Telegram bot (requires working `ssl`/network environment).
2) Simulation / dry-run mode — used when the sandbox lacks the `ssl` module (or other networking dependencies). It provides a local, terminal-based simulation of the bot and simple unit-like tests so you can continue developing and verifying handlers without a live Telegram connection.

When run in an environment that has `ssl` and can import `aiogram`, the real bot code will execute. When `ssl` is missing (ModuleNotFoundError when importing `ssl`), the script falls back to the simulator.

If you want changes to handler text, flows, or additional commands, tell me in chat what the expected behaviour is and I will update the code.

"""

# ----------------------
# Imports & environment check
# ----------------------
import logging
import sys

logging.basicConfig(level=logging.INFO)

# Try to import aiogram. If an import error occurs (commonly because 'ssl' is missing
# in the sandbox), fall back to a local simulator so the user can continue development.
USE_REAL_BOT = True
try:
    # Import inside try so missing ssl triggers here
    from aiogram import Bot, Dispatcher, types
    from aiogram.utils import executor
except Exception as e:
    USE_REAL_BOT = False
    logging.warning("Falling back to simulator mode because aiogram import failed: %s", e)


# ----------------------
# Shared handler logic (pure functions for easy testing)
# ----------------------

def format_admission_text() -> str:
    return (
        "📘 *Как поступить в META?*\n\n"
        "1️⃣ Подать онлайн-заявление на сайте META.\n"
        "2️⃣ Пройти регистрацию и загрузить документы.\n"
        "3️⃣ Сдать вступительные экзамены или предоставить результаты тестирований.\n"
        "4️⃣ Дождаться приказа о зачислении.\n\n"
        "Хочешь узнать список документов? Напиши: *Документы*."
    )


def format_documents_text() -> str:
    return (
        "🗂️ *Документы для поступления в META:*\n"
        "• Удостоверение личности\n"
        "• Аттестат или диплом\n"
        "• Фото 3×4\n"
        "• Медсправка 075У\n"
        "• Заявление (заполняется онлайн)"
    )


def format_locations_prompt() -> str:
    return "Что именно интересует?"


def format_corp_info() -> str:
    return (
        "🏫 *Корпуса META:*\n\n"
        "• Главный корпус — ул. Центральная 12\n"
        "• IT-корпус — ул. Цифровая 5\n"
        "• Научный центр — пр. Инноваций 21"
    )


def format_dekan_info() -> str:
    return (
        "📍 *Деканаты факультетов:*\n\n"
        "• IT факультет — каб. 203 (Главный корпус)\n"
        "• Бизнес факультет — каб. 118 (Главный корпус)\n"
        "• Дизайн — 2 этаж (IT-корпус)"
    )


def format_other_info() -> str:
    return "🍽️ *Столовая:* Главный корпус, цокольный этаж\n🏋️ *Спортзал:* IT-корпус, 1 этаж"


def format_contacts_text() -> str:
    return (
        "☎️ *Контакты META:*\n"
        "Приёмная комиссия: +7 777 000 00 00\n"
        "Email: admissions@meta.edu\n"
        "Официальный сайт: meta.edu"
    )


# ----------------------
# Real bot implementation (runs only if aiogram imported successfully)
# ----------------------
if USE_REAL_BOT:
    API_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"

    bot = Bot(token=API_TOKEN)
    dp = Dispatcher(bot)

    # --- Main menu keyboard ---
    main_kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    main_kb.add("Как поступить?", "Где что находится?", "Контакты META")

    # --- Start Command ---
    @dp.message_handler(commands=["start", "help"])
    async def send_welcome(message: types.Message):
        await message.answer(
            "Привет! Я бот-помощник первокурсников университета META. Чем помочь?",
            reply_markup=main_kb,
        )

    # --- Admission Info ---
    @dp.message_handler(lambda m: m.text == "Как поступить?")
    async def admission_info(message: types.Message):
        await message.answer(format_admission_text(), parse_mode="Markdown")

    @dp.message_handler(lambda m: m.text and m.text.lower() == "документы")
    async def docs(message: types.Message):
        await message.answer(format_documents_text(), parse_mode="Markdown")

    # --- Locations ---
    @dp.message_handler(lambda m: m.text == "Где что находится?")
    async def locations(message: types.Message):
        kb = types.InlineKeyboardMarkup()
        kb.add(types.InlineKeyboardButton("Корпуса", callback_data="corp"))
        kb.add(types.InlineKeyboardButton("Деканаты", callback_data="dekan"))
        kb.add(types.InlineKeyboardButton("Столовая / Спортзал", callback_data="other"))
        await message.answer(format_locations_prompt(), reply_markup=kb)

    @dp.callback_query_handler(lambda c: c.data == "corp")
    async def corp_info(callback: types.CallbackQuery):
        await callback.message.answer(format_corp_info(), parse_mode="Markdown")
        await callback.answer()

    @dp.callback_query_handler(lambda c: c.data == "dekan")
    async def dekan_info(callback: types.CallbackQuery):
        await callback.message.answer(format_dekan_info(), parse_mode="Markdown")
        await callback.answer()

    @dp.callback_query_handler(lambda c: c.data == "other")
    async def other_info(callback: types.CallbackQuery):
        await callback.message.answer(format_other_info(), parse_mode="Markdown")
        await callback.answer()

    # --- Contacts ---
    @dp.message_handler(lambda m: m.text == "Контакты META")
    async def contacts(message: types.Message):
        await message.answer(format_contacts_text(), parse_mode="Markdown")

    # --- Run Bot ---
    if __name__ == "__main__":
        # NOTE: in the sandbox this branch will likely not run due to missing ssl.
        logging.info("Starting real Telegram bot (aiogram). If this fails, check ssl and network.")
        executor.start_polling(dp, skip_updates=True)


# ----------------------
# Simulator (runs when aiogram or ssl unavailable)
# ----------------------
else:
    # Simple mapping-based simulator for local testing.
    PROMPT = (
        "=== META Freshman Bot Simulator ===\n"
        "Введите команду или текст (например: 'Как поступить?', 'Документы', 'Где что находится?', 'Корпуса', 'Контакты META').\n"
        "Напишите 'exit' чтобы выйти.\n"
    )

    def handle_message_simulator(text: str) -> str:
        if not text:
            return "Я не расслышал — введите команду."
        t = text.strip()
        if t == "/start" or t.lower() in ("привет", "start", "help"):
            return "Привет! Я бот-помощник первокурсников университета META. Чем помочь?"
        if t == "Как поступить?":
            return format_admission_text()
        if t.lower() == "документы":
            return format_documents_text()
        if t == "Где что находится?":
            # In the real bot this would show inline buttons. Simulator returns list.
            return "Корпуса | Деканаты | Столовая / Спортзал"
        if t == "Корпуса":
            return format_corp_info()
        if t == "Деканаты":
            return format_dekan_info()
        if t == "Столовая" or t == "Столовая / Спортзал":
            return format_other_info()
        if t == "Контакты META":
            return format_contacts_text()
        return "Команда не распознана. Попробуйте: 'Как поступить?', 'Документы', 'Где что находится?', 'Контакты META'"

    # Basic tests — more can be added. These run automatically when executing the file in simulator mode.
    def run_tests():
        tests = [
            ("Как поступить?", "Как поступить в META"),
            ("документы", "Документы для поступления"),
            ("Где что находится?", "Корпуса | Деканаты | Столовая / Спортзал"),
            ("Корпуса", "Главный корпус"),
            ("Контакты META", "Приёмная комиссия"),
        ]
        passed = 0
        for i, (inp, expect_substr) in enumerate(tests, 1):
            out = handle_message_simulator(inp)
            ok = expect_substr in out
            print(f"Test {i}: input={inp!r} -> {'PASS' if ok else 'FAIL'}")
            if not ok:
                print("  Expected substring:", expect_substr)
                print("  Got:", out)
            else:
                passed += 1
        print(f"{passed}/{len(tests)} tests passed.")

    if __name__ == "__main__":
        print("aiogram or ssl not available — running simulator instead.")
        # Run tests first
        run_tests()
        # Then interactive loop
        try:
            while True:
                user_in = input("\nYou: ").strip()
                if user_in.lower() in ("exit", "quit"):
                    print("Выход. Пока!")
                    break
                resp = handle_message_simulator(user_in)
                print("Bot:", resp)
        except (KeyboardInterrupt, EOFError):
            print("\nВыход. Пока!")


# ----------------------
# Development note for the user
# ----------------------
# The error you hit (ModuleNotFoundError: No module named 'ssl') indicates the Python
# runtime in your environment lacks the 'ssl' module, which aiohttp/aiogram need for HTTPS.
# Fixes (choose one):
#  - Run this bot in an environment with full Python stdlib that includes 'ssl' (recommended).
#  - Use a hosting provider (Heroku, Railway, VPS, etc.) where ssl is available.
#  - If you want, I can rework this code to use a webhook over plain HTTP — that still requires
#    networking and TLS in production, so generally moving to a proper environment is best.
#
# If you'd like, tell me what behaviour you expect for any command that's ambiguous and I'll
# update responses or add new features (schedules, maps, file attachments, user DB, locale support).

