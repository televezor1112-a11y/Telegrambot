import logging
import asyncio
import sqlite3
import uuid
import json
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.utils.keyboard import InlineKeyboardBuilder, ReplyKeyboardBuilder
from flask import Flask
from threading import Thread

# ASOSIY SOZLAMALAR
TOKEN = "bot tokin kirit"
BOSH_ADMIN = 5694315751

# FLASK WEB SERVER
app = Flask('')

@app.route('/')
def home():
    return "Bot ishlayapti!"

def run_flask():
    app.run(host='0.0.0.0', port=8080)

Thread(target=run_flask, daemon=True).start()

logging.basicConfig(level=logging.INFO)
bot = Bot(token=TOKEN)
dp = Dispatcher(storage=MemoryStorage())

# BAZA
conn = sqlite3.connect("donat_bot_v3.db")
cursor = conn.cursor()

cursor.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username TEXT, status TEXT DEFAULT 'active', total_orders INTEGER DEFAULT 0, last_seen TEXT)")
cursor.execute("CREATE TABLE IF NOT EXISTS admin_levels (user_id INTEGER PRIMARY KEY, level INTEGER)")
cursor.execute("CREATE TABLE IF NOT EXISTS categories (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT UNIQUE)")
cursor.execute("CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY AUTOINCREMENT, category_id INTEGER, photo_id TEXT DEFAULT NULL, description TEXT)")
cursor.execute("CREATE TABLE IF NOT EXISTS promocodes (code TEXT PRIMARY KEY, game TEXT, max_uses INTEGER, used_count INTEGER DEFAULT 0)")
cursor.execute("CREATE TABLE IF NOT EXISTS settings (key TEXT PRIMARY KEY, val_text TEXT, val_photo TEXT)")
cursor.execute("CREATE TABLE IF NOT EXISTS channels (id INTEGER PRIMARY KEY AUTOINCREMENT, channel_id TEXT UNIQUE, title TEXT)")
cursor.execute("CREATE TABLE IF NOT EXISTS order_messages (order_id TEXT PRIMARY KEY, messages TEXT, status TEXT DEFAULT 'pending', user_id INTEGER)")
cursor.execute("CREATE TABLE IF NOT EXISTS ratings (order_id TEXT PRIMARY KEY, user_id INTEGER, rating INTEGER)")
conn.commit()

# last_seen ustunini qo'shish (eski bazalar uchun)
try:
    cursor.execute("ALTER TABLE users ADD COLUMN last_seen TEXT")
    conn.commit()
except:
    pass

cursor.execute("INSERT OR IGNORE INTO settings VALUES ('card_info', '8600 0000 0000 0000\nEshmatov Toshmat', NULL)")
cursor.execute("INSERT OR IGNORE INTO settings VALUES ('qayerdan', 'Kanalimizga a''zo bo''ling!', NULL)")
cursor.execute("INSERT OR IGNORE INTO settings VALUES ('murojaat', 'Aloqa uchun ish vaqti: 09:00 - 22:00\nAdmin: @username', NULL)")
conn.commit()

def get_admin_level(user_id):
    if user_id == BOSH_ADMIN: return 1
    cursor.execute("SELECT level FROM admin_levels WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    return row[0] if row else 0

async def get_all_admins():
    cursor.execute("SELECT user_id FROM admin_levels")
    return [BOSH_ADMIN] + [r[0] for r in cursor.fetchall()]

async def check_subscription(user_id):
    cursor.execute("SELECT channel_id FROM channels")
    channels = cursor.fetchall()
    for ch in channels:
        try:
            member = await bot.get_chat_member(ch[0], user_id)
            if member.status in ["left", "kicked"]:
                return False, ch[0]
        except:
            pass
    return True, None

def update_last_seen(user_id):
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute("UPDATE users SET last_seen = ? WHERE id = ?", (now, user_id))
    conn.commit()

class BuilderStates(StatesGroup):
    add_category = State()
    add_product_content = State()

class AdminStates(StatesGroup):
    add_admin_id = State()
    add_admin_level = State()
    remove_admin_id = State()
    change_card = State()
    change_qayerdan = State()
    change_murojaat = State()
    add_promo_game = State()
    add_promo_code = State()
    add_promo_limit = State()
    ban_user = State()
    unban_user = State()
    search_user = State()
    search_username = State()
    broadcast_msg = State()
    add_channel = State()
    edit_category_name = State()
    send_msg_to_user = State()

class UserStates(StatesGroup):
    wait_for_player_id = State()
    wait_for_receipt = State()
    wait_for_promo = State()
    wait_for_promo_id = State()
    wait_for_confirm = State()
    wait_for_extra_msg = State()
    wait_for_rating = State()

@dp.message.middleware()
async def check_ban_middleware(handler, event: types.Message, data: dict):
    cursor.execute("SELECT status FROM users WHERE id = ?", (event.from_user.id,))
    row = cursor.fetchone()
    if row and row[0] == 'banned':
        await event.answer("❌ Siz botdan bloklangansiz!")
        return
    update_last_seen(event.from_user.id)
    return await handler(event, data)

def get_main_menu(user_id):
    builder = ReplyKeyboardBuilder()
    builder.button(text="🎮 O'yinlar va Narxlar")
    builder.button(text="🎟 Promokod")
    builder.button(text="✍️ Adminga murojaat")
    builder.button(text="📜 Buyurtma tarixi")
    builder.button(text="🏆 Top donaterlar")
    if get_admin_level(user_id) in [1, 2, 3]:
        builder.button(text="👑 Admin Panel")
    builder.adjust(2)
    return builder.as_markup(resize_keyboard=True)

def get_admin_menu(user_id):
    lvl = get_admin_level(user_id)
    builder = InlineKeyboardBuilder()
    if lvl == 1:
        builder.button(text="👥 Adminlarni Boshqarish", callback_data="adm_manage_admins")
        builder.button(text="📡 Majburiy Obuna", callback_data="adm_channels")
        builder.button(text="🔍 Username bo'yicha ID topish", callback_data="adm_find_username")
    if lvl in [1, 2]:
        builder.button(text="💳 Karta Sozlamalari", callback_data="adm_card_set")
        builder.button(text="📝 Qayerdan olaman tahriri", callback_data="adm_qayerdan_set")
        builder.button(text="✍️ Murojaat matni tahriri", callback_data="adm_murojaat_set")
        builder.button(text="🎟 Promokod Qo'shish", callback_data="adm_add_promo")
        builder.button(text="✉️ Rassilka", callback_data="adm_broadcast")
        builder.button(text="📊 Statistika", callback_data="adm_stats")
        builder.button(text="🔍 Foydalanuvchi Qidirish", callback_data="adm_search")
    if lvl in [1, 2, 3]:
        builder.button(text="🚫 Ban", callback_data="adm_ban")
        builder.button(text="🟢 Unban", callback_data="adm_unban")
    builder.adjust(1)
    return builder.as_markup()

# START
@dp.message(Command("start"))
async def start_cmd(message: types.Message):
    user_id = message.from_user.id
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute("INSERT OR IGNORE INTO users (id, username, last_seen) VALUES (?, ?, ?)", (user_id, message.from_user.username, now))
    cursor.execute("UPDATE users SET username = ?, last_seen = ? WHERE id = ?", (message.from_user.username, now, user_id))
    conn.commit()
    lvl = get_admin_level(user_id)
    role = "Bosh Admin 👑" if lvl == 1 else ("Asosiy Admin 🥈" if lvl == 2 else ("Moderator 🥉" if lvl == 3 else "Player 🎮"))
    await message.answer(f"👋 Xush kelibsiz!\n\nDarajangiz: *{role}*", reply_markup=get_main_menu(user_id), parse_mode="Markdown")

# MAJBURIY OBUNA
@dp.callback_query(F.data == "adm_channels")
async def adm_channels(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) != 1: return
    cursor.execute("SELECT id, channel_id, title FROM channels")
    channels = cursor.fetchall()
    builder = InlineKeyboardBuilder()
    builder.button(text="➕ Kanal qo'shish", callback_data="adm_add_channel")
    for ch in channels:
        builder.button(text=f"🗑 {ch[2]} o'chirish", callback_data=f"adm_del_channel_{ch[0]}")
    builder.adjust(1)
    text = "📡 Majburiy obuna kanallari:\n\n"
    if not channels:
        text += "Hozircha kanal yo'q."
    else:
        for ch in channels:
            text += f"• {ch[2]} — `{ch[1]}`\n"
    await call.message.answer(text, reply_markup=builder.as_markup(), parse_mode="Markdown")

@dp.callback_query(F.data == "adm_add_channel")
async def adm_add_channel_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("📡 Kanal yoki guruh ID sini yuboring:\nMasalan: @kanalim yoki -1001234567890\n\n⚠️ Bot kanalga admin qilingan bo'lishi kerak!")
    await state.set_state(AdminStates.add_channel)

@dp.message(AdminStates.add_channel)
async def adm_add_channel_finish(message: types.Message, state: FSMContext):
    channel_id = message.text.strip()
    try:
        chat = await bot.get_chat(channel_id)
        cursor.execute("INSERT OR IGNORE INTO channels (channel_id, title) VALUES (?, ?)", (channel_id, chat.title))
        conn.commit()
        await message.answer(f"✅ '{chat.title}' kanali qo'shildi!")
    except:
        await message.answer("❌ Kanal topilmadi! Bot kanalga admin qilinganmi?")
    await state.clear()

@dp.callback_query(F.data.startswith("adm_del_channel_"))
async def adm_del_channel(call: types.CallbackQuery):
    ch_id = int(call.data.split("_")[3])
    cursor.execute("DELETE FROM channels WHERE id = ?", (ch_id,))
    conn.commit()
    await call.message.answer("🗑 Kanal o'chirildi.")

@dp.callback_query(F.data == "check_sub")
async def check_sub(call: types.CallbackQuery):
    subscribed, channel = await check_subscription(call.from_user.id)
    if subscribed:
        await call.message.answer("✅ Rahmat! Endi botdan foydalanishingiz mumkin.", reply_markup=get_main_menu(call.from_user.id))
    else:
        await call.answer("❌ Hali a'zo bo'lmadingiz!", show_alert=True)

# USERNAME BO'YICHA ID TOPISH
@dp.callback_query(F.data == "adm_find_username")
async def find_username_start(call: types.CallbackQuery, state: FSMContext):
    if get_admin_level(call.from_user.id) != 1: return
    await call.message.answer("🔍 Username kiriting (@username):")
    await state.set_state(AdminStates.search_username)

@dp.message(AdminStates.search_username)
async def find_username_finish(message: types.Message, state: FSMContext):
    username = message.text.strip().replace("@", "")
    cursor.execute("SELECT id, username, status, total_orders FROM users WHERE username = ?", (username,))
    row = cursor.fetchone()
    if row:
        await message.answer(f"✅ Topildi!\n\n🆔 ID: `{row[0]}`\n👤 Username: @{row[1]}\n⚙️ Holat: {row[2]}\n📈 Xaridlar: {row[3]} ta", parse_mode="Markdown")
    else:
        await message.answer("❌ Topilmadi. Foydalanuvchi /start bosmagandir.")
    await state.clear()

# ADMINLAR BOSHQARUVI
@dp.callback_query(F.data == "adm_manage_admins")
async def manage_admins(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) != 1: return
    builder = InlineKeyboardBuilder()
    builder.button(text="➕ Admin Qo'shish", callback_data="adm_add_new")
    builder.button(text="➖ Adminni O'chirish", callback_data="adm_del")
    builder.button(text="📋 Adminlar Ro'yxati", callback_data="adm_list")
    builder.adjust(1)
    await call.message.answer("👥 Adminlarni boshqarish:", reply_markup=builder.as_markup())

@dp.callback_query(F.data == "adm_add_new")
async def add_admin_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("🔢 Yangi adminning Telegram ID sini yozing:")
    await state.set_state(AdminStates.add_admin_id)

@dp.message(AdminStates.add_admin_id)
async def add_admin_id_handler(message: types.Message, state: FSMContext):
    await state.update_data(target_adm_id=int(message.text.strip()))
    builder = InlineKeyboardBuilder()
    builder.button(text="2-daraja (Asosiy Admin)", callback_data="set_lvl_2")
    builder.button(text="3-daraja (Moderator/Kassir)", callback_data="set_lvl_3")
    builder.adjust(1)
    await message.answer("Daraja tanlang:", reply_markup=builder.as_markup())
    await state.set_state(AdminStates.add_admin_level)

@dp.callback_query(AdminStates.add_admin_level, F.data.startswith("set_lvl_"))
async def add_admin_finish(call: types.CallbackQuery, state: FSMContext):
    lvl = int(call.data.split("_")[2])
    data = await state.get_data()
    cursor.execute("INSERT OR REPLACE INTO admin_levels (user_id, level) VALUES (?, ?)", (data['target_adm_id'], lvl))
    conn.commit()
    await call.message.answer("✅ Admin tayinlandi!")
    await state.clear()

@dp.callback_query(F.data == "adm_del")
async def del_admin_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("🔢 O'chiriladigan admin ID sini kiriting:")
    await state.set_state(AdminStates.remove_admin_id)

@dp.message(AdminStates.remove_admin_id)
async def del_admin_finish(message: types.Message, state: FSMContext):
    cursor.execute("DELETE FROM admin_levels WHERE user_id = ?", (int(message.text.strip()),))
    conn.commit()
    await message.answer("🗑 Admin o'chirildi.")
    await state.clear()

@dp.callback_query(F.data == "adm_list")
async def list_admins(call: types.CallbackQuery):
    cursor.execute("SELECT user_id, level FROM admin_levels")
    text = f"👑 Bosh Admin: `{BOSH_ADMIN}`\n\n"
    rows = cursor.fetchall()
    if not rows:
        text += "Boshqa adminlar yo'q."
    for r in rows:
        role = "Asosiy Admin (2)" if r[1] == 2 else "Moderator (3)"
        text += f"👤 ID: `{r[0]}` — {role}\n"
    await call.message.answer(text, parse_mode="Markdown")

# SOZLAMALAR
@dp.callback_query(F.data == "adm_card_set")
async def change_card_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("💳 Yangi karta raqami va egasining ismini kiriting:")
    await state.set_state(AdminStates.change_card)

@dp.message(AdminStates.change_card)
async def change_card_finish(message: types.Message, state: FSMContext):
    cursor.execute("UPDATE settings SET val_text = ? WHERE key = 'card_info'", (message.text.strip(),))
    conn.commit()
    await message.answer("✅ Karta yangilandi!")
    await state.clear()

@dp.callback_query(F.data == "adm_qayerdan_set")
async def change_q_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("📝 Yangi postni (rasm yoki matn) yuboring:")
    await state.set_state(AdminStates.change_qayerdan)

@dp.message(AdminStates.change_qayerdan)
async def change_q_finish(message: types.Message, state: FSMContext):
    p_id = message.photo[-1].file_id if message.photo else None
    desc = message.caption if message.photo else message.text
    cursor.execute("UPDATE settings SET val_text = ?, val_photo = ? WHERE key = 'qayerdan'", (desc, p_id))
    conn.commit()
    await message.answer("✅ Post yangilandi!")
    await state.clear()

@dp.callback_query(F.data == "adm_murojaat_set")
async def change_m_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("✍️ Yangi murojaat matnini kiriting:")
    await state.set_state(AdminStates.change_murojaat)

@dp.message(AdminStates.change_murojaat)
async def change_m_finish(message: types.Message, state: FSMContext):
    cursor.execute("UPDATE settings SET val_text = ? WHERE key = 'murojaat'", (message.text.strip(),))
    conn.commit()
    await message.answer("✅ Murojaat matni yangilandi!")
    await state.clear()

# O'YINLAR VA TAHRIRLASH
@dp.message(F.text == "🎮 O'yinlar va Narxlar")
async def show_games(message: types.Message):
    subscribed, channel = await check_subscription(message.from_user.id)
    if not subscribed:
        builder = InlineKeyboardBuilder()
        builder.button(text="✅ A'zo bo'ldim", callback_data="check_sub")
        await message.answer(f"❌ Botdan foydalanish uchun avval kanalga a'zo bo'ling:\n{channel}", reply_markup=builder.as_markup())
        return
    cursor.execute("SELECT id, name FROM categories")
    cats = cursor.fetchall()
    builder = InlineKeyboardBuilder()
    lvl = get_admin_level(message.from_user.id)
    for c in cats:
        if lvl in [1, 2]:
            builder.button(text=f"⚙️ {c[1]}", callback_data=f"adm_cat_manage_{c[0]}")
        else:
            builder.button(text=c[1], callback_data=f"v_cat_{c[0]}")
    if lvl in [1, 2]:
        builder.button(text="➕ Yangi O'yin Qo'shish", callback_data="add_new_cat")
    builder.adjust(2)
    await message.answer("🎮 O'yinni tanlang:", reply_markup=builder.as_markup())

@dp.callback_query(F.data == "add_new_cat")
async def add_cat_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("📝 Yangi o'yin nomini kiriting:")
    await state.set_state(BuilderStates.add_category)

@dp.message(BuilderStates.add_category)
async def add_cat_finish(message: types.Message, state: FSMContext):
    try:
        cursor.execute("INSERT INTO categories (name) VALUES (?)", (message.text.strip(),))
        conn.commit()
        await message.answer("✅ O'yin qo'shildi!")
    except:
        await message.answer("❌ Bu nom mavjud!")
    await state.clear()

@dp.callback_query(F.data.startswith("adm_cat_manage_"))
async def adm_cat_manage(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) not in [1, 2]: return
    cat_id = int(call.data.split("_")[3])
    cursor.execute("SELECT name FROM categories WHERE id = ?", (cat_id,))
    cat_name = cursor.fetchone()[0]
    builder = InlineKeyboardBuilder()
    builder.button(text="📝 Nomini almashtirish", callback_data=f"adm_cat_edit_{cat_id}")
    builder.button(text="🗑 O'yinni o'chirish", callback_data=f"adm_cat_del_{cat_id}")
    builder.button(text="👁 Tariflarni ko'rish", callback_data=f"v_cat_{cat_id}")
    builder.button(text="⬅️ Orqaga", callback_data="adm_back_to_games")
    builder.adjust(1)
    await call.message.edit_text(f"🎮 *{cat_name}* o'yinini boshqarish:", reply_markup=builder.as_markup(), parse_mode="Markdown")

@dp.callback_query(F.data == "adm_back_to_games")
async def adm_back_to_games(call: types.CallbackQuery):
    await call.message.delete()
    await show_games(call.message)

@dp.callback_query(F.data.startswith("adm_cat_del_"))
async def adm_cat_delete(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) not in [1, 2]: return
    cat_id = int(call.data.split("_")[3])
    cursor.execute("DELETE FROM categories WHERE id = ?", (cat_id,))
    cursor.execute("DELETE FROM products WHERE category_id = ?", (cat_id,))
    conn.commit()
    await call.answer("🗑 O'yin va uning barcha tariflari o'chirildi!", show_alert=True)
    await call.message.delete()
    await show_games(call.message)

@dp.callback_query(F.data.startswith("adm_cat_edit_"))
async def adm_cat_edit_start(call: types.CallbackQuery, state: FSMContext):
    if get_admin_level(call.from_user.id) not in [1, 2]: return
    cat_id = int(call.data.split("_")[3])
    await state.update_data(edit_cat_id=cat_id)
    await call.message.answer("📝 Ushbu o'yin uchun yangi nom kiriting:")
    await state.set_state(AdminStates.edit_category_name)

@dp.message(AdminStates.edit_category_name)
async def adm_cat_edit_finish(message: types.Message, state: FSMContext):
    new_name = message.text.strip()
    data = await state.get_data()
    cat_id = data['edit_cat_id']
    try:
        cursor.execute("UPDATE categories SET name = ? WHERE id = ?", (new_name, cat_id))
        conn.commit()
        await message.answer(f"✅ O'yin nomi *{new_name}* ga o'zgartirildi!", parse_mode="Markdown")
    except:
        await message.answer("❌ Xatolik! Bu nom mavjud bo'lishi mumkin.")
    await state.clear()

@dp.callback_query(F.data.startswith("v_cat_"))
async def view_category(call: types.CallbackQuery):
    cat_id = int(call.data.split("_")[2])
    cursor.execute("SELECT id, photo_id, description FROM products WHERE category_id = ?", (cat_id,))
    products = cursor.fetchall()
    if not products:
        await call.message.answer("📭 Bu kategoriyada hozircha narx yo'q.")
        if get_admin_level(call.from_user.id) in [1, 2]:
            opt = InlineKeyboardBuilder()
            opt.button(text="📝 Post qo'shish", callback_data=f"add_post_{cat_id}")
            await call.message.answer("Boshqaruv:", reply_markup=opt.as_markup())
        return
    for p in products:
        b = InlineKeyboardBuilder()
        b.button(text="🛍 Buyurtma berish", callback_data=f"b_prod_{p[0]}")
        if get_admin_level(call.from_user.id) in [1, 2]:
            b.button(text="🗑 O'chirish", callback_data=f"d_prod_{p[0]}")
        b.adjust(1)
        if p[1]:
            await call.message.answer_photo(photo=p[1], caption=p[2], reply_markup=b.as_markup())
        else:
            await call.message.answer(text=p[2], reply_markup=b.as_markup())
    if get_admin_level(call.from_user.id) in [1, 2]:
        opt = InlineKeyboardBuilder()
        opt.button(text="📝 Post qo'shish", callback_data=f"add_post_{cat_id}")
        await call.message.answer("Boshqaruv:", reply_markup=opt.as_markup())

@dp.callback_query(F.data.startswith("add_post_"))
async def add_post_start(call: types.CallbackQuery, state: FSMContext):
    await state.update_data(c_id=int(call.data.split("_")[2]))
    await call.message.answer("📝 Post matni yoki rasmini yuboring:")
    await state.set_state(BuilderStates.add_product_content)

@dp.message(BuilderStates.add_product_content)
async def add_post_finish(message: types.Message, state: FSMContext):
    data = await state.get_data()
    p_id = message.photo[-1].file_id if message.photo else None
    desc = message.caption if message.photo else message.text
    cursor.execute("INSERT INTO products (category_id, photo_id, description) VALUES (?, ?, ?)", (data['c_id'], p_id, desc))
    conn.commit()
    await message.answer("✅ Post joylandi!")
    await state.clear()

@dp.callback_query(F.data.startswith("d_prod_"))
async def del_post(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) in [1, 2]:
        cursor.execute("DELETE FROM products WHERE id = ?", (int(call.data.split("_")[2]),))
        conn.commit()
        await call.message.delete()

# BUYURTMA TIZIMI
@dp.callback_query(F.data.startswith("b_prod_"))
async def buy_start(call: types.CallbackQuery, state: FSMContext):
    subscribed, channel = await check_subscription(call.from_user.id)
    if not subscribed:
        builder = InlineKeyboardBuilder()
        builder.button(text="✅ A'zo bo'ldim", callback_data="check_sub")
        await call.message.answer(f"❌ Avval kanalga a'zo bo'ling:\n{channel}", reply_markup=builder.as_markup())
        return
    p_id = int(call.data.split("_")[2])
    cursor.execute("SELECT p.description, c.name FROM products p JOIN categories c ON p.category_id = c.id WHERE p.id = ?", (p_id,))
    row = cursor.fetchone()
    await state.update_data(o_game=row[1], o_desc=row[0])
    await call.message.answer(f"🎮 *O'yin:* {row[1]}\n📋 *Tarif:*\n{row[0]}\n\n🆔 O'yindagi *ID raqamingizni* yozing:", parse_mode="Markdown")
    await state.set_state(UserStates.wait_for_player_id)

@dp.message(UserStates.wait_for_player_id)
async def buy_id_input(message: types.Message, state: FSMContext):
    await state.update_data(o_player=message.text.strip())
    cursor.execute("SELECT val_text FROM settings WHERE key = 'card_info'")
    card = cursor.fetchone()[0]
    await message.answer(f"💳 *To'lov ma'lumotlari:*\n\nKarta raqami:\n`{card}`\n\nTo'lov qiling va *chek rasmini* yuboring:", parse_mode="Markdown")
    await state.set_state(UserStates.wait_for_receipt)

@dp.message(UserStates.wait_for_receipt, F.photo)
async def buy_receipt_finish(message: types.Message, state: FSMContext):
    data = await state.get_data()
    await state.update_data(receipt_photo=message.photo[-1].file_id)

    # ID to'g'riligini so'rash
    builder = InlineKeyboardBuilder()
    builder.button(text="✅ Ha, to'g'ri", callback_data="confirm_yes")
    builder.button(text="❌ Yo'q, xato", callback_data="confirm_no")
    builder.adjust(2)
    await message.answer(
        f"📋 *Buyurtma ma'lumotlari:*\n\n"
        f"🎮 O'yin: {data['o_game']}\n"
        f"🆔 O'yin ID: `{data['o_player']}`\n\n"
        f"Gapingiz yoki ID to'g'rimi? ✅",
        reply_markup=builder.as_markup(),
        parse_mode="Markdown"
    )
    await state.set_state(UserStates.wait_for_confirm)

@dp.callback_query(UserStates.wait_for_confirm, F.data == "confirm_yes")
async def confirm_yes(call: types.CallbackQuery, state: FSMContext):
    await call.message.edit_reply_markup()
    await call.message.answer("✅ Zo'r! Buyurtmangiz adminga yuborilmoqda...")
    await send_order_to_admins(call.message, state, call.from_user)

@dp.callback_query(UserStates.wait_for_confirm, F.data == "confirm_no")
async def confirm_no(call: types.CallbackQuery, state: FSMContext):
    await call.message.edit_reply_markup()
    await call.message.answer("✏️ Xabaringizni yozing, to'g'rilab yuboramiz:")
    await state.set_state(UserStates.wait_for_extra_msg)

@dp.message(UserStates.wait_for_extra_msg)
async def extra_msg_handler(message: types.Message, state: FSMContext):
    await state.update_data(extra_msg=message.text.strip())
    await message.answer("✅ Xabaringiz qabul qilindi! Adminga yuborilmoqda...")
    await send_order_to_admins(message, state, message.from_user)

async def send_order_to_admins(message, state: FSMContext, user):
    data = await state.get_data()
    cursor.execute("SELECT total_orders FROM users WHERE id = ?", (user.id,))
    row = cursor.fetchone()
    orders = row[0] if row else 0
    order_id = str(uuid.uuid4())[:8]
    extra = data.get('extra_msg', '')

    builder = InlineKeyboardBuilder()
    builder.button(text="✉️ Mijozga xabar", callback_data=f"msg_user_{user.id}_{order_id}")
    builder.button(text="🟢 Bajarildi", callback_data=f"done_{user.id}_{order_id}")
    builder.button(text="🔴 Bekor qilish", callback_data=f"cancel_{user.id}_{order_id}")
    builder.button(text="🚫 Tezkor Bloklash", callback_data=f"quickban_{user.id}")
    builder.adjust(2)

    msg_text = (
        f"🛍 *YANGI BUYURTMA!*\n\n"
        f"🆔 ID: `{user.id}`\n"
        f"🌐 User: @{user.username or 'Yoq'}\n"
        f"🎮 O'yin: {data['o_game']}\n"
        f"📋 Tarif: {data['o_desc']}\n"
        f"🎮 O'yin ID: {data['o_player']}\n"
        f"📈 Jami: {orders + 1}-marta\n"
        f"🔖 Order: #{order_id}"
    )
    if extra:
        msg_text += f"\n\n💬 Mijoz xabari: {extra}"

    sent_messages = {}
    for adm in await get_all_admins():
        try:
            sent = await bot.send_photo(
                chat_id=adm,
                photo=data['receipt_photo'],
                caption=msg_text,
                reply_markup=builder.as_markup(),
                parse_mode="Markdown"
            )
            sent_messages[str(adm)] = sent.message_id
        except:
            pass

    cursor.execute("INSERT INTO order_messages (order_id, messages, status, user_id) VALUES (?, ?, 'pending', ?)",
                   (order_id, json.dumps(sent_messages), user.id))
    conn.commit()

    await message.answer(
        f"✅ Buyurtmangiz adminga yuborildi!\n\n🔖 Buyurtma raqami: *#{order_id}*\n⏳ Holat: Kutilmoqda",
        parse_mode="Markdown"
    )
    await state.clear()

# ADMIN — MIJOZGA XABAR YUBORISH
@dp.callback_query(F.data.startswith("msg_user_"))
async def msg_user_start(call: types.CallbackQuery, state: FSMContext):
    if get_admin_level(call.from_user.id) not in [1, 2, 3]: return
    parts = call.data.split("_")
    t_id = int(parts[2])
    order_id = parts[3]
    await state.update_data(msg_target_id=t_id, msg_order_id=order_id)
    await call.message.answer(f"✉️ Mijozga yuboriladigan xabarni yozing:\n(Order: #{order_id})")
    await state.set_state(AdminStates.send_msg_to_user)

@dp.message(AdminStates.send_msg_to_user)
async def msg_user_finish(message: types.Message, state: FSMContext):
    data = await state.get_data()
    t_id = data['msg_target_id']
    order_id = data['msg_order_id']
    try:
        await bot.send_message(
            t_id,
            f"📩 *Admindan xabar* (Order #{order_id}):\n\n{message.text}",
            parse_mode="Markdown"
        )
        await message.answer("✅ Xabar mijozga yuborildi!")
    except:
        await message.answer("❌ Xabar yuborib bo'lmadi.")
    await state.clear()

# BUYURTMA BAJARILDI
@dp.callback_query(F.data.startswith("done_"))
async def order_done(call: types.CallbackQuery):
    parts = call.data.split("_")
    t_id = int(parts[1])
    order_id = parts[2]
    cursor.execute("UPDATE users SET total_orders = total_orders + 1 WHERE id = ?", (t_id,))
    cursor.execute("UPDATE order_messages SET status = 'done' WHERE order_id = ?", (order_id,))
    conn.commit()

    # Mijozga rasmiy chek
    cursor.execute("SELECT messages FROM order_messages WHERE order_id = ?", (order_id,))
    row = cursor.fetchone()
    order_caption = call.message.caption or ""

    now = datetime.now().strftime("%d.%m.%Y %H:%M")
    chek_text = (
        f"✅ *Buyurtmangiz bajarildi!*\n\n"
        f"🔖 Order: *#{order_id}*\n"
        f"📅 Sana: {now}\n\n"
        f"Xarid qilganingiz uchun rahmat! 🎮"
    )

    # Baho so'rash
    rating_builder = InlineKeyboardBuilder()
    for i in range(1, 6):
        rating_builder.button(text=f"{'⭐' * i}", callback_data=f"rate_{order_id}_{i}")
    rating_builder.adjust(5)

    try:
        await bot.send_message(
            t_id,
            chek_text,
            parse_mode="Markdown"
        )
        await bot.send_message(
            t_id,
            "⭐ Xizmatimizga baho bering:",
            reply_markup=rating_builder.as_markup()
        )
    except:
        pass

    # Admin xabarini yangilash
    if row:
        messages = json.loads(row[0])
        new_caption = f"{order_caption}\n\n✅ Bajarildi! (@{call.from_user.username or call.from_user.id})"
        for adm_id, msg_id in messages.items():
            try:
                await bot.edit_message_caption(
                    chat_id=int(adm_id),
                    message_id=msg_id,
                    caption=new_caption,
                    parse_mode="Markdown"
                )
            except:
                pass

# BUYURTMA BEKOR QILISH
@dp.callback_query(F.data.startswith("cancel_"))
async def order_cancel(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) not in [1, 2, 3]: return
    parts = call.data.split("_")
    t_id = int(parts[1])
    order_id = parts[2]
    cursor.execute("UPDATE order_messages SET status = 'cancelled' WHERE order_id = ?", (order_id,))
    conn.commit()

    try:
        await bot.send_message(
            t_id,
            f"❌ *Buyurtmangiz bekor qilindi.*\n\n🔖 Order: *#{order_id}*\n\nSavollar uchun adminga murojaat qiling.",
            parse_mode="Markdown"
        )
    except:
        pass

    cursor.execute("SELECT messages FROM order_messages WHERE order_id = ?", (order_id,))
    row = cursor.fetchone()
    if row:
        messages = json.loads(row[0])
        new_caption = f"{call.message.caption}\n\n❌ Bekor qilindi! (@{call.from_user.username or call.from_user.id})"
        for adm_id, msg_id in messages.items():
            try:
                await bot.edit_message_caption(
                    chat_id=int(adm_id),
                    message_id=msg_id,
                    caption=new_caption,
                    parse_mode="Markdown"
                )
            except:
                pass

# BAHO TIZIMI
@dp.callback_query(F.data.startswith("rate_"))
async def rate_order(call: types.CallbackQuery):
    parts = call.data.split("_")
    order_id = parts[1]
    rating = int(parts[2])

    cursor.execute("INSERT OR IGNORE INTO ratings (order_id, user_id, rating) VALUES (?, ?, ?)",
                   (order_id, call.from_user.id, rating))
    conn.commit()

    stars = "⭐" * rating
    await call.message.edit_text(f"Rahmat! Bahoyingiz: {stars} ({rating}/5) 🙏")

    # Adminga baho xabari
    for adm in await get_all_admins():
        try:
            await bot.send_message(
                adm,
                f"⭐ *Yangi baho!*\n\n"
                f"🔖 Order: #{order_id}\n"
                f"👤 @{call.from_user.username or call.from_user.id}\n"
                f"Baho: {stars} ({rating}/5)",
                parse_mode="Markdown"
            )
        except:
            pass

# QUICKBAN
@dp.callback_query(F.data.startswith("quickban_"))
async def quick_ban(call: types.CallbackQuery):
    if get_admin_level(call.from_user.id) not in [1, 2, 3]: return
    t_id = int(call.data.split("_")[1])
    if t_id == BOSH_ADMIN: return
    cursor.execute("UPDATE users SET status = 'banned' WHERE id = ?", (t_id,))
    conn.commit()
    await call.message.edit_caption(caption=f"{call.message.caption}\n\n🚫 Bloklandi!")

# PROMOKOD TIZIMI
@dp.callback_query(F.data == "adm_add_promo")
async def add_promo_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("🎟 O'yin nomini kiriting:")
    await state.set_state(AdminStates.add_promo_game)

@dp.message(AdminStates.add_promo_game)
async def add_promo_game_handler(message: types.Message, state: FSMContext):
    await state.update_data(p_game=message.text.strip())
    await message.answer("🔑 Promokod matnini kiriting:")
    await state.set_state(AdminStates.add_promo_code)

@dp.message(AdminStates.add_promo_code)
async def add_promo_code_handler(message: types.Message, state: FSMContext):
    await state.update_data(p_code=message.text.strip().upper())
    await message.answer("🔢 Limit (raqam) kiriting:")
    await state.set_state(AdminStates.add_promo_limit)

@dp.message(AdminStates.add_promo_limit)
async def add_promo_limit_handler(message: types.Message, state: FSMContext):
    try:
        data = await state.get_data()
        cursor.execute("INSERT OR REPLACE INTO promocodes VALUES (?, ?, ?, 0)", (data['p_code'], data['p_game'], int(message.text.strip())))
        conn.commit()
        await message.answer("✅ Promokod yaratildi!")
    except:
        await message.answer("❌ Xato!")
    await state.clear()

@dp.message(F.text == "🎟 Promokod")
async def user_promo(message: types.Message):
    subscribed, channel = await check_subscription(message.from_user.id)
    if not subscribed:
        builder = InlineKeyboardBuilder()
        builder.button(text="✅ A'zo bo'ldim", callback_data="check_sub")
        await message.answer(f"❌ Avval kanalga a'zo bo'ling:\n{channel}", reply_markup=builder.as_markup())
        return
    builder = InlineKeyboardBuilder()
    builder.button(text="🔑 Promokod ishlatish", callback_data="u_use_promo")
    builder.button(text="❓ Qayerdan olaman?", callback_data="u_get_promo")
    builder.adjust(1)
    await message.answer("🎟 Promokod bo'limi:", reply_markup=builder.as_markup())

@dp.callback_query(F.data == "u_get_promo")
async def u_get_promo(call: types.CallbackQuery):
    cursor.execute("SELECT val_text, val_photo FROM settings WHERE key = 'qayerdan'")
    row = cursor.fetchone()
    if row[1]:
        await call.message.answer_photo(photo=row[1], caption=row[0])
    else:
        await call.message.answer(row[0])

@dp.callback_query(F.data == "u_use_promo")
async def u_use_promo_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("🔑 Promokodni yozib yuboring:")
    await state.set_state(UserStates.wait_for_promo)

@dp.message(UserStates.wait_for_promo)
async def u_use_promo_finish(message: types.Message, state: FSMContext):
    code_input = message.text.strip().upper()
    cursor.execute("SELECT game, max_uses, used_count FROM promocodes WHERE code = ?", (code_input,))
    row = cursor.fetchone()
    if row:
        game, max_uses, used_count = row
        if used_count < max_uses:
            await state.update_data(p_code=code_input, p_game=game)
            await message.answer(f"✅ Kod to'g'ri! Bu promokod *{game}* uchun.\n\n🆔 O'yindagi *ID raqamingizni* yozing:", parse_mode="Markdown")
            await state.set_state(UserStates.wait_for_promo_id)
        else:
            await message.answer("❌ Bu promokodning limiti tugagan!")
            await state.clear()
    else:
        await message.answer("❌ Bunday promokod topilmadi!")
        await state.clear()

@dp.message(UserStates.wait_for_promo_id)
async def u_promo_id_finish(message: types.Message, state: FSMContext):
    player_data = message.text.strip()
    data = await state.get_data()
    code_input = data['p_code']
    game = data['p_game']
    cursor.execute("UPDATE promocodes SET used_count = used_count + 1 WHERE code = ?", (code_input,))
    conn.commit()
    await message.answer("🎉 Muvaffaqiyatli! Tez orada donat tashlab beriladi.")
    builder = InlineKeyboardBuilder()
    builder.button(text="🟢 Bajarildi", callback_data=f"promodone_{message.from_user.id}")
    builder.adjust(1)
    admin_msg = (
        f"🎟 *PROMOKOD ISHLATILDI!*\n\n"
        f"🆔 TG ID: `{message.from_user.id}`\n"
        f"🌐 User: @{message.from_user.username or 'Yashirin'}\n"
        f"🔑 Kod: `{code_input}`\n"
        f"🎮 O'yin: {game}\n"
        f"🆔 O'yin ID: {player_data}"
    )
    for adm in await get_all_admins():
        try:
            await bot.send_message(chat_id=adm, text=admin_msg, reply_markup=builder.as_markup(), parse_mode="Markdown")
        except:
            pass
    await state.clear()

@dp.callback_query(F.data.startswith("promodone_"))
async def promo_done(call: types.CallbackQuery):
    t_id = int(call.data.split("_")[1])
    try:
        await bot.send_message(t_id, "✅ Promokodingiz bajarildi!")
    except:
        pass
    await call.message.edit_text(f"{call.message.text}\n\n✅ Bajarildi! (@{call.from_user.username or call.from_user.id})")

# BAN / UNBAN
@dp.callback_query(F.data == "adm_ban")
async def ban_s(call: types.CallbackQuery, state: FSMContext):
    if get_admin_level(call.from_user.id) not in [1, 2, 3]: return
    await call.message.answer("🚫 Bloklanadigan ID kiriting:")
    await state.set_state(AdminStates.ban_user)

@dp.message(AdminStates.ban_user)
async def ban_f(message: types.Message, state: FSMContext):
    t_id = int(message.text.strip())
    if t_id != BOSH_ADMIN:
        cursor.execute("UPDATE users SET status = 'banned' WHERE id = ?", (t_id,))
        conn.commit()
        await message.answer("🔴 Bloklandi.")
    await state.clear()

@dp.callback_query(F.data == "adm_unban")
async def unban_start(call: types.CallbackQuery, state: FSMContext):
    if get_admin_level(call.from_user.id) not in [1, 2, 3]: return
    await call.message.answer("🟢 Ochiladigan ID yozing:")
    await state.set_state(AdminStates.unban_user)

@dp.message(AdminStates.unban_user)
async def unban_finish(message: types.Message, state: FSMContext):
    cursor.execute("UPDATE users SET status = 'active' WHERE id = ?", (int(message.text.strip()),))
    conn.commit()
    await message.answer("🟢 Blokdan ochildi.")
    await state.clear()

# QIDIRISH
@dp.callback_query(F.data == "adm_search")
async def search_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("🔍 ID kiriting:")
    await state.set_state(AdminStates.search_user)

@dp.message(AdminStates.search_user)
async def search_finish(message: types.Message, state: FSMContext):
    cursor.execute("SELECT username, status, total_orders FROM users WHERE id = ?", (int(message.text.strip()),))
    row = cursor.fetchone()
    if row:
        await message.answer(f"🆔 ID: `{message.text}`\n👤 Username: @{row[0] or 'Yoq'}\n⚙️ Holat: {row[1]}\n📈 Xaridlar: {row[2]} ta", parse_mode="Markdown")
    else:
        await message.answer("❌ Topilmadi.")
    await state.clear()

# STATISTIKA
@dp.callback_query(F.data == "adm_stats")
async def adm_stats(call: types.CallbackQuery):
    cursor.execute("SELECT COUNT(*) FROM users")
    u_c = cursor.fetchone()[0]
    cursor.execute("SELECT SUM(total_orders) FROM users")
    total = cursor.fetchone()[0] or 0
    cursor.execute("SELECT COUNT(*) FROM users WHERE status = 'banned'")
    banned = cursor.fetchone()[0]
    cursor.execute("SELECT AVG(rating) FROM ratings")
    avg_rating = cursor.fetchone()[0]
    avg_str = f"{avg_rating:.1f}" if avg_rating else "Yo'q"
    await call.message.answer(
        f"📊 *Statistika:*\n\n"
        f"👥 A'zolar: {u_c} ta\n"
        f"📦 Jami donatlar: {total} ta\n"
        f"🚫 Bloklangan: {banned} ta\n"
        f"⭐ O'rtacha baho: {avg_str}",
        parse_mode="Markdown"
    )

# BROADCAST
@dp.callback_query(F.data == "adm_broadcast")
async def broad_start(call: types.CallbackQuery, state: FSMContext):
    await call.message.answer("✉️ Reklama xabarini kiriting:")
    await state.set_state(AdminStates.broadcast_msg)

@dp.message(AdminStates.broadcast_msg)
async def broad_finish(message: types.Message, state: FSMContext):
    cursor.execute("SELECT id FROM users")
    count = 0
    for u in cursor.fetchall():
        try:
            await message.copy_to(chat_id=u[0])
            count += 1
        except:
            pass
    await message.answer(f"📢 {count} kishiga yuborildi!")
    await state.clear()

# FOYDALANUVCHI BO'LIMLARI
@dp.message(F.text == "✍️ Adminga murojaat")
async def user_murojaat(message: types.Message):
    cursor.execute("SELECT val_text FROM settings WHERE key = 'murojaat'")
    await message.answer(cursor.fetchone()[0])

@dp.message(F.text == "👑 Admin Panel")
async def show_admin_panel(message: types.Message):
    if get_admin_level(message.from_user.id) in [1, 2, 3]:
        await message.answer("👑 Admin boshqaruv paneli:", reply_markup=get_admin_menu(message.from_user.id))

# BUYURTMA TARIXI — holat bilan
@dp.message(F.text == "📜 Buyurtma tarixi")
async def history(message: types.Message):
    cursor.execute("SELECT total_orders FROM users WHERE id = ?", (message.from_user.id,))
    row = cursor.fetchone()
    total = row[0] if row else 0

    cursor.execute("SELECT order_id, status FROM order_messages WHERE user_id = ? ORDER BY rowid DESC LIMIT 5", (message.from_user.id,))
    orders = cursor.fetchall()

    text = f"📜 *Buyurtma tarixingiz:*\n\nJami xaridlar: *{total}* marta\n\n"
    if orders:
        text += "🕐 *Oxirgi 5 buyurtma:*\n"
        for o in orders:
            status_icon = "✅" if o[1] == "done" else ("❌" if o[1] == "cancelled" else "⏳")
            status_text = "Bajarildi" if o[1] == "done" else ("Bekor qilindi" if o[1] == "cancelled" else "Kutilmoqda")
            text += f"{status_icon} #{o[0]} — {status_text}\n"

    await message.answer(text, parse_mode="Markdown")

@dp.message(F.text == "🏆 Top donaterlar")
async def top_donat(message: types.Message):
    cursor.execute("SELECT username, total_orders FROM users ORDER BY total_orders DESC LIMIT 10")
    text = "🏆 *TOP 10 Reyting:*\n\n"
    for i, r in enumerate(cursor.fetchall(), 1):
        text += f"{i}. @{r[0] or 'Yashirin'} — {r[1]} ta\n"
    await message.answer(text, parse_mode="Markdown")

# 7 KUN KIRMASA ESLATMA
async def check_inactive_users():
    while True:
        await asyncio.sleep(86400)  # har 24 soatda
        seven_days_ago = (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d %H:%M:%S")
        cursor.execute("SELECT id FROM users WHERE last_seen < ? AND status = 'active'", (seven_days_ago,))
        inactive = cursor.fetchall()
        for u in inactive:
            try:
                await bot.send_message(
                    u[0],
                    "👋 Salom! Sizni sog'indik 😊\n\n🎮 Yangi tariflar va promokodlar qo'shildi!\nQarang 👇",
                )
            except:
                pass

async def main():
    asyncio.create_task(check_inactive_users())
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
