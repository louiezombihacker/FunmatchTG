import json
import os
import shutil
from typing import Dict, Any, List, Tuple, Optional

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, InputMediaPhoto, ReplyKeyboardRemove
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, ContextTypes, filters

DATA_FILE = "profiles.json"
BACKUP_FILE = "profiles_backup.json"
CURRENT_VERSION = 1

PRIVACY_POLICY = """FunMatch Bot – Privacy Policy

Effective Date: [Insert Date]

By using FunMatch (“the Bot”), you agree to this Privacy Policy. This explains how your information is collected, stored, and used when you interact with the Bot on Telegram.

1. Information We Collect
We store your Telegram user ID, profile details (name, bio, photo), hearts you give (for notification only), and visibility status (sleep mode).

2. How We Use Your Information
Your data is used to create/manage your profile, match you with others, notify hearts (notifications do not increase the public heart count), and improve UX. We will never sell your data.

3. Data Storage
Data is stored in profiles.json with a backup profiles_backup.json.

4. Data Control & Deletion
You may edit or delete your profile at any time. Deleting is permanent.

5. Security
We take reasonable measures but no system is 100% secure.

6. Consent
By creating a profile and using FunMatch, you consent to this Privacy Policy.
"""

ADMIN_CONTACT = "bot_admin_here"

CREATE_NAME, CREATE_BIO, CREATE_PHOTO = range(3)
EDIT_NAME, EDIT_BIO, EDIT_PHOTO = range(3, 6)
POLICY_AGREE = 6

def _ensure_schema(d: Dict[str, Any]) -> Dict[str, Any]:
    d.setdefault("version", CURRENT_VERSION)
    d.setdefault("profiles", {})
    d.setdefault("hearts", {})
    d.setdefault("given", {})
    return d

def normalize_db(data: Dict[str, Any]) -> Dict[str, Any]:
    data = _ensure_schema(data)
    profiles = data.get("profiles", {}) or {}
    new_profiles: Dict[str, Dict[str, Any]] = {}
    for k, v in profiles.items():
        sid = str(k)
        if not isinstance(v, dict):
            v = {}
        v.setdefault("name", None)
        v.setdefault("bio", None)
        v.setdefault("photo", None)
        v.setdefault("sleep", False)
        v.setdefault("agreed", False)
        new_profiles[sid] = v
    data["profiles"] = new_profiles
    hearts = data.get("hearts", {}) or {}
    new_hearts: Dict[str, int] = {}
    for k, v in hearts.items():
        try:
            new_hearts[str(k)] = int(v)
        except Exception:
            new_hearts[str(k)] = 0
    data["hearts"] = new_hearts
    given = data.get("given", {}) or {}
    new_given: Dict[str, List[str]] = {}
    for k, lst in given.items():
        try:
            new_given[str(k)] = [str(x) for x in lst] if isinstance(lst, list) else []
        except Exception:
            new_given[str(k)] = []
    data["given"] = new_given
    data.setdefault("version", CURRENT_VERSION)
    return data

def save_db(data: Dict[str, Any]):
    try:
        if os.path.exists(DATA_FILE):
            shutil.copy(DATA_FILE, BACKUP_FILE)
    except Exception:
        pass
    tmp = DATA_FILE + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)
    os.replace(tmp, DATA_FILE)

def overwrite_backup_with_current_db():
    try:
        with open(BACKUP_FILE, "w", encoding="utf-8") as f:
            json.dump(db, f, indent=2, ensure_ascii=False)
    except Exception:
        pass

def load_db() -> Dict[str, Any]:
    data: Dict[str, Any] = {}
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
        except Exception:
            if os.path.exists(BACKUP_FILE):
                try:
                    with open(BACKUP_FILE, "r", encoding="utf-8") as f:
                        data = json.load(f)
                    save_db(data)
                except Exception:
                    data = {}
            else:
                data = {}
    else:
        if os.path.exists(BACKUP_FILE):
            try:
                with open(BACKUP_FILE, "r", encoding="utf-8") as f:
                    data = json.load(f)
                save_db(data)
            except Exception:
                data = {}
        else:
            data = {}
    data = normalize_db(data)
    save_db(data)
    return data

db = load_db()

def ensure_profile_slot(user_id: str):
    user_id = str(user_id)
    if user_id not in db["profiles"]:
        db["profiles"][user_id] = {"name": None, "bio": None, "photo": None, "sleep": False, "agreed": False}
    db["hearts"].setdefault(user_id, 0)
    db["given"].setdefault(user_id, [])

def has_profile(user_id: str) -> bool:
    user_id = str(user_id)
    return user_id in db["profiles"] and all(db["profiles"][user_id].get(k) for k in ("name", "bio", "photo"))

def browse_candidates(viewer_id: str) -> List[str]:
    viewer_id = str(viewer_id)
    given = set(db.get("given", {}).get(viewer_id, []))
    out: List[str] = []
    for uid, p in db["profiles"].items():
        sid = str(uid)
        if sid == viewer_id:
            continue
        if p.get("sleep"):
            continue
        if not p.get("photo"):
            continue
        if sid in given:
            continue
        out.append(sid)
    return out

def profile_caption(uid: str) -> str:
    p = db["profiles"].get(str(uid), {})
    hearts = int(db["hearts"].get(str(uid), 0))
    return f"👤 {p.get('name','')}\n\n{p.get('bio','')}\n\n💖 Hearts: {hearts}"

def browse_keyboard(viewer_id: str, target_id: str) -> InlineKeyboardMarkup:
    viewer_id = str(viewer_id)
    target_id = str(target_id)
    given = set(db.get("given", {}).get(viewer_id, []))
    if target_id in given:
        heart_button = InlineKeyboardButton("❤️ Hearted", callback_data="hearted")
    else:
        heart_button = InlineKeyboardButton("❤️ Heart", callback_data=f"heart:{target_id}")
    return InlineKeyboardMarkup([
        [heart_button],
        [InlineKeyboardButton("💬 Chat", callback_data="chat")],
        [InlineKeyboardButton("➡ Next Profile", callback_data="next")],
        [InlineKeyboardButton("⬅ Back to Menu", callback_data=f"menu:main:{viewer_id}")],
        [InlineKeyboardButton("💖 My Hearts", callback_data="myhearts")],
    ])

def main_menu_keyboard(user_id: str) -> InlineKeyboardMarkup:
    user_id = str(user_id)
    hearts = int(db["hearts"].get(user_id, 0))
    sleep = db["profiles"].get(user_id, {}).get("sleep", False)
    sleep_label = "😴 Sleep: ON (hidden)" if sleep else "🌞 Sleep: OFF (visible)"
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("✏️ Edit Profile", callback_data=f"menu:edit:{user_id}")],
        [InlineKeyboardButton(f"💖 Total Hearts: {hearts}", callback_data=f"menu:hearts:{user_id}")],
        [InlineKeyboardButton(sleep_label, callback_data=f"menu:sleep:{user_id}")],
        [InlineKeyboardButton("➡ Browse Profiles", callback_data=f"menu:browse:{user_id}")],
        [InlineKeyboardButton("👀 View My Profile", callback_data=f"menu:view:{user_id}")],
        [InlineKeyboardButton("➕ Create New Account", callback_data=f"menu:create:{user_id}")],
        [InlineKeyboardButton("🗑 Delete Account", callback_data=f"menu:delete:{user_id}")],
    ])

async def record_heart(context: ContextTypes.DEFAULT_TYPE, giver: str, target: str) -> bool:
    giver = str(giver)
    target = str(target)
    if target == giver:
        return False
    ensure_profile_slot(giver)
    if target not in db["profiles"] or not all(db["profiles"][target].get(k) for k in ("name", "bio", "photo")):
        return False
    given_list = db["given"].setdefault(giver, [])
    if target in given_list:
        return False
    given_list.append(target)
    save_db(db)
    try:
        hearter_display = None
        try:
            chat_info = await context.bot.get_chat(int(giver))
            hearter_display = chat_info.username or getattr(chat_info, "first_name", None) or str(giver)
        except Exception:
            hearter_display = db["profiles"].get(giver, {}).get("name") or str(giver)
        p = db["profiles"].get(giver, {})
        caption = f"❤️ {hearter_display} hearted your profile!\n\nName: {p.get('name','')}\nBio: {p.get('bio','')}"
        if p.get("photo"):
            try:
                await context.bot.send_photo(chat_id=int(target), photo=p["photo"], caption=caption)
            except Exception:
                try:
                    await context.bot.send_message(chat_id=int(target), text=caption)
                except Exception:
                    pass
        else:
            try:
                await context.bot.send_message(chat_id=int(target), text=caption)
            except Exception:
                pass
    except Exception:
        pass
    return True

async def safe_edit_caption(context: ContextTypes.DEFAULT_TYPE, chat_id: int, message_id: int, caption: str, reply_markup=None) -> bool:
    try:
        await context.bot.edit_message_caption(chat_id=chat_id, message_id=message_id, caption=caption, reply_markup=reply_markup)
        return True
    except Exception:
        try:
            await context.bot.edit_message_text(chat_id=chat_id, message_id=message_id, text=caption, reply_markup=reply_markup)
            return True
        except Exception:
            return False

async def safe_edit_media(context: ContextTypes.DEFAULT_TYPE, chat_id: int, message_id: int, media: InputMediaPhoto, caption: str, reply_markup=None) -> bool:
    try:
        await context.bot.edit_message_media(chat_id=chat_id, message_id=message_id, media=media, reply_markup=reply_markup)
        try:
            await context.bot.edit_message_caption(chat_id=chat_id, message_id=message_id, caption=caption, reply_markup=reply_markup)
        except Exception:
            pass
        return True
    except Exception:
        try:
            await context.bot.edit_message_text(chat_id=chat_id, message_id=message_id, text=caption, reply_markup=reply_markup)
            return True
        except Exception:
            return False

def set_profile_msg_ref(context: ContextTypes.DEFAULT_TYPE, chat_id: int, message_id: int):
    context.user_data["profile_msg_ref"] = (chat_id, message_id)

def get_profile_msg_ref(context: ContextTypes.DEFAULT_TYPE) -> Optional[Tuple[int, int]]:
    return context.user_data.get("profile_msg_ref")

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user is None:
        return
    user_id = str(user.id)
    if has_profile(user_id):
        try:
            if update.message:
                await update.message.reply_text(f"👋 Welcome back, {db['profiles'][user_id]['name']}!", reply_markup=main_menu_keyboard(user_id))
        except Exception:
            pass
        return
    ensure_profile_slot(user_id)
    p = db["profiles"].get(user_id, {})
    if not p.get("agreed"):
        kb = InlineKeyboardMarkup([[InlineKeyboardButton("✅ Agree", callback_data=f"policy:agree:{user_id}"), InlineKeyboardButton("❌ Disagree", callback_data=f"policy:disagree:{user_id}")]])
        try:
            if update.message:
                await update.message.reply_text(PRIVACY_POLICY + "\n\nDo you agree to the FunMatch Privacy Policy?", reply_markup=kb)
            else:
                await context.bot.send_message(chat_id=update.effective_chat.id, text=PRIVACY_POLICY + "\n\nDo you agree to the FunMatch Privacy Policy?", reply_markup=kb)
        except Exception:
            pass
        return
    context.user_data["flow"] = "create_name"
    try:
        await update.message.reply_text("Welcome! Let’s create your profile.\n\nWhat’s your name?")
    except Exception:
        pass

async def callback_router(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = getattr(update, "callback_query", None)
    if query is None:
        return
    await query.answer()
    data = query.data or ""
    user = query.from_user or update.effective_user
    if user is None:
        return
    user_id = str(user.id)

    if data.startswith("policy:"):
        parts = data.split(":")
        if len(parts) < 3 or parts[2] != user_id:
            try:
                await query.answer("You cannot respond to someone else's policy prompt.", show_alert=True)
            except Exception:
                pass
            return
        if parts[1] == "agree":
            db["profiles"].setdefault(user_id, {}).update({"agreed": True})
            save_db(db)
            context.user_data["flow"] = "create_name"
            try:
                await query.edit_message_text("Thank you. Let’s create your profile.\n\nWhat’s your name?")
            except Exception:
                try:
                    await query.message.reply_text("Thank you. Let’s create your profile.\n\nWhat’s your name?")
                except Exception:
                    pass
        else:
            try:
                await query.edit_message_text("You must agree to the Privacy Policy to create a profile.")
            except Exception:
                try:
                    await query.message.reply_text("You must agree to the Privacy Policy to create a profile.")
                except Exception:
                    pass
        return

    if data.startswith("menu:"):
        parts = data.split(":")
        if len(parts) < 3:
            return
        action = parts[1]
        owner = parts[2]
        if owner != user_id:
            try:
                await query.answer("You cannot control another user's menu.", show_alert=True)
            except Exception:
                pass
            return
        if action == "browse":
            context.user_data["browse_index"] = 0
            await show_next_profile(query, context, replace=True)
            return
        if action == "hearts":
            hearts = int(db["hearts"].get(user_id, 0))
            try:
                await query.edit_message_text(f"💖 You have received {hearts} hearts.", reply_markup=main_menu_keyboard(user_id))
            except Exception:
                try:
                    await query.message.reply_text(f"💖 You have received {hearts} hearts.")
                except Exception:
                    pass
            return
        if action == "sleep":
            db["profiles"][user_id]["sleep"] = not db["profiles"][user_id].get("sleep", False)
            save_db(db)
            status = "😴 Sleep mode ON (you are hidden)" if db["profiles"][user_id]["sleep"] else "🌞 Sleep mode OFF (you are visible)"
            try:
                await query.edit_message_text(status, reply_markup=main_menu_keyboard(user_id))
            except Exception:
                try:
                    await query.message.reply_text(status)
                except Exception:
                    pass
            return
        if action == "edit":
            if not has_profile(user_id):
                try:
                    await query.edit_message_text("You don't have a profile to edit. Use /start to create one.")
                except Exception:
                    pass
                return
            p = db["profiles"].get(user_id, {})
            cap = profile_caption(user_id)
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("Edit Name", callback_data=f"edit:name:{user_id}"), InlineKeyboardButton("Edit Bio", callback_data=f"edit:bio:{user_id}")],
                [InlineKeyboardButton("Edit Photo", callback_data=f"edit:photo:{user_id}")],
                [InlineKeyboardButton("⬅ Back to Menu", callback_data=f"menu:main:{user_id}")],
            ])
            try:
                if p.get("photo"):
                    await query.edit_message_media(InputMediaPhoto(media=p["photo"], caption=cap), reply_markup=kb)
                else:
                    await query.edit_message_text(cap, reply_markup=kb)
                set_profile_msg_ref(context, query.message.chat.id, query.message.message_id)
            except Exception:
                try:
                    await query.message.reply_text(cap, reply_markup=kb)
                except Exception:
                    pass
            return
        if action == "view":
            if not has_profile(user_id):
                try:
                    await query.edit_message_text("You don't have a profile. Use /start to create one.")
                except Exception:
                    pass
                return
            p = db["profiles"].get(user_id, {})
            cap = profile_caption(user_id)
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("✏️ Edit Profile", callback_data=f"menu:edit:{user_id}")],
                [InlineKeyboardButton("⬅ Back", callback_data=f"menu:main:{user_id}")],
            ])
            try:
                if p.get("photo"):
                    await query.edit_message_media(InputMediaPhoto(media=p["photo"], caption=cap), reply_markup=kb)
                else:
                    await query.edit_message_text(cap, reply_markup=kb)
                set_profile_msg_ref(context, query.message.chat.id, query.message.message_id)
            except Exception:
                try:
                    await query.message.reply_text(cap, reply_markup=kb)
                except Exception:
                    pass
            return
        if action == "main":
            try:
                await query.edit_message_text("📍 Main Menu", reply_markup=main_menu_keyboard(user_id))
            except Exception:
                try:
                    await query.message.reply_text("📍 Main Menu", reply_markup=main_menu_keyboard(user_id))
                except Exception:
                    pass
            return
        if action == "delete":
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("❗ Confirm Delete (permanent)", callback_data=f"delete:confirm:{user_id}")],
                [InlineKeyboardButton("Cancel", callback_data=f"delete:cancel:{user_id}")],
            ])
            try:
                await query.edit_message_text("Warning: Deleting your account will permanently remove your profile and cannot be undone. Do you want to proceed?", reply_markup=kb)
            except Exception:
                try:
                    await query.message.reply_text("Warning: Deleting your account will permanently remove your profile and cannot be undone. Do you want to proceed?", reply_markup=kb)
                except Exception:
                    pass
            return
        if action == "create":
            p = db["profiles"].get(user_id, {})
            if p.get("name") or p.get("bio") or p.get("photo"):
                kb = InlineKeyboardMarkup([
                    [InlineKeyboardButton("Delete my existing account first", callback_data=f"menu:delete:{user_id}")],
                    [InlineKeyboardButton("Cancel", callback_data=f"menu:main:{user_id}")],
                ])
                try:
                    await query.edit_message_text("You already have an account. To create a brand new account you must delete your current one first.", reply_markup=kb)
                except Exception:
                    try:
                        await query.message.reply_text("You already have an account. To create a brand new account you must delete your current one first.", reply_markup=kb)
                    except Exception:
                        pass
            else:
                try:
                    await query.edit_message_text("To create a new account run /start")
                except Exception:
                    try:
                        await query.message.reply_text("To create a new account run /start")
                    except Exception:
                        pass
            return

    if data.startswith("edit:"):
        parts = data.split(":")
        if len(parts) < 3:
            return
        what = parts[1]
        owner = parts[2]
        if owner != user_id:
            try:
                await query.answer("You cannot edit another user's profile.", show_alert=True)
            except Exception:
                pass
            return
        context.user_data["await_edit_owner"] = owner
        if what == "photo":
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("I understand — proceed", callback_data=f"photo_confirm:proceed:{owner}"), InlineKeyboardButton("Cancel", callback_data=f"photo_confirm:cancel:{owner}")],
            ])
            try:
                await safe_edit_caption(context, query.message.chat.id, query.message.message_id, (query.message.caption or "") + "\n\nWarning: Once you save a new profile photo it will be permanent. To change it later you must delete your account and create a new one.", reply_markup=kb)
            except Exception:
                try:
                    await query.message.reply_text("Warning: Once you save a new profile photo it will be permanent. To change it later you must delete your account and create a new one.", reply_markup=kb)
                except Exception:
                    pass
            context.user_data["await_edit"] = "photo_confirm"
            return
        context.user_data["await_edit"] = what
        prompts = {
            "name": "✏️ Send your new name now.",
            "bio": "✏️ Send your new bio now.",
        }
        caption_prompt = (getattr(query.message, "caption", None) or f"Editing {what}…") + f"\n\n{prompts.get(what, 'Send now.')}"
        try:
            await safe_edit_caption(context, query.message.chat.id, query.message.message_id, caption_prompt)
        except Exception:
            try:
                await query.message.reply_text(caption_prompt)
            except Exception:
                pass
        return

    if data.startswith("photo_confirm:"):
        parts = data.split(":")
        if len(parts) < 3:
            return
        action = parts[1]
        owner = parts[2]
        if owner != user_id:
            try:
                await query.answer("You cannot confirm photo edits for another user.", show_alert=True)
            except Exception:
                pass
            return
        if action == "proceed":
            context.user_data["await_edit"] = "photo"
            context.user_data["await_edit_owner"] = owner
            try:
                await safe_edit_caption(context, query.message.chat.id, query.message.message_id, "Send your new photo now.")
            except Exception:
                try:
                    await query.message.reply_text("Send your new photo now.")
                except Exception:
                    pass
        else:
            try:
                await query.edit_message_text("Edit photo cancelled.")
            except Exception:
                try:
                    await query.message.reply_text("Edit photo cancelled.")
                except Exception:
                    pass
            context.user_data.pop("await_edit", None)
            context.user_data.pop("await_edit_owner", None)
        return

    if data.startswith("confirm:"):
        parts = data.split(":")
        if len(parts) < 4:
            return
        action = parts[1]
        field = parts[2]
        owner = parts[3]
        if owner != user_id:
            try:
                await query.answer("You cannot confirm edits for another user's profile.", show_alert=True)
            except Exception:
                pass
            return
        pending = context.user_data.get("pending_edit") or {}
        pending_type = pending.get("type")
        pending_value = pending.get("value")
        is_creation = bool(pending.get("is_creation"))
        if field != pending_type:
            if action == "again":
                prompts = {
                    "name": "Okay — send your new name again.",
                    "bio": "Okay — send your new bio again.",
                    "photo": "Okay — send your new photo again.",
                }
                try:
                    await query.edit_message_text(prompts.get(field, "Okay — send again."))
                except Exception:
                    try:
                        await query.message.reply_text(prompts.get(field, "Okay — send again."))
                    except Exception:
                        pass
                context.user_data["await_edit"] = field
            return
        if action == "save" and pending_type in ("name", "bio", "photo"):
            ensure_profile_slot(user_id)
            if pending_type == "name":
                db["profiles"][user_id]["name"] = str(pending_value)
                msg_txt = "Name saved!"
            elif pending_type == "bio":
                db["profiles"][user_id]["bio"] = str(pending_value)
                msg_txt = "Bio saved!"
            else:
                db["profiles"][user_id]["photo"] = str(pending_value)
                msg_txt = "Profile pic saved!"
            save_db(db)
            kb_edit = InlineKeyboardMarkup([
                [InlineKeyboardButton("Edit Name", callback_data=f"edit:name:{user_id}"), InlineKeyboardButton("Edit Bio", callback_data=f"edit:bio:{user_id}")],
                [InlineKeyboardButton("Edit Photo", callback_data=f"edit:photo:{user_id}")],
                [InlineKeyboardButton("⬅ Back to Menu", callback_data=f"menu:main:{user_id}")],
            ])
            try:
                if db["profiles"][user_id].get("photo"):
                    await safe_edit_media(context, query.message.chat.id, query.message.message_id, InputMediaPhoto(media=db["profiles"][user_id]["photo"], caption=profile_caption(user_id)), caption=profile_caption(user_id), reply_markup=kb_edit)
                else:
                    await safe_edit_caption(context, query.message.chat.id, query.message.message_id, profile_caption(user_id), reply_markup=kb_edit)
                set_profile_msg_ref(context, query.message.chat.id, query.message.message_id)
            except Exception:
                try:
                    await query.edit_message_text(msg_txt)
                except Exception:
                    try:
                        await query.message.reply_text(msg_txt)
                    except Exception:
                        pass
                try:
                    if db["profiles"][user_id].get("photo"):
                        await query.message.reply_photo(photo=db["profiles"][user_id]["photo"], caption=profile_caption(user_id), reply_markup=kb_edit)
                    else:
                        await query.message.reply_text(profile_caption(user_id), reply_markup=kb_edit)
                except Exception:
                    pass
            context.user_data.pop("pending_edit", None)
            context.user_data.pop("await_edit", None)
            context.user_data.pop("await_edit_owner", None)
            if is_creation:
                try:
                    await query.edit_message_text("✅ Profile created! Use /menu to open the main menu.")
                except Exception:
                    try:
                        await query.message.reply_text("✅ Profile created! Use /menu to open the main menu.")
                    except Exception:
                        pass
        elif action == "again":
            prompts = {
                "name": "Okay — send your new name again.",
                "bio": "Okay — send your new bio again.",
                "photo": "Okay — send your new photo again.",
            }
            try:
                await query.edit_message_text(prompts.get(pending_type, "Okay — send again."))
            except Exception:
                try:
                    await query.message.reply_text(prompts.get(pending_type, "Okay — send again."))
                except Exception:
                    pass
            context.user_data["await_edit"] = pending_type
        return

    if data.startswith("heart:") or data in ("next","chat","myhearts","hearted"):
        if data == "hearted":
            try:
                await query.message.reply_text("You already hearted this profile.")
            except Exception:
                pass
            return
        if data == "next":
            await show_next_profile(query, context, replace=True)
            return
        if data == "chat":
            try:
                cap = (getattr(query.message, "caption", None) or "") + "\n\n💬 Chat feature coming soon!"
                await safe_edit_caption(context, query.message.chat.id, query.message.message_id, cap, reply_markup=query.message.reply_markup)
            except Exception:
                try:
                    await query.message.reply_text("💬 Chat feature coming soon!")
                except Exception:
                    pass
            return
        if data == "myhearts":
            try:
                await query.message.reply_text(f"💖 You have received {int(db['hearts'].get(user_id,0))} hearts.")
            except Exception:
                pass
            return
        if data.startswith("heart:"):
            target_id = data.split(":", 1)[1]
            if target_id not in db["profiles"] or not all(db["profiles"][target_id].get(k) for k in ("name", "bio", "photo")):
                try:
                    await query.edit_message_text("This profile is no longer available.")
                except Exception:
                    try:
                        await query.message.reply_text("This profile is no longer available.")
                    except Exception:
                        pass
                return
            if str(target_id) == user_id:
                try:
                    await query.message.reply_text("You can’t heart your own profile.")
                except Exception:
                    pass
                return
            success = await record_heart(context, user_id, target_id)
            if success:
                new_cap = profile_caption(target_id) + "\n\n❤️ You gave a heart! (notification sent; total hearts not incremented)"
                kb = browse_keyboard(user_id, target_id)
                try:
                    await safe_edit_caption(context, query.message.chat.id, query.message.message_id, new_cap, reply_markup=kb)
                except Exception:
                    try:
                        await query.message.reply_text("❤️ Heart sent! The user was notified.")
                    except Exception:
                        pass
            else:
                try:
                    await query.message.reply_text("You already hearted this profile or the profile is unavailable.")
                except Exception:
                    pass
            return

    if data.startswith("delete:"):
        parts = data.split(":")
        if len(parts) < 3:
            return
        action = parts[1]
        owner = parts[2]
        if owner != user_id:
            try:
                await query.answer("You cannot delete another user's account.", show_alert=True)
            except Exception:
                pass
            return
        if action == "confirm":
            if user_id in db["profiles"]:
                db["profiles"].pop(user_id, None)
            db["hearts"].pop(user_id, None)
            if "given" in db:
                for giver, lst in list(db["given"].items()):
                    db["given"][giver] = [x for x in lst if x != user_id]
            db["given"].pop(user_id, None)
            save_db(db)
            overwrite_backup_with_current_db()
            try:
                await query.edit_message_text("Your account has been permanently deleted.")
            except Exception:
                try:
                    await query.message.reply_text("Your account has been permanently deleted.")
                except Exception:
                    pass
        else:
            try:
                await query.edit_message_text("Delete cancelled.")
            except Exception:
                try:
                    await query.message.reply_text("Delete cancelled.")
                except Exception:
                    pass
        return

async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user is None or update.message is None:
        return
    user_id = str(update.effective_user.id)
    flow = context.user_data.get("flow")
    await_edit = context.user_data.get("await_edit")
    if flow == "create_name":
        name = (update.message.text or "").strip()
        if not name:
            try:
                await update.message.reply_text("Please send a valid name.")
            except Exception:
                pass
            return
        ensure_profile_slot(user_id)
        db["profiles"][user_id]["name"] = name
        save_db(db)
        context.user_data["flow"] = "create_bio"
        try:
            await update.message.reply_text("Great! Now send your bio.")
        except Exception:
            pass
        return
    if flow == "create_bio":
        bio = (update.message.text or "").strip()
        if not bio:
            try:
                await update.message.reply_text("Please send a valid bio.")
            except Exception:
                pass
            return
        ensure_profile_slot(user_id)
        db["profiles"][user_id]["bio"] = bio
        save_db(db)
        context.user_data["flow"] = "create_photo"
        try:
            await update.message.reply_text("Nice! Finally, send a profile photo (not a file).\n\nWarning: Once you save your profile photo it will be permanent. To change it later you must delete your account and create a new one.")
        except Exception:
            pass
        return
    if await_edit in ("name", "bio"):
        owner = context.user_data.get("await_edit_owner", user_id)
        if owner != user_id:
            try:
                await update.message.reply_text("You cannot submit edits for another user's profile.")
            except Exception:
                pass
            return
        text = (update.message.text or "").strip()
        if not text:
            try:
                await update.message.reply_text("Please send valid text.")
            except Exception:
                pass
            return
        current = db["profiles"].get(user_id, {})
        if await_edit == "name":
            cap = f"👤 {text}\n\n{current.get('bio','')}\n\n💖 Hearts: {int(db['hearts'].get(user_id,0))}\n\n(Preview — not saved yet)"
        else:
            cap = f"👤 {current.get('name','')}\n\n{text}\n\n💖 Hearts: {int(db['hearts'].get(user_id,0))}\n\n(Preview — not saved yet)"
        ref = get_profile_msg_ref(context)
        if ref and ref[1]:
            try:
                await safe_edit_caption(context, ref[0], ref[1], cap)
            except Exception:
                pass
        context.user_data["pending_edit"] = {"type": await_edit, "value": text, "owner": user_id}
        kb = InlineKeyboardMarkup([[InlineKeyboardButton("✅ Save", callback_data=f"confirm:save:{await_edit}:{user_id}"), InlineKeyboardButton("✏️ Keep editing", callback_data=f"confirm:again:{await_edit}:{user_id}")]])
        try:
            await update.message.reply_text("Is this okay?", reply_markup=kb)
        except Exception:
            pass
        return

async def photo_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user is None or update.message is None:
        return
    user_id = str(update.effective_user.id)
    flow = context.user_data.get("flow")
    await_edit = context.user_data.get("await_edit")
    if flow == "create_photo":
        if not update.message.photo:
            try:
                await update.message.reply_text("Please send a photo (not a document).")
            except Exception:
                pass
            return
        new_photo_id = update.message.photo[-1].file_id
        context.user_data["pending_edit"] = {"type": "photo", "value": new_photo_id, "is_creation": True, "owner": user_id}
        kb = InlineKeyboardMarkup([[InlineKeyboardButton("✅ Save", callback_data=f"confirm:save:photo:{user_id}"), InlineKeyboardButton("✏️ Keep editing", callback_data=f"confirm:again:photo:{user_id}")]])
        try:
            await update.message.reply_photo(photo=new_photo_id, caption="Preview — this photo will be permanent. To change it later you must delete your account and create a new one.\n\nIs this okay?", reply_markup=kb)
        except Exception:
            try:
                await update.message.reply_text("Preview — this photo will be permanent. To change it later you must delete your account and create a new one.\n\nIs this okay?", reply_markup=kb)
            except Exception:
                pass
        return
    if await_edit == "photo":
        if not update.message.photo:
            try:
                await update.message.reply_text("Please send a photo (not a document).")
            except Exception:
                pass
            return
        new_photo_id = update.message.photo[-1].file_id
        ref = get_profile_msg_ref(context)
        cap = "Preview: new profile photo\n\nWarning: Once saved this photo is permanent. To change it later you must delete your account and create a new one."
        if ref and ref[1]:
            try:
                await safe_edit_media(context, ref[0], ref[1], InputMediaPhoto(media=new_photo_id, caption=cap), caption=cap)
            except Exception:
                pass
        context.user_data["pending_edit"] = {"type": "photo", "value": new_photo_id, "owner": user_id}
        kb = InlineKeyboardMarkup([[InlineKeyboardButton("✅ Save", callback_data=f"confirm:save:photo:{user_id}"), InlineKeyboardButton("✏️ Keep editing", callback_data=f"confirm:again:photo:{user_id}")]])
        try:
            await update.message.reply_text("Is this okay?", reply_markup=kb)
        except Exception:
            pass
        return

async def show_next_profile(query, context: ContextTypes.DEFAULT_TYPE, replace: bool = False):
    q = getattr(query, "callback_query", None) or query
    if q is None:
        return
    user = q.from_user or getattr(query, "effective_user", None)
    if user is None:
        return
    viewer_id = str(user.id)
    cands = browse_candidates(viewer_id)
    if not cands:
        try:
            await q.edit_message_text("No more profiles — no new profiles yet.")
        except Exception:
            try:
                await q.message.reply_text("No more profiles — no new profiles yet.")
            except Exception:
                pass
        return
    idx = context.user_data.get("browse_index", 0) % len(cands)
    target_id = cands[idx]
    target = db["profiles"].get(target_id, {})
    cap = profile_caption(target_id)
    kb = browse_keyboard(viewer_id, target_id)
    try:
        if replace:
            if target.get("photo"):
                await q.edit_message_media(InputMediaPhoto(media=target.get("photo"), caption=cap), reply_markup=kb)
            else:
                await q.edit_message_text(cap, reply_markup=kb)
        else:
            if target.get("photo"):
                await q.message.reply_photo(photo=target.get("photo"), caption=cap, reply_markup=kb)
            else:
                await q.message.reply_text(cap, reply_markup=kb)
    except Exception:
        try:
            await q.message.reply_text(cap, reply_markup=kb)
        except Exception:
            pass
    context.user_data["browse_index"] = idx + 1

async def on_error(update: object, context: ContextTypes.DEFAULT_TYPE):
    print("Exception:", getattr(context, "error", None))

def main():
    BOT_TOKEN = "YOUR_TOKEN_HERE"
    if not BOT_TOKEN or BOT_TOKEN == "YOUR_TOKEN_HERE":
        print("Set BOT_TOKEN in the script before running.")
        return
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("menu", lambda u, c: menu_handler(u, c)))
    app.add_handler(CallbackQueryHandler(callback_router, pattern=r".*"))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))
    app.add_handler(MessageHandler(filters.PHOTO, photo_handler))
    app.add_error_handler(on_error)
    print("🤖 Bot running with memory-safe storage…")
    app.run_polling()

async def menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user is None:
        return
    user_id = str(update.effective_user.id)
    if not has_profile(user_id):
        try:
            await update.message.reply_text("⚠️ You must create a profile first. Use /start.")
        except Exception:
            pass
        return
    try:
        await update.message.reply_text("📍 Main Menu", reply_markup=main_menu_keyboard(user_id))
    except Exception:
        pass

if __name__ == "__main__":
    main()
