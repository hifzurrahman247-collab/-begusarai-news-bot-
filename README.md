import os
import asyncio
import logging
import requests
from bs4 import BeautifulSoup
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

# ========== CONFIGURATION ==========
# Token from environment variable (Render pe set karein)
BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "7629889769:AAFhX9vAW854g7ClkFSb1arJb4KD9cOQbj0")

# Store subscribed users (RAM mein - restart pe reset hoga)
subscribed_users = set()

# Store latest news cache
news_cache = []
last_update_time = None

# ========== LOGGING ==========
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# ========== NEWS FETCHING ==========
def fetch_news_from_web():
    """Fetch news from Begusarai news websites"""
    news_items = []

    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
    }

    # Try Live Hindustan Begusarai
    try:
        response = requests.get("https://www.livehindustan.com/bihar/begusarai", 
                               headers=headers, timeout=15)
        if response.status_code == 200:
            soup = BeautifulSoup(response.content, 'html.parser')
            articles = soup.find_all(['a', 'h2', 'h3'], href=True)

            for article in articles[:15]:
                text = article.get_text(strip=True)
                href = article.get('href', '')
                if text and len(text) > 15 and len(text) < 200:
                    if not href.startswith('http'):
                        href = 'https://www.livehindustan.com' + href
                    news_items.append({
                        'title': text,
                        'link': href,
                        'source': 'Live Hindustan',
                        'time': datetime.now().strftime('%H:%M')
                    })
    except Exception as e:
        logger.error(f"Error fetching Hindustan: {e}")

    # Try Dainik Jagran
    try:
        response = requests.get("https://www.jagran.com/bihar/begusarai-news-hindi.html", 
                               headers=headers, timeout=15)
        if response.status_code == 200:
            soup = BeautifulSoup(response.content, 'html.parser')
            articles = soup.find_all(['a', 'h2', 'h3'], href=True)

            for article in articles[:10]:
                text = article.get_text(strip=True)
                href = article.get('href', '')
                if text and len(text) > 15 and len(text) < 200:
                    if not href.startswith('http'):
                        href = 'https://www.jagran.com' + href
                    news_items.append({
                        'title': text,
                        'link': href,
                        'source': 'Dainik Jagran',
                        'time': datetime.now().strftime('%H:%M')
                    })
    except Exception as e:
        logger.error(f"Error fetching Jagran: {e}")

    # Remove duplicates
    seen = set()
    unique_news = []
    for item in news_items:
        if item['title'] not in seen:
            seen.add(item['title'])
            unique_news.append(item)

    # Fallback
    if not unique_news:
        unique_news = [
            {'title': '📰 Begusarai Latest News - Live Hindustan', 'link': 'https://www.livehindustan.com/bihar/begusarai', 'source': 'Live Hindustan', 'time': datetime.now().strftime('%H:%M')},
            {'title': '📰 Begusarai News - Dainik Jagran', 'link': 'https://www.jagran.com/bihar/begusarai-news-hindi.html', 'source': 'Dainik Jagran', 'time': datetime.now().strftime('%H:%M')},
            {'title': '📰 Begusarai News - Prabhat Khabar', 'link': 'https://www.prabhatkhabar.com/state/bihar/begusarai', 'source': 'Prabhat Khabar', 'time': datetime.now().strftime('%H:%M')},
            {'title': '📰 Bihar News - News18 Hindi', 'link': 'https://hindi.news18.com/news/bihar/', 'source': 'News18', 'time': datetime.now().strftime('%H:%M')},
            {'title': '📰 Aaj Tak Bihar News', 'link': 'https://www.aajtak.in/bihar', 'source': 'Aaj Tak', 'time': datetime.now().strftime('%H:%M')}
        ]

    return unique_news[:15]

def get_begusarai_news():
    """Get latest Begusarai news with caching"""
    global news_cache, last_update_time

    if last_update_time is None or (datetime.now() - last_update_time).total_seconds() > 1800:
        news_cache = fetch_news_from_web()
        last_update_time = datetime.now()

    return news_cache

# ========== BOT COMMANDS ==========
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    welcome_text = """
🙏 *Namaste! Main aapka Begusarai News Bot hoon!* 🇮🇳

📍 *Bihar ke Begusarai jile ki 24/7 news aapke liye!*

*Commands:*
📰 /news - Latest Begusarai news dekhein
🔔 /subscribe - Daily news alerts ke liye subscribe karein
❌ /unsubscribe - Alerts band karein
📊 /categories - News categories dekhein
ℹ️ /about - Bot ke baare mein jaanein
❓ /help - Help message

*Aapko kya dekhna hai?* 👇
"""
    keyboard = [
        [InlineKeyboardButton("📰 Latest News", callback_data='latest_news')],
        [InlineKeyboardButton("🔔 Subscribe for Alerts", callback_data='subscribe')],
        [InlineKeyboardButton("📊 News Categories", callback_data='categories')],
        [InlineKeyboardButton("🌐 News Websites", callback_data='websites')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(welcome_text, parse_mode='Markdown', reply_markup=reply_markup)

async def news_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    processing_msg = await update.message.reply_text("⏳ *Begusarai ki latest news fetch ho rahi hai...*", parse_mode='Markdown')

    news_items = get_begusarai_news()

    if not news_items:
        await processing_msg.edit_text("😔 Abhi koi news available nahi hai. Kuch der baad try karein.")
        return

    message = "📰 *BEGUSARAI LATEST NEWS* 📰

"
    message += f"🕐 _Updated: {datetime.now().strftime('%d %b %Y, %I:%M %p')}_
"
    message += "━" * 20 + "

"

    for i, item in enumerate(news_items[:8], 1):
        message += f"{i}. *{item['title']}*
"
        message += f"   📎 [Read More]({item['link']}) | 🏢 {item['source']}

"

    message += "━" * 20 + "
"
    message += "💡 /news se aur news paayein!"

    keyboard = [
        [InlineKeyboardButton("🔄 Refresh News", callback_data='latest_news')],
        [InlineKeyboardButton("🔔 Subscribe for Alerts", callback_data='subscribe')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)

    await processing_msg.edit_text(message, parse_mode='Markdown', 
                                   reply_markup=reply_markup, disable_web_page_preview=True)

async def subscribe_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    if user_id in subscribed_users:
        await update.message.reply_text("✅ *Aap pehle se subscribed hain!*

Aapko har 2 ghante mein latest news milti rahegi.", parse_mode='Markdown')
    else:
        subscribed_users.add(user_id)
        await update.message.reply_text(
            "🎉 *Subscription Successful!*

"
            "✅ Aap ab Begusarai news alerts ke liye subscribed ho gaye hain!

"
            "📢 Aapko har 2 ghante mein latest news automatically bheji jayegi.
"
            "🔕 Unsubscribe karne ke liye /unsubscribe type karein.",
            parse_mode='Markdown'
        )

async def unsubscribe_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    if user_id in subscribed_users:
        subscribed_users.remove(user_id)
        await update.message.reply_text(
            "❌ *Unsubscribed!*

"
            "Aap ab news alerts nahi paayenge.
"
            "Dobara subscribe karne ke liye /subscribe type karein.",
            parse_mode='Markdown'
        )
    else:
        await update.message.reply_text("⚠️ Aap pehle se subscribed nahi hain.")

async def categories_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    categories_text = """
📊 *BEGUSARAI NEWS CATEGORIES* 📊

🏛️ *Politics* - Local politics & elections
🚔 *Crime* - Police & crime reports
📚 *Education* - Schools & colleges
🏥 *Health* - Hospitals & health news
🌾 *Agriculture* - Farming & crops
🏗️ *Development* - Roads & infrastructure
🎉 *Culture* - Festivals & events
💼 *Jobs* - Employment news

*News websites par jaane ke liye neeche buttons dabayein:*
"""
    keyboard = [
        [InlineKeyboardButton("🏛️ Politics", url='https://www.livehindustan.com/bihar/begusarai'),
         InlineKeyboardButton("🚔 Crime", url='https://www.jagran.com/bihar/begusarai-news-hindi.html')],
        [InlineKeyboardButton("📚 Education", url='https://www.prabhatkhabar.com/state/bihar/begusarai'),
         InlineKeyboardButton("🏥 Health", url='https://www.livehindustan.com/bihar/begusarai')],
        [InlineKeyboardButton("🌾 Agriculture", url='https://www.jagran.com/bihar/begusarai-news-hindi.html'),
         InlineKeyboardButton("🏗️ Development", url='https://www.prabhatkhabar.com/state/bihar/begusarai')],
        [InlineKeyboardButton("📰 All News", callback_data='latest_news')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(categories_text, parse_mode='Markdown', reply_markup=reply_markup)

async def about_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    about_text = """
ℹ️ *ABOUT BEGUSARAI NEWS BOT* ℹ️

🤖 *Bot Name:* @Boardscrackerbot
📍 *Location:* Begusarai, Bihar, India
🎯 *Purpose:* 24/7 local news delivery

*Features:*
✅ Real-time news updates
✅ Multiple news sources
✅ Auto news alerts (every 2 hours)
✅ Easy category browsing
✅ Hindi & English news

*News Sources:*
📰 Live Hindustan
📰 Dainik Jagran
📰 Prabhat Khabar
📰 News18 Bihar

*Created with ❤️ for Begusarai*
"""
    await update.message.reply_text(about_text, parse_mode='Markdown')

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = """
❓ *HELP & COMMANDS* ❓

*/start* - Bot start karein
*/news* - Latest news dekhein
*/subscribe* - Alerts ke liye subscribe
*/unsubscribe* - Alerts band karein
*/categories* - News categories
*/about* - Bot ke baare mein
*/help* - Yeh help message

*Buttons ka use kaise karein:*
👆 Button press karke direct news website par ja sakte hain.

*Koi problem hai?*
Bot se related koi bhi sawaal ho toh pooch sakte hain!
"""
    await update.message.reply_text(help_text, parse_mode='Markdown')

# ========== CALLBACK HANDLERS ==========
async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == 'latest_news':
        news_items = get_begusarai_news()

        message = "📰 *BEGUSARAI LATEST NEWS* 📰

"
        message += f"🕐 _Updated: {datetime.now().strftime('%d %b %Y, %I:%M %p')}_
"
        message += "━" * 20 + "

"

        for i, item in enumerate(news_items[:8], 1):
            message += f"{i}. *{item['title']}*
"
            message += f"   📎 [Read More]({item['link']}) | 🏢 {item['source']}

"

        message += "━" * 20 + "
"

        keyboard = [
            [InlineKeyboardButton("🔄 Refresh", callback_data='latest_news')],
            [InlineKeyboardButton("🔔 Subscribe", callback_data='subscribe')]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)

        try:
            await query.edit_message_text(message, parse_mode='Markdown', 
                                         reply_markup=reply_markup, disable_web_page_preview=True)
        except:
            await context.bot.send_message(chat_id=update.effective_chat.id, text=message, 
                                          parse_mode='Markdown', reply_markup=reply_markup, 
                                          disable_web_page_preview=True)

    elif query.data == 'subscribe':
        user_id = update.effective_user.id
        if user_id in subscribed_users:
            await query.edit_message_text("✅ *Aap pehle se subscribed hain!*

Aapko har 2 ghante mein latest news milti rahegi.", parse_mode='Markdown')
        else:
            subscribed_users.add(user_id)
            await query.edit_message_text(
                "🎉 *Subscription Successful!*

"
                "✅ Aap ab har 2 ghante mein latest Begusarai news paayenge!

"
                "🔕 Unsubscribe karne ke liye /unsubscribe type karein.",
                parse_mode='Markdown'
            )

    elif query.data == 'categories':
        categories_text = """
📊 *NEWS CATEGORIES* 📊

News websites par jaane ke liye neeche buttons dabayein:
"""
        keyboard = [
            [InlineKeyboardButton("📰 Live Hindustan", url='https://www.livehindustan.com/bihar/begusarai')],
            [InlineKeyboardButton("📰 Dainik Jagran", url='https://www.jagran.com/bihar/begusarai-news-hindi.html')],
            [InlineKeyboardButton("📰 Prabhat Khabar", url='https://www.prabhatkhabar.com/state/bihar/begusarai')],
            [InlineKeyboardButton("📰 News18 Bihar", url='https://hindi.news18.com/news/bihar/')],
            [InlineKeyboardButton("🔙 Back", callback_data='back_to_start')]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        try:
            await query.edit_message_text(categories_text, parse_mode='Markdown', reply_markup=reply_markup)
        except:
            await context.bot.send_message(chat_id=update.effective_chat.id, text=categories_text, 
                                          parse_mode='Markdown', reply_markup=reply_markup)

    elif query.data == 'websites':
        websites_text = """
🌐 *BEGUSARAI NEWS WEBSITES* 🌐

*Official News Portals:*
"""
        keyboard = [
            [InlineKeyboardButton("📰 Live Hindustan - Begusarai", url='https://www.livehindustan.com/bihar/begusarai')],
            [InlineKeyboardButton("📰 Dainik Jagran - Begusarai", url='https://www.jagran.com/bihar/begusarai-news-hindi.html')],
            [InlineKeyboardButton("📰 Prabhat Khabar - Begusarai", url='https://www.prabhatkhabar.com/state/bihar/begusarai')],
            [InlineKeyboardButton("📰 News18 Bihar", url='https://hindi.news18.com/news/bihar/')],
            [InlineKeyboardButton("📰 Aaj Tak Bihar", url='https://www.aajtak.in/bihar')],
            [InlineKeyboardButton("🔙 Back", callback_data='back_to_start')]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        try:
            await query.edit_message_text(websites_text, parse_mode='Markdown', reply_markup=reply_markup)
        except:
            await context.bot.send_message(chat_id=update.effective_chat.id, text=websites_text, 
                                          parse_mode='Markdown', reply_markup=reply_markup)

    elif query.data == 'back_to_start':
        welcome_text = """
🙏 *Welcome back!* 🇮🇳

*Kya dekhna chahate hain?* 👇
"""
        keyboard = [
            [InlineKeyboardButton("📰 Latest News", callback_data='latest_news')],
            [InlineKeyboardButton("🔔 Subscribe", callback_data='subscribe')],
            [InlineKeyboardButton("📊 Categories", callback_data='categories')],
            [InlineKeyboardButton("🌐 News Websites", callback_data='websites')]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        try:
            await query.edit_message_text(welcome_text, parse_mode='Markdown', reply_markup=reply_markup)
        except:
            await context.bot.send_message(chat_id=update.effective_chat.id, text=welcome_text, 
                                          parse_mode='Markdown', reply_markup=reply_markup)

# ========== SCHEDULED JOBS ==========
async def send_scheduled_news(context: ContextTypes.DEFAULT_TYPE):
    """Send news to all subscribed users every 2 hours"""
    if not subscribed_users:
        return

    news_items = get_begusarai_news()

    if not news_items:
        return

    message = "🔔 *BEGUSARAI NEWS ALERT* 🔔

"
    message += f"🕐 _{datetime.now().strftime('%d %b %Y, %I:%M %p')}_
"
    message += "━" * 20 + "

"

    for i, item in enumerate(news_items[:5], 1):
        message += f"{i}. *{item['title']}*
"
        message += f"   📎 [Read More]({item['link']})

"

    message += "━" * 20 + "
"
    message += "💡 /news se aur news paayein!"

    for user_id in list(subscribed_users):
        try:
            await context.bot.send_message(
                chat_id=user_id,
                text=message,
                parse_mode='Markdown',
                disable_web_page_preview=True
            )
        except Exception as e:
            logger.error(f"Failed to send message to {user_id}: {e}")
            if "blocked" in str(e).lower() or "not found" in str(e).lower():
                subscribed_users.discard(user_id)

# ========== MAIN FUNCTION ==========
def main():
    application = Application.builder().token(BOT_TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("news", news_command))
    application.add_handler(CommandHandler("subscribe", subscribe_command))
    application.add_handler(CommandHandler("unsubscribe", unsubscribe_command))
    application.add_handler(CommandHandler("categories", categories_command))
    application.add_handler(CommandHandler("about", about_command))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CallbackQueryHandler(button_callback))

    job_queue = application.job_queue
    job_queue.run_repeating(send_scheduled_news, interval=7200, first=10)

    print("🚀 Begusarai News Bot (@Boardscrackerbot) is running...")
    print("📰 Fetching news every 2 hours for subscribed users")
    print("💡 Press Ctrl+C to stop")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
    
