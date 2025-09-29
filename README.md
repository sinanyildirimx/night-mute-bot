# night_mute_bot.py
import logging
import json
from zoneinfo import ZoneInfo
from datetime import time

from telegram import ChatPermissions, Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

# ---------- AYARLAR ----------
BOT_TOKEN = "8427516119:AAHyl7dPcFQ3e3j_lEUIeitmBt-iV1Gt6zA"
TIMEZONE = "Europe/Istanbul"
MUTE_START = time(hour=0, minute=0)   # 00:30
MUTE_END   = time(hour=8, minute=0)   # 08:00
GROUPS_FILE = "groups.json"           # kaydedilen grup id'leri
# -----------------------------

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)

def load_groups():
    try:
        with open(GROUPS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        return []

def save_groups(groups):
    with open(GROUPS_FILE, "w", encoding="utf-8") as f:
        json.dump(groups, f, ensure_ascii=False, indent=2)

async def register(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Grup içinde /register yazınca bu grup id'sini kaydeder."""
    chat = update.effective_chat
    if chat.type not in ("group", "supergroup"):
        await update.message.reply_text("Bu komut sadece grup içinde çalışır. Botu gruba ekleyip tekrar deneyin.")
        return
    groups = load_groups()
    if chat.id in groups:
        await update.message.reply_text("Bu grup zaten kayıtlı.")
    else:
        groups.append(chat.id)
        save_groups(groups)
        await update.message.reply_text("Grup kaydedildi. Bot gece sessiz modu otomatik uygulayacak (00:00 - 08:00).")

async def manual_mute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Admin manuel sessize alma"""
    chat = update.effective_chat
    # sadece grup içinde çalışsın
    if chat.type not in ("group", "supergroup"):
        await update.message.reply_text("Bu komut sadece grup içinde çalışır.")
        return
    try:
        await context.bot.set_chat_permissions(chat.id, ChatPermissions(can_send_messages=False))
        await update.message.reply_text("Grup manuel olarak sessize alındı.")
    except Exception as e:
        await update.message.reply_text(f"Hata: {e}")

async def manual_unmute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat = update.effective_chat
    if chat.type not in ("group", "supergroup"):
        await update.message.reply_text("Bu komut sadece grup içinde çalışır.")
        return
    try:
        # tüm üyeler tekrar mesaj atabilsin
        await context.bot.set_chat_permissions(chat.id, ChatPermissions(can_send_messages=True,
                                                                       can_send_media_messages=True,
                                                                       can_send_polls=True,
                                                                       can_send_other_messages=True,
                                                                       can_add_web_page_previews=True))
        await update.message.reply_text("Grup tekrar aktif hale getirildi (sessiz mod kapandı).")
    except Exception as e:
        await update.message.reply_text(f"Hata: {e}")

async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat = update.effective_chat
    if chat.type not in ("group", "supergroup"):
        await update.message.reply_text("Bu komut sadece grup içinde çalışır.")
        return
    try:
        info = await context.bot.get_chat(chat.id)
        perms = info.permissions
        await update.message.reply_text(f"Mevcut izinler: {perms}")
    except Exception as e:
        await update.message.reply_text(f"Hata: {e}")

async def mute_groups(app):
    tz = ZoneInfo(TIMEZONE)
    groups = load_groups()
    if not groups:
        logger.info("Kayıtlı grup yok, mute yapılmadı.")
    for gid in groups:
        try:
            logger.info(f"Muting group {gid}")
            await app.bot.set_chat_permissions(gid, ChatPermissions(can_send_messages=False))
        except Exception as e:
            logger.error(f"Group {gid} mute hatası: {e}")

async def unmute_groups(app):
    tz = ZoneInfo(TIMEZONE)
    groups = load_groups()
    if not groups:
        logger.info("Kayıtlı grup yok, unmute yapılmadı.")
    for gid in groups:
        try:
            logger.info(f"Unmuting group {gid}")
            await app.bot.set_chat_permissions(gid, ChatPermissions(
                can_send_messages=True,
                can_send_media_messages=True,
                can_send_polls=True,
                can_send_other_messages=True,
                can_add_web_page_previews=True
            ))
        except Exception as e:
            logger.error(f"Group {gid} unmute hatası: {e}")

def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    # Komutlar
    app.add_handler(CommandHandler("register", register))       # grup içinde çalıştır admin olmasına gerek yok (komutu kullanan admin olmalı)
    app.add_handler(CommandHandler("muteban", manual_mute))     # manuel sessize alma
    app.add_handler(CommandHandler("unmuteban", manual_unmute)) # manuel açma
    app.add_handler(CommandHandler("status", status))

    # Scheduler
    scheduler = AsyncIOScheduler(timezone=ZoneInfo(TIMEZONE))
    # Cron ile 00:00'da mute
    scheduler.add_job(lambda: app.create_task(mute_groups(app)),
                      CronTrigger(hour=MUTE_START.hour, minute=MUTE_START.minute))
    # Cron ile 08:00'de unmute
    scheduler.add_job(lambda: app.create_task(unmute_groups(app)),
                      CronTrigger(hour=MUTE_END.hour, minute=MUTE_END.minute))
    scheduler.start()

    logger.info("Bot başlatılıyor...")
    app.run_polling()

if __name__ == "__main__":
    main()
