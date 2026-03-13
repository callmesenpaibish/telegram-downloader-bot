# Ultimate Telegram Multi‑Platform Downloader Bot
# Supports: YouTube, Instagram, TikTok, Reddit + 1000s more via yt‑dlp
# Features:
# - Automatic platform detection
# - Video / Audio selection
# - Anti‑spam protection
# - Admin commands
# - User statistics (SQLite)
# - Parallel downloads
# - Auto file cleanup
# - Ready for free hosting

import os
import re
import sqlite3
import asyncio
import yt_dlp
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

BOT_TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))

DOWNLOAD_FOLDER = "downloads"
os.makedirs(DOWNLOAD_FOLDER, exist_ok=True)

# ---------------- DATABASE ----------------

conn = sqlite3.connect("bot.db", check_same_thread=False)
cur = conn.cursor()

cur.execute(
    """
CREATE TABLE IF NOT EXISTS users(
    user_id INTEGER PRIMARY KEY,
    downloads INTEGER DEFAULT 0
)
"""
)

conn.commit()

# ---------------- RATE LIMIT ----------------

last_request = {}
COOLDOWN = 5

# ---------------- HELPERS ----------------

def register_user(user_id):
    cur.execute("INSERT OR IGNORE INTO users(user_id) VALUES(?)", (user_id,))
    conn.commit()


def increment_download(user_id):
    cur.execute("UPDATE users SET downloads = downloads + 1 WHERE user_id=?", (user_id,))
    conn.commit()


def total_users():
    cur.execute("SELECT COUNT(*) FROM users")
    return cur.fetchone()[0]


async def download_media(url, audio=False):

    loop = asyncio.get_event_loop()

    def run():

        ydl_opts = {
            "outtmpl": f"{DOWNLOAD_FOLDER}/%(title)s.%(ext)s",
            "noplaylist": True,
        }

        if audio:
            ydl_opts["format"] = "bestaudio"
        else:
            ydl_opts["format"] = "best"

        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            return ydl.prepare_filename(info)

    return await loop.run_in_executor(None, run)


# ---------------- COMMANDS ----------------

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user = update.effective_user.id
    register_user(user)

    await update.message.reply_text(
        "Send a video link (YouTube / Instagram / TikTok / Reddit).\n"
        "I'll download it for you."
    )


async def stats(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if update.effective_user.id != ADMIN_ID:
        return

    users = total_users()

    await update.message.reply_text(f"Users: {users}")


# ---------------- LINK HANDLER ----------------

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user = update.effective_user.id
    register_user(user)

    now = datetime.now().timestamp()

    if user in last_request and now - last_request[user] < COOLDOWN:
        await update.message.reply_text("Slow down.")
        return

    last_request[user] = now

    text = update.message.text

    url_pattern = r"https?://[^\\s]+"
    match = re.search(url_pattern, text)

    if not match:
        return

    url = match.group()

    keyboard = [
        [
            InlineKeyboardButton("📹 Video", callback_data=f"video|{url}"),
            InlineKeyboardButton("🎵 Audio", callback_data=f"audio|{url}"),
        ]
    ]

    await update.message.reply_text(
        "Choose format:", reply_markup=InlineKeyboardMarkup(keyboard)
    )


# ---------------- BUTTON HANDLER ----------------

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query
    await query.answer()

    mode, url = query.data.split("|", 1)

    await query.edit_message_text("Downloading...")

    try:

        audio = mode == "audio"

        file_path = await download_media(url, audio)

        with open(file_path, "rb") as f:

            if audio:
                await query.message.reply_audio(f)
            else:
                await query.message.reply_video(f)

        os.remove(file_path)

        increment_download(query.from_user.id)

    except Exception as e:

        await query.message.reply_text("Download failed.")
        print(e)


# ---------------- MAIN ----------------

app = ApplicationBuilder().token(BOT_TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("stats", stats))

app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_message))

app.add_handler(CallbackQueryHandler(button))

print("Bot running...")

app.run_polling()
