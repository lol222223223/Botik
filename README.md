import logging
import html
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application, 
    CommandHandler, 
    CallbackQueryHandler, 
    MessageHandler, 
    filters, 
    CallbackContext
)
import httpx
import json
from typing import Optional, Dict, Any

# Настройка логирования
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Конфигурация
BOT_TOKEN = "7386282153:AAHyO1n7W3Boy8pe58d-zTUEdq0FxDY4RRs"  # Замените на ваш токен
API_BASE_URL = "https://rscripts.net/api/v2"

# Глобальные переменные для кэширования
user_sessions = {}

async def search_scripts(query: str, page: int = 1) -> Optional[Dict[str, Any]]:
    """Поиск скриптов по названию через API"""
    url = f"{API_BASE_URL}/scripts"
    params = {
        "page": page,
        "orderBy": "date",
        "sort": "desc",
        "title": query.strip()
    }
    
    async with httpx.AsyncClient() as client:
        try:
            logger.info(f"Searching for: {query}, page: {page}")
            response = await client.get(url, params=params, timeout=15.0)
            response.raise_for_status()
            return response.json()
        except httpx.TimeoutException:
            logger.error("Request timeout")
            return None
        except Exception as e:
            logger.error(f"API request error: {e}")
            return None

async def get_script_by_id(script_id: str) -> Optional[Dict[str, Any]]:
    """Получение конкретного скрипта по ID"""
    url = f"{API_BASE_URL}/script"
    params = {"id": script_id}
    
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, params=params, timeout=10.0)
            response.raise_for_status()
            data = response.json()
            return data.get("success") if data.get("success") else None
        except Exception as e:
            logger.error(f"Error fetching script {script_id}: {e}")
            return None

async def start(update: Update, context: CallbackContext):
    """Обработчик команды /start"""
    welcome_text = (
        "👋 *Привет! Я бот для поиска скриптов Rscripts*\n\n"
        "📌 *Доступные команды:*\n"
        "/start - Начать работу\n"
        "/search - Поиск скриптов\n"
        "/trending - Популярные скрипты\n"
        "/help - Помощь\n\n"
        "🔍 *Просто отправьте мне название скрипта для поиска!*"
    )
    await update.message.reply_text(welcome_text, parse_mode='Markdown')

async def help_command(update: Update, context: CallbackContext):
    """Обработчик команды /help"""
    help_text = (
        "📖 *Как пользоваться ботом:*\n\n"
        "1. Отправьте название скрипта для поиска\n"
        "2. Используйте кнопки навигации для просмотра результатов\n"
        "3. Нажмите 'Показать код' для просмотра содержимого скрипта\n"
        "4. Используйте 'Ссылка на источник' для перехода на сайт\n\n"
        "📌 *Примеры запросов:*\n"
        "• Auto Clicker\n"
        "• Game Hack\n"
        "• Utility Script\n"
        "• Bot"
    )
    await update.message.reply_text(help_text, parse_mode='Markdown')

async def trending_scripts(update: Update, context: CallbackContext):
    """Получение популярных скриптов"""
    url = f"{API_BASE_URL}/trending"
    
    async with httpx.AsyncClient() as client:
        try:
            await update.message.chat.send_action(action="typing")
            response = await client.get(url, timeout=10.0)
            data = response.json()
            
            if data.get("success"):
                scripts = data["success"][:5]  # Берем первые 5
                
                if not scripts:
                    await update.message.reply_text("Нет популярных скриптов в данный момент.")
                    return
                
                message_text = "🔥 *Популярные скрипты:*\n\n"
                for i, script in enumerate(scripts, 1):
                    title = script.get("title", "Без названия")
                    description = script.get("description", "Нет описания")[:100]
                    message_text += f"{i}. *{title}*\n{description}...\n\n"
                
                await update.message.reply_text(message_text, parse_mode='Markdown')
                
                # Сохраняем первый скрипт для детального просмотра
                if scripts:
                    user_id = update.effective_user.id
                    user_sessions[user_id] = {
                        "scripts": scripts,
                        "current_index": 0,
                        "query": "trending"
                    }
                    
                    keyboard = [
                        [InlineKeyboardButton("📄 Показать первый скрипт", 
                          callback_data=f"view_0_trending")]
                    ]
                    reply_markup = InlineKeyboardMarkup(keyboard)
                    await update.message.reply_text(
                        "Хотите посмотреть детали первого скрипта?",
                        reply_markup=reply_markup
                    )
            else:
                await update.message.reply_text("Не удалось загрузить популярные скрипты.")
                
        except Exception as e:
            logger.error(f"Trending error: {e}")
            await update.message.reply_text("Ошибка при загрузке популярных скриптов.")

async def handle_search_command(update: Update, context: CallbackContext):
    """Обработчик команды /search"""
    await update.message.reply_text(
        "🔍 *Режим поиска*\n\n"
        "Отправьте мне название скрипта, который хотите найти.\n"
        "Например: *Auto Clicker* или *Game Hack*",
        parse_mode='Markdown'
    )

async def handle_text_message(update: Update, context: CallbackContext):
    """Обработчик текстовых сообщений (поиск)"""
    user_query = update.message.text.strip()
    
    if not user_query or len(user_query) < 2:
        await update.message.reply_text("Пожалуйста, введите запрос длиннее 2 символов.")
        return
    
    # Показываем статус "печатает"
    await update.message.chat.send_action(action="typing")
    
    # Ищем скрипты
    search_result = await search_scripts(user_query)
    
    if not search_result:
        await update.message.reply_text(
            "❌ Не удалось подключиться к API. Попробуйте позже."
        )
        return
    
    if "scripts" not in search_result:
        await update.message.reply_text("😔 По вашему запросу ничего не найдено.")
        return
    
    scripts = search_result["scripts"]
    
    if not scripts:
        await update.message.reply_text("😔 Скрипты не найдены. Попробуйте другой запрос.")
        return
    
    # Сохраняем результаты в сессию пользователя
    user_id = update.effective_user.id
    user_sessions[user_id] = {
        "scripts": scripts,
        "current_index": 0,
        "query": user_query,
        "page": 1,
        "total_pages": search_result.get("info", {}).get("maxPages", 1)
    }
    
    # Отправляем первый результат
    await send_script_preview(update, context, user_id)

async def send_script_preview(update, context, user_id):
    """Отправка предпросмотра скрипта"""
    if user_id not in user_sessions:
        return
    
    session = user_sessions[user_id]
    scripts = session["scripts"]
    current_index = session["current_index"]
    
    if current_index >= len(scripts):
        return
    
    script = scripts[current_index]
    
    # Формируем сообщение
    title = html.escape(script.get("title", "Без названия"))
    description = html.escape(script.get("description", "Нет описания"))
    creator = html.escape(script.get("creator", "Неизвестен"))
    script_id = script.get("_id", "")
    
    message_text = (
        f"📄 *{title}*\n\n"
        f"📝 *Описание:* {description}\n"
        f"👤 *Автор:* {creator}\n"
        f"🔢 *Просмотры:* {script.get('views', 0)}\n"
        f"⭐ *Рейтинг:* {script.get('rating', 0)}/5\n\n"
        f"Результат {current_index + 1} из {len(scripts)}"
    )
    
    # Создаем клавиатуру
    keyboard = []
    
    # Кнопки навигации
    nav_buttons = []
    if current_index > 0:
        nav_buttons.append(InlineKeyboardButton("◀️ Назад", callback_data=f"prev_{user_id}"))
    
    nav_buttons.append(InlineKeyboardButton(
        f"{current_index + 1}/{len(scripts)}", 
        callback_data="page_info"
    ))
    
    if current_index < len(scripts) - 1:
        nav_buttons.append(InlineKeyboardButton("Вперед ▶️", callback_data=f"next_{user_id}"))
    
    if nav_buttons:
        keyboard.append(nav_buttons)
    
    # Основные кнопки
    keyboard.append([
        InlineKeyboardButton("📋 Показать код", callback_data=f"code_{script_id}"),
        InlineKeyboardButton("🌐 Ссылка на источник", url=script.get("url", "https://rscripts.net"))
    ])
    
    # Кнопка загрузки
    if script.get("rawScript"):
        keyboard.append([
            InlineKeyboardButton("⬇️ Скачать скрипт", url=script.get("rawScript"))
        ])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    # Отправляем или обновляем сообщение
    if update.callback_query:
        await update.callback_query.edit_message_text(
            text=message_text,
            parse_mode='Markdown',
            reply_markup=reply_markup
        )
        await update.callback_query.answer()
    else:
        await update.message.reply_text(
            message_text,
            parse_mode='Markdown',
            reply_markup=reply_markup
        )

async def button_callback(update: Update, context: CallbackContext):
    """Обработчик нажатий на кнопки"""
    query = update.callback_query
    await query.answer()
    
    data = query.data
    user_id = query.from_user.id
    
    # Обработка навигации
    if data.startswith("prev_") or data.startswith("next_"):
        if user_id not in user_sessions:
            await query.message.reply_text("Сессия истекла. Начните поиск заново.")
            return
        
        session = user_sessions[user_id]
        
        if data.startswith("prev_"):
            if session["current_index"] > 0:
                session["current_index"] -= 1
        else:  # next
            if session["current_index"] < len(session["scripts"]) - 1:
                session["current_index"] += 1
        
        await send_script_preview(update, context, user_id)
    
    # Показать код скрипта
    elif data.startswith("code_"):
        script_id = data.replace("code_", "")
        await show_script_code(update, context, script_id)
    
    # Просмотр скрипта из trending
    elif data.startswith("view_"):
        parts = data.split("_")
        if len(parts) == 3 and parts[2] == "trending":
            index = int(parts[1])
            if user_id in user_sessions:
                user_sessions[user_id]["current_index"] = index
                await send_script_preview(update, context, user_id)

async def show_script_code(update: Update, context: CallbackContext, script_id: str):
    """Показать код скрипта"""
    query = update.callback_query
    await query.answer()
    
    # Получаем скрипт
    script = await get_script_by_id(script_id)
    
    if not script:
        await query.message.reply_text("Не удалось загрузить код скрипта.")
        return
    
    # Получаем raw скрипт
    raw_url = script.get("rawScript")
    if not raw_url:
        await query.message.reply_text("Ссылка на код не найдена.")
        return
    
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(raw_url, timeout=10.0)
            script_content = response.text[:3000]  # Ограничиваем длину
            
            if len(response.text) > 3000:
                script_content += "\n\n... (код обрезан, полная версия по ссылке выше)"
            
            # Экранируем специальные символы для Markdown
            script_content = html.escape(script_content)
            
            message_text = (
                f"📄 *Код скрипта:* {html.escape(script.get('title', 'Без названия'))}\n\n"
                f"```\n{script_content}\n```"
            )
            
            keyboard = [[
                InlineKeyboardButton("🌐 Открыть на сайте", url=script.get("url", "https://rscripts.net"))
            ]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.message.reply_text(
                message_text,
                parse_mode='MarkdownV2',
                reply_markup=reply_markup
            )
            
        except Exception as e:
            logger.error(f"Error fetching script code: {e}")
            await query.message.reply_text("Ошибка при загрузке кода скрипта.")

async def error_handler(update: Update, context: CallbackContext):
    """Обработчик ошибок"""
    logger.error(f"Update {update} caused error {context.error}")
    
    if update and update.effective_message:
        error_message = "⚠️ Произошла ошибка. Попробуйте еще раз."
        await update.effective_message.reply_text(error_message)

def main():
    """Основная функция запуска бота"""
    # Создаем приложение
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Добавляем обработчики команд
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CommandHandler("search", handle_search_command))
    application.add_handler(CommandHandler("trending", trending_scripts))
    
    # Обработчик текстовых сообщений (поиск)
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text_message))
    
    # Обработчик кнопок
    application.add_handler(CallbackQueryHandler(button_callback))
    
    # Обработчик ошибок
    application.add_error_handler(error_handler)
    
    # Запуск бота
    print("Бот запущен...")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
