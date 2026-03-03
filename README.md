import os
import yt_dlp
import instaloader
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, MessageHandler, filters, ContextTypes, CallbackQueryHandler

# BOT TOKENINGIZNI YOZING
TOKEN = "8760722470:AAFttSBMh51shbTGWgzOu_EKUVnmhzi-8O0"

# Qidiruv natijalarini saqlash uchun
search_data = {}

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    chat_id = update.message.chat_id

    # 1. INSTAGRAM LINKI
    if "instagram.com" in text:
        await update.message.reply_text("📸 Instagram video yuklanmoqda...")
        L = instaloader.Instaloader()
        try:
            shortcode = text.split("/")[-2]
            post = instaloader.Post.from_shortcode(L.context, shortcode)
            await context.bot.send_video(chat_id=chat_id, video=post.video_url, caption="Tayyor! ✅")
        except:
            await update.message.reply_text("❌ Instagram videosini olib bo'lmadi.")

    # 2. YOUTUBE LINKI
    elif "http" in text:
        await update.message.reply_text("📥 Video yuklanmoqda...")
        opts = {'format': 'best', 'outtmpl': f'v_{chat_id}.mp4', 'quiet': True}
        try:
            with yt_dlp.YoutubeDL(opts) as ydl:
                ydl.download([text])
            await context.bot.send_video(chat_id=chat_id, video=open(f'v_{chat_id}.mp4', 'rb'))
            os.remove(f'v_{chat_id}.mp4')
        except:
            await update.message.reply_text("❌ YouTube videoni yuklashda xatolik (Blok bo'lishi mumkin).")

    # 3. MUSIQA QIDIRUV (1-10 tugmalar)
    else:
        await update.message.reply_text(f"🔍 '{text}' qidirilmoqda...")
        opts = {'format': 'bestaudio', 'default_search': 'ytsearch10', 'quiet': True}
        try:
            with yt_dlp.YoutubeDL(opts) as ydl:
                info = ydl.extract_info(f"ytsearch10:{text}", download=False)
                entries = info['entries']
                
                res_txt = f"🔎 Natijalar: {text}\n\n"
                buttons = []
                row = []
                
                for i, entry in enumerate(entries, 1):
                    res_txt += f"{i}. {entry['title']} [{entry.get('duration_string', '0:00')}]\n"
                    search_data[f"{chat_id}_{i}"] = entry['url']
                    row.append(InlineKeyboardButton(str(i), callback_data=f"m|{chat_id}_{i}"))
                    if i % 5 == 0:
                        buttons.append(row)
                        row = []
                
                await update.message.reply_text(res_txt, reply_markup=InlineKeyboardMarkup(buttons))
        except:
            await update.message.reply_text("😔 Hech narsa topilmadi.")

async def callback_query(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    if query.data.startswith("m|"):
        key = query.data.split('|')[1]
        url = search_data.get(key)
        if url:
            await query.edit_message_text("🎵 Yuklanmoqda...")
            file = f"{key}.mp3"
            opts = {'format': 'bestaudio', 'outtmpl': file, 'quiet': True}
            with yt_dlp.YoutubeDL(opts) as ydl:
                ydl.download([url])
            await context.bot.send_audio(chat_id=query.message.chat_id, audio=open(file, 'rb'))
            os.remove(file)

if __name__ == "__main__":
    app = Application.builder().token(TOKEN).build()
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.add_handler(CallbackQueryHandler(callback_query))
    app.run_polling()
    
    
