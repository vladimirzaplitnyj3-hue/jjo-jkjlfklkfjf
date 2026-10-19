import os, io, logging, tempfile, shutil, base64, time, json
import requests
import telebot
from telebot import types
from PIL import Image

from mutagen.id3   import ID3, TIT2, TPE1, APIC, ID3NoHeaderError
from mutagen.mp4   import MP4, MP4Cover
from mutagen.flac  import FLAC, Picture
from mutagen.oggvorbis import OggVorbis
from mutagen.oggopus   import OggOpus
from mutagen.asf       import ASF
from mutagen.wave      import WAVE

TOKEN = "8831747632:AAHQBP07TKxs9jc9Nc8opT5cQqTcurfgeCA"
bot   = telebot.TeleBot(TOKEN, parse_mode="HTML")

logging.basicConfig(level=logging.INFO, format="%(asctime)s | %(levelname)s | %(message)s")
log = logging.getLogger(__name__)

def ce(eid: str, fb: str = "⭐") -> str:
    return f'<tg-emoji emoji-id="{eid}">{fb}</tg-emoji>'

E_MUSIC  = "5429351107576997410"
E_TITLE  = "5343792034736009111"
E_ARTIST = "5343792034736009111"
E_COVER  = "6039802767931871481"
E_SKIP   = "5343669198671342116"
E_SEND   = "5377811534384768829"
E_WAIT   = "5260396302562266399"
E_DONE   = "5260521071362214474"

AUDIO_MIME: dict[str, str] = {
    "audio/mpeg": ".mp3", "audio/mp3": ".mp3",
    "audio/mp4":  ".m4a", "audio/m4a": ".m4a", "audio/x-m4a": ".m4a",
    "video/mp4":  ".mp4",
    "audio/aac":  ".aac",
    "audio/ogg":  ".ogg", "audio/vorbis": ".ogg",
    "audio/opus": ".opus",
    "audio/flac": ".flac", "audio/x-flac": ".flac",
    "audio/wav":  ".wav",  "audio/x-wav": ".wav", "audio/wave": ".wav",
    "audio/wma":  ".wma",  "audio/x-ms-wma": ".wma",
    "audio/webm": ".webm", "video/webm": ".webm",
}
FMT_NAME: dict[str, str] = {
    ".mp3": "MP3", ".m4a": "M4A", ".mp4": "MP4", ".aac": "AAC",
    ".ogg": "OGG", ".opus": "OPUS", ".flac": "FLAC",
    ".wav": "WAV", ".wma": "WMA", ".webm": "WebM",
}

def detect_audio_ext(mime: str, filename: str = "") -> str | None:
    if filename:
        _, ext = os.path.splitext(filename.lower())
        if ext in FMT_NAME:
            return ext
    if mime:
        ml = mime.lower()
        for k, v in AUDIO_MIME.items():
            if ml.startswith(k):
                return v
        if "audio" in ml:
            return ".mp3"
    return None

users: dict = {}

def _blank() -> dict:
    return {
        "step":        "WAIT_AUDIO",
        "audio_path":  None,
        "fmt":         ".mp3",
        "title":       None,
        "artist":      None,
        "cover_path":  None,
        "last_msg_id": None,
    }

def get_user(uid: int) -> dict:
    if uid not in users:
        users[uid] = _blank()
    return users[uid]

def reset(uid: int):
    users[uid] = _blank()

def delete_last(uid: int):
    u   = get_user(uid)
    mid = u.get("last_msg_id")
    if mid:
        try: bot.delete_message(uid, mid)
        except Exception: pass
        u["last_msg_id"] = None

def send_step(uid: int, text: str, markup=None) -> types.Message:
    delete_last(uid)
    msg = bot.send_message(uid, text, reply_markup=markup, parse_mode="HTML")
    get_user(uid)["last_msg_id"] = msg.message_id
    return msg

def skip_kb() -> str:
    return json.dumps({
        "inline_keyboard": [[
            {
                "text": "пропустить",
                "callback_data": "skip",
                "icon_custom_emoji_id": E_SKIP
            }
        ]]
    })

def download_file(file_id: str, suffix: str = ".mp3") -> str:
    info = bot.get_file(file_id)
    url  = f"https://api.telegram.org/file/bot{TOKEN}/{info.file_path}"
    r = requests.get(url, timeout=120)
    r.raise_for_status()
    tmp = tempfile.NamedTemporaryFile(delete=False, suffix=suffix)
    tmp.write(r.content)
    tmp.flush(); tmp.close()
    return tmp.name

def cover_jpeg_bytes(cover_path: str) -> bytes:
    with open(cover_path, "rb") as f:
        raw = f.read()
    img = Image.open(io.BytesIO(raw)).convert("RGB")
    buf = io.BytesIO()
    img.save(buf, format="JPEG", quality=92)
    return buf.getvalue()

def apply_tags(audio_path: str, fmt: str,
               title: str | None, artist: str | None,
               cover_path: str | None) -> str:
    out = audio_path + "_tagged" + fmt
    shutil.copy2(audio_path, out)

    cover = cover_jpeg_bytes(cover_path) if cover_path else None

    if fmt in (".mp3", ".aac"):
        try:    tags = ID3(out)
        except ID3NoHeaderError: tags = ID3()
        if title:  tags["TIT2"] = TIT2(encoding=3, text=title)
        if artist: tags["TPE1"] = TPE1(encoding=3, text=artist)
        if cover:
            tags["APIC"] = APIC(encoding=3, mime="image/jpeg",
                                type=3, desc="Cover", data=cover)
        tags.save(out, v2_version=3)

    elif fmt in (".m4a", ".mp4"):
        tags = MP4(out)
        if title:  tags["\xa9nam"] = [title]
        if artist: tags["\xa9ART"] = [artist]
        if cover:
            tags["covr"] = [MP4Cover(cover, imageformat=MP4Cover.FORMAT_JPEG)]
        tags.save()

    elif fmt == ".flac":
        tags = FLAC(out)
        if title:  tags["title"]  = [title]
        if artist: tags["artist"] = [artist]
        if cover:
            pic = Picture()
            pic.type = 3; pic.mime = "image/jpeg"; pic.data = cover
            tags.clear_pictures()
            tags.add_picture(pic)
        tags.save()

    elif fmt == ".ogg":
        tags = OggVorbis(out)
        if title:  tags["title"]  = [title]
        if artist: tags["artist"] = [artist]
        if cover:
            pic = Picture()
            pic.type = 3; pic.mime = "image/jpeg"; pic.data = cover
            tags["metadata_block_picture"] = [
                base64.b64encode(pic.write()).decode("ascii")
            ]
        tags.save()

    elif fmt == ".opus":
        tags = OggOpus(out)
        if title:  tags["title"]  = [title]
        if artist: tags["artist"] = [artist]
        if cover:
            pic = Picture()
            pic.type = 3; pic.mime = "image/jpeg"; pic.data = cover
            tags["metadata_block_picture"] = [
                base64.b64encode(pic.write()).decode("ascii")
            ]
        tags.save()

    elif fmt == ".wma":
        tags = ASF(out)
        if title:  tags["Title"]  = [title]
        if artist: tags["Author"] = [artist]
        tags.save()

    elif fmt == ".wav":
        tags = WAVE(out)
        if not tags.tags: tags.add_tags()
        if title:  tags.tags["TIT2"] = TIT2(encoding=3, text=title)
        if artist: tags.tags["TPE1"] = TPE1(encoding=3, text=artist)
        if cover:
            tags.tags["APIC"] = APIC(encoding=3, mime="image/jpeg",
                                     type=3, desc="Cover", data=cover)
        tags.save()

    else:
        try:
            try:    tags = ID3(out)
            except ID3NoHeaderError: tags = ID3()
            if title:  tags["TIT2"] = TIT2(encoding=3, text=title)
            if artist: tags["TPE1"] = TPE1(encoding=3, text=artist)
            if cover:
                tags["APIC"] = APIC(encoding=3, mime="image/jpeg",
                                    type=3, desc="Cover", data=cover)
            tags.save(out, v2_version=3)
        except Exception as e:
            log.warning(f"Fallback tag write failed ({fmt}): {e}")

    return out

def ask_title(uid: int):
    send_step(uid,
        f"{ce(E_TITLE, '✏️')} Напишите новое название трека:",
        markup=skip_kb())
    get_user(uid)["step"] = "WAIT_TITLE"

def ask_artist(uid: int):
    send_step(uid,
        f"{ce(E_ARTIST, '🎤')} Напишите нового исполнителя:",
        markup=skip_kb())
    get_user(uid)["step"] = "WAIT_ARTIST"

def ask_cover(uid: int):
    send_step(uid,
        f"{ce(E_COVER, '🖼️')} Пришли новую обложку:",
        markup=skip_kb())
    get_user(uid)["step"] = "WAIT_COVER"

def do_save(uid: int):
    u = get_user(uid)
    delete_last(uid)

    m1 = bot.send_message(uid, f"{ce(E_SEND, '📤')} отправляю файл!", parse_mode="HTML")
    time.sleep(0.6)
    try: bot.delete_message(uid, m1.message_id)
    except Exception: pass

    m2 = bot.send_message(uid, f"{ce(E_WAIT, '⏳')} ожидайте!", parse_mode="HTML")

    try:
        out_path = apply_tags(
            u["audio_path"], u["fmt"],
            u["title"], u["artist"], u["cover_path"]
        )
    except Exception as e:
        log.exception(e)
        bot.edit_message_text(f"❌ Ошибка: {e}", uid, m2.message_id)
        return

    try: bot.delete_message(uid, m2.message_id)
    except Exception: pass

    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("владелец", url="https://t.me/vk_only"))

    fmt  = u["fmt"]
    fname = f"{u['artist'] or 'Unknown'} - {u['title'] or 'Track'}{fmt}"
    try:
        with open(out_path, "rb") as f:
            bot.send_audio(uid, (fname, f),
                           title=u["title"] or None,
                           performer=u["artist"] or None)
        bot.send_message(uid,
            f"готово {ce(E_DONE, '✅')} вот твой трек!",
            reply_markup=kb, parse_mode="HTML")
    except Exception as e:
        log.exception(e)
        bot.send_message(uid, f"❌ Не удалось отправить: {e}")
    finally:
        for p in [u.get("audio_path"), u.get("cover_path"), out_path]:
            if p and os.path.exists(p):
                try: os.remove(p)
                except Exception: pass
        reset(uid)

@bot.message_handler(commands=["start", "help"])
def cmd_start(msg: types.Message):
    uid = msg.from_user.id
    reset(uid)
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton("владелец", url="https://t.me/vk_only"))
    bot.send_message(uid,
        f"{ce(E_MUSIC, '🎵')} Пришли трек, чтобы изменить название, исполнителя или обложку.",
        reply_markup=kb, parse_mode="HTML")

@bot.message_handler(commands=["reset"])
def cmd_reset(msg: types.Message):
    reset(msg.from_user.id)
    bot.send_message(msg.from_user.id, "🔄 Сброшено. Отправь новый трек.")

def _route_document(msg: types.Message):
    uid  = msg.from_user.id
    u    = get_user(uid)
    mime = msg.document.mime_type or ""
    fname = msg.document.file_name or ""

    if u["step"] == "WAIT_COVER" and mime.startswith("image/"):
        _save_cover_from_file_id(uid, u, msg.document.file_id)
        return

    ext = detect_audio_ext(mime, fname)
    if not ext:
        supported = ", ".join(sorted(FMT_NAME.values()))
        bot.send_message(uid,
            f"⚠️ Формат не поддерживается.\n"
            f"Поддерживаю: <b>{supported}</b>",
            parse_mode="HTML")
        return
    _handle_audio_file(uid, u, msg.document.file_id, mime, fname)

def _save_cover_from_file_id(uid: int, u: dict, file_id: str):
    try:
        path = download_file(file_id, suffix=".jpg")
    except Exception as e:
        log.exception(e)
        bot.send_message(uid, "❌ Не удалось скачать обложку.")
        return
    u["cover_path"] = path
    do_save(uid)

def _handle_audio_file(uid: int, u: dict, file_id: str, mime: str, fname: str):
    if u["step"] == "WAIT_COVER":
        bot.send_message(uid, "⚠️ Жду фото-обложку, не аудио. Отправь картинку или нажми «пропустить».")
        return
    ext = detect_audio_ext(mime, fname) or ".mp3"
    wm = bot.send_message(uid,
        f"{ce(E_SEND, '📤')} скачиваю трек...", parse_mode="HTML")
    try:
        path = download_file(file_id, suffix=ext)
    except Exception as e:
        log.exception(e)
        bot.edit_message_text("❌ Не удалось скачать файл.", uid, wm.message_id)
        return
    try: bot.delete_message(uid, wm.message_id)
    except Exception: pass
    reset(uid)
    u = get_user(uid)
    u["audio_path"] = path
    u["fmt"]        = ext
    ask_title(uid)

@bot.message_handler(content_types=["audio"])
def handle_audio(msg: types.Message):
    uid = msg.from_user.id
    u   = get_user(uid)
    _handle_audio_file(uid, u,
        msg.audio.file_id,
        msg.audio.mime_type or "",
        msg.audio.file_name or "")

@bot.message_handler(content_types=["video"])
def handle_video(msg: types.Message):
    uid = msg.from_user.id
    u   = get_user(uid)
    _handle_audio_file(uid, u,
        msg.video.file_id,
        msg.video.mime_type or "video/mp4",
        msg.video.file_name or "video.mp4")

@bot.message_handler(content_types=["document"])
def handle_document(msg: types.Message):
    _route_document(msg)

@bot.message_handler(content_types=["photo"])
def handle_photo(msg: types.Message):
    uid = msg.from_user.id
    u   = get_user(uid)
    if u["step"] != "WAIT_COVER":
        bot.send_message(uid, "⚠️ Сначала отправь аудио-файл.")
        return
    _save_cover_from_file_id(uid, u, msg.photo[-1].file_id)

@bot.message_handler(content_types=["text"])
def handle_text(msg: types.Message):
    uid  = msg.from_user.id
    u    = get_user(uid)
    text = msg.text.strip()
    if text.startswith("/"):
        return
    step = u.get("step")
    if step == "WAIT_TITLE":
        try: bot.delete_message(uid, msg.message_id)
        except Exception: pass
        u["title"] = text
        ask_artist(uid)
    elif step == "WAIT_ARTIST":
        try: bot.delete_message(uid, msg.message_id)
        except Exception: pass
        u["artist"] = text
        ask_cover(uid)
    elif step == "WAIT_COVER":
        bot.send_message(uid, f"{ce(E_COVER, '🖼️')} Жду <b>фото</b>, не текст!",
                         parse_mode="HTML")
    else:
        bot.send_message(uid, "🎵 Сначала отправь аудио-файл!")

@bot.callback_query_handler(func=lambda c: c.data == "skip")
def handle_skip(call: types.CallbackQuery):
    uid = call.from_user.id
    u   = get_user(uid)
    bot.answer_callback_query(call.id)
    step = u.get("step")
    if step == "WAIT_TITLE":
        ask_artist(uid)
    elif step == "WAIT_ARTIST":
        ask_cover(uid)
    elif step == "WAIT_COVER":
        do_save(uid)

if __name__ == "__main__":
    log.info("🎵 Bot started")
    bot.infinity_polling(timeout=60, long_polling_timeout=30)
