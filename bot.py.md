import asyncio  
import logging  
import sqlite3  
from datetime import datetime  
from aiogram import Bot, Dispatcher, F, types  
from aiogram.filters import Command  
from aiogram.fsm.context import FSMContext  
from aiogram.fsm.state import State, StatesGroup  
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder  
  
# ВАШ УТВЕРЖДЕННЫЙ ТОКЕН  
TOKEN = "8837415861:AAEoAgfnTokniMS6uh_uu7gX9E46SmpnSw4"  
  
logging.basicConfig(level=logging.INFO)  
bot = Bot(token=TOKEN)  
dp = Dispatcher()  
  
# Тарифная сетка для МАН 10т (без КФС)  
TARIFS = {  
    "1 рейс": 1500,  
    "2 рейс": 1600,  
    "3 рейс": 1700,  
    "Рейс «Заморозка»": 1700,  
    "Рейс «АТБ»": 1700,  
    "Рейс «Дубечня»": 800,  
    "Стоянка > 5ч": 800,  
    "Перемещение 1-3ч": 500,  
    "Перемещение > 3ч": 1000,  
    "Региональный (за 1 км)": 5.0  
}  
  
def init_db():  
    conn = sqlite3.connect("salary.db")  
    cursor = conn.cursor()  
    cursor.execute("""  
        CREATE TABLE IF NOT EXISTS entries (  
            id INTEGER PRIMARY KEY AUTOINCREMENT,  
            user_id INTEGER,  
            date TEXT,  
            work_type TEXT,  
            quantity REAL,  
            earnings REAL,  
            advance REAL DEFAULT 0  
        )  
    """)  
    cursor.execute("""  
        CREATE TABLE IF NOT EXISTS active_trips (  
            user_id INTEGER PRIMARY KEY,  
            start_odometer REAL  
        )  
    """)  
    conn.commit()  
    conn.close()  
  
class Form(StatesGroup):  
    entering_start_odo = State()  
    entering_finish_odo = State()  
    entering_advance = State()  
  
def main_menu(user_id):  
    conn = sqlite3.connect("salary.db")  
    cursor = conn.cursor()  
    cursor.execute("SELECT start_odometer FROM active_trips WHERE user_id = ?", (user_id,))  
    active_trip = cursor.fetchone()  
    conn.close()  
  
    builder = ReplyKeyboardBuilder()  
    builder.button(text="➕ Добавить работу")  
      
    if active_trip:  
        builder.button(text="🏁 Финиш регионального рейса")  
    else:  
        builder.button(text="🚚 Старт регионального рейса")  
          
    builder.button(text="💸 Записать аванс")  
    builder.button(text="📊 Статистика за месяц")  
    builder.adjust(2, 2)  
    return builder.as_markup(resize_keyboard=True)  
  
@dp.message(Command("start"))  
async def cmd_start(message: types.Message):  
    await message.answer(  
        "Привет! Я бот-калькулятор зарплаты для вашего **МАН -10т**.\n"  
        "Региональные рейсы считаются по одометру (Старт/Финиш).",  
        reply_markup=main_menu(message.from_user.id)  
    )  
  
@dp.message(F.text == "➕ Добавить работу")  
async def add_work(message: types.Message):  
    builder = InlineKeyboardBuilder()  
    for name in TARIFS.keys():  
        if "Региональный" not in name:  
            builder.button(text=name, callback_data=f"work:{name}")  
    builder.adjust(2)  
    await message.answer("Выберите выполненную работу:", reply_markup=builder.as_markup())  
  
@dp.callback_query(F.data.startswith("work:"))  
async def process_work_choice(callback: types.CallbackQuery):  
    work_type = callback.data.split(":")[1]  
    today = datetime.now().strftime("%Y-%m-%d")  
    earnings = TARIFS[work_type]  
      
    conn = sqlite3.connect("salary.db")  
    cursor = conn.cursor()  
    cursor.execute(  
        "INSERT INTO entries (user_id, date, work_type, quantity, earnings) VALUES (?, ?, ?, ?, ?)",  
        (callback.from_user.id, today, work_type, 1, earnings)  
    )  
    conn.commit()  
    conn.close()  
  
    await callback.message.answer(f"✅ Записано: **{work_type}** (+{earnings} грн.)")  
    await callback.answer()  
  
@dp.message(F.text == "🚚 Старт регионального рейса")  
async def start_regional(message: types.Message, state: FSMContext):  
    await state.set_state(Form.entering_start_odo)  
    await message.answer("Введите начальные показания одометра (км):")  
  
@dp.message(Form.entering_start_odo)  
async def process_start_odo(message: types.Message, state: FSMContext):  
    try:  
        start_val = float(message.text.replace(",", "."))  
        conn = sqlite3.connect("salary.db")  
        cursor = conn.cursor()  
        cursor.execute("INSERT OR REPLACE INTO active_trips (user_id, start_odometer) VALUES (?, ?)",   
                       (message.from_user.id, start_val))  
        conn.commit()  
        conn.close()  
  
        await message.answer(f"🏁 Рейс открыт. Стартовый пробег: **{start_val} км**.",   
                             reply_markup=main_menu(message.from_user.id))  
        await state.clear()  
    except ValueError:  
        await message.answer("Пожалуйста, введите корректное число.")  
  
@dp.message(F.text == "🏁 Финиш регионального рейса")  
async def finish_regional(message: types.Message, state: FSMContext):  
    conn = sqlite3.connect("salary.db")  
    cursor = conn.cursor()  
    cursor.execute("SELECT start_odometer FROM active_trips WHERE user_id = ?", (message.from_user.id,))  
    row = cursor.fetchone()  
    conn.close()  
  
    if not row:  
        await message.answer("У вас нет запущенных рейсов.", reply_markup=main_menu(message.from_user.id))  
        return  
  
    await state.set_state(Form.entering_finish_odo)  
    await message.answer(f"Начальный пробег был: **{row[0]} км**.\nВведите конечные показания одометра:")  
  
@dp.message(Form.entering_finish_odo)  
async def process_finish_odo(message: types.Message, state: FSMContext):  
    try:  
        finish_val = float(message.text.replace(",", "."))  
          
        conn = sqlite3.connect("salary.db")  
        cursor = conn.cursor()  
        cursor.execute("SELECT start_odometer FROM active_trips WHERE user_id = ?", (message.from_user.id,))  
        start_val = cursor.fetchone()[0]  
  
        if finish_val < start_val:  
            await message.answer("Конечный пробег не может быть меньше стартового. Проверьте цифры:")  
            conn.close()  
            return  
  
        km = finish_val - start_val  
        earnings = km * TARIFS["Региональный (за 1 км)"]  
        today = datetime.now().strftime("%Y-%m-%d")  
  
        cursor.execute(  
            "INSERT INTO entries (user_id, date, work_type, quantity, earnings) VALUES (?, ?, ?, ?, ?)",  
            (message.from_user.id, today, "Региональный (км)", km, earnings)  
        )  
        cursor.execute("DELETE FROM active_trips WHERE user_id = ?", (message.from_user.id,))  
        conn.commit()  
        conn.close()  
  
        await message.answer(  
            f"✅ Пробег посчитан!\n"  
            f"• Пройдено: **{km:.1f} км**\n"  
            f"• Заработано: **{earnings:.2f} грн.**",   
            reply_markup=main_menu(message.from_user.id)  
        )  
        await state.clear()  
    except ValueError:  
        await message.answer("Пожалуйста, введите корректное число.")  
  
@dp.message(F.text == "💸 Записать аванс")  
async def add_advance(message: types.Message, state: FSMContext):  
    await state.set_state(Form.entering_advance)  
    await message.answer("Введите сумму полученного аванса (грн.):")  
  
@dp.message(Form.entering_advance)  
async def process_advance(message: types.Message, state: FSMContext):  
    try:  
        amount = float(message.text.replace(",", "."))  
        today = datetime.now().strftime("%Y-%m-%d")  
  
        conn = sqlite3.connect("salary.db")  
        cursor = conn.cursor()  
        cursor.execute(  
            "INSERT INTO entries (user_id, date, work_type, quantity, earnings, advance) VALUES (?, ?, ?, ?, ?, ?)",  
            (message.from_user.id, today, "Аванс", 0, 0, amount)  
        )  
        conn.commit()  
        conn.close()  
  
        await message.answer(f"✅ Выдан аванс: **{amount} грн.**", reply_markup=main_menu(message.from_user.id))  
        await state.clear()  
    except ValueError:  
        await message.answer("Пожалуйста, введите сумму цифрами.")  
  
@dp.message(F.text == "📊 Статистика за month" or F.text == "📊 Статистика за месяц")  
async def show_stats(message: types.Message):  
    current_month = datetime.now().strftime("%Y-%m")  
      
    conn = sqlite3.connect("salary.db")  
    cursor = conn.cursor()  
    cursor.execute("""  
        SELECT work_type, SUM(quantity), SUM(earnings), SUM(advance)   
        FROM entries   
        WHERE user_id = ? AND date LIKE ?   
        GROUP BY work_type  
    """, (message.from_user.id, f"{current_month}%"))  
      
    rows = cursor.fetchall()  
    conn.close()  
  
    if not rows:  
        await message.answer("За этот месяц записей пока нет.", reply_markup=main_menu(message.from_user.id))  
        return  
  
    total_earned = 0  
    total_advance = 0  
    report_lines = [f"📊 **Отчет за текущий месяц:**\n"]  
  
    for row in rows:  
        work_type, quantity, earnings, advance = row  
        total_earned += earnings  
        total_advance += advance  
        if earnings > 0:  
            if "км" in work_type:  
                report_lines.append(f"• {work_type}: {quantity:.1f} км — {earnings:.2f} грн.")  
            else:  
                report_lines.append(f"• {work_type}: {int(quantity)} шт. — {earnings:.2f} грн.")  
        if advance > 0:  
            report_lines.append(f"• Получено авансов: {advance:.2f} грн.")  
  
    rest_to_pay = total_earned - total_advance  
      
    report_lines.append(f"\n💰 **Итого грязыми:** {total_earned:.2f} грн.")  
    report_lines.append(f"💸 **Всего авансов:** {total_advance:.2f} грн.")  
    report_lines.append(f"💵 **Остаток к выдаче:** {rest_to_pay:.2f} грн.")  
  
    await message.answer("\n".join(report_lines), reply_markup=main_menu(message.from_user.id))  
  
async def main():  
    init_db()  
    await dp.start_polling(bot)  
  
if __name__ == "__main__":  
    asyncio.run(main())  
