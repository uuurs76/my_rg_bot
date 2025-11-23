
import json
import os
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, ConversationHandler

# ВСТАВЬ СЮДА СВОЙ ТОКЕН ОТ @BotFather
BOT_TOKEN = "8579449874:AAGx4xpEHF8rfd3NlVVy43P5v4zPjkr7jMI"

# Файл для хранения всех жалоб
DATA_FILE = "reports.json"

# Состояния разговора
PHOTO, LOCATION = range(2)

# Временное хранилище для данных пользователей
user_data = {}

def load_data():
    """Загружает данные из файла"""
    if not os.path.exists(DATA_FILE):
        return []
    with open(DATA_FILE, 'r', encoding='utf-8') as f:
        return json.load(f)

def save_data(data):
    """Сохраняет данные в файл"""
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Реакция на команду /start"""
    user = update.message.from_user
    await update.message.reply_text(
        f'Привет, {user.first_name}! Я бот-экопатруль.\n'
        'Отправь мне фото экологической проблемы!'
    )
    return PHOTO

async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обрабатывает фото"""
    user = update.message.from_user
    
    # Сохраняем фото во временное хранилище
    photo_file = await update.message.photo[-1].get_file()
    user_data[user.id] = {
        'photo_id': photo_file.file_id,
        'user_name': user.first_name,
        'user_id': user.id,
        'description': update.message.caption or "Без описания"
    }
    
    await update.message.reply_text(
        'Отличное фото! 📸\n'
        'Теперь отправь геолокацию этого места.'
    )
    return LOCATION

async def handle_location(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обрабатывает геолокацию"""
    user = update.message.from_user
    
    if user.id not in user_data:
        await update.message.reply_text(
            'Сначала отправь фото проблемы!'
        )
        return PHOTO
    
    # Получаем локацию
    location = update.message.location
    lat, lon = location.latitude, location.longitude
    
    # Получаем данные из временного хранилища
    report_data = user_data[user.id]
    
    # Создаём полную запись о проблеме
    report = {
        "user_id": report_data['user_id'],
        "user_name": report_data['user_name'],
        "photo_id": report_data['photo_id'],
        "latitude": lat,
        "longitude": lon,
        "description": report_data['description']
    }

    # Сохраняем в файл
    data = load_data()
    data.append(report)
    save_data(data)
    
    # Очищаем временные данные
    del user_data[user.id]

    await update.message.reply_text(
        f'✅ Готово! Жалоба добавлена на карту.\n'
        f'Координаты: {lat:.4f}, {lon:.4f}\n'
        f'Спасибо за вклад в чистоту города! 🌱\n\n'
        f'Чтобы отправить новую проблему, напиши /start'
    )
    return ConversationHandler.END

async def cancel_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Отмена операции"""
    user = update.message.from_user
    if user.id in user_data:
        del user_data[user.id]
    
    await update.message.reply_text(
        'Операция отменена. Чтобы начать заново, напиши /start'
    )
    return ConversationHandler.END

async def handle_other_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обрабатывает другие сообщения"""
    await update.message.reply_text(
        'Отправь фото экологической проблемы или напиши /start чтобы начать.'
    )

def main():
    app = Application.builder().token(BOT_TOKEN).build()
    
    # Создаем обработчик разговора
    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start_command)],
        states={
            PHOTO: [
                MessageHandler(filters.PHOTO, handle_photo),
                MessageHandler(filters.TEXT & ~filters.COMMAND, handle_other_messages)
            ],
            LOCATION: [
                MessageHandler(filters.LOCATION, handle_location),
                MessageHandler(filters.TEXT & ~filters.COMMAND, handle_other_messages)
            ],
        },
        fallbacks=[CommandHandler('cancel', cancel_command)]
    )
    
    app.add_handler(conv_handler)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_other_messages))
    
    print("🟢 Бот-экопатруль запущен!")
    app.run_polling()

if __name__ == '__main__':
    main()
