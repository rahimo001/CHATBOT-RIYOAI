# CHATBOT-RIYOAI
RiyoAI is an intelligent virtual assistant designed to simplify your digital life. Powered by advanced AI, RiyoAI can answer questions, automate tasks, analyze data, create content, and assist with coding or research — all in real time. Whether you’re a student, developer, entrepreneur, or content creator.



openai>=1.0.0
anthropic>=0.25.0
google-generativeai>=0.3.0
requests>=2.31.0
pillow>=10.0.0
python-multipart>=0.0.6
fastapi>=0.104.0
uvicorn>=0.24.0
aiohttp>=3.9.0
python-dotenv>=1.0.0
numpy>=1.24.0

BOT.PY   FOR LUNCH ..............................

..........
.........
.........
import logging
import sqlite3
import asyncio
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, LabeledPrice
from telegram.ext import (
    Application, CommandHandler, MessageHandler, 
    filters, CallbackContext, CallbackQueryHandler, PreCheckoutQueryHandler
)
import google.generativeai as genai
import os
from dotenv import load_dotenv
from typing import Dict, Tuple, Optional, Any


# تحميل المتغيرات من ملف .env
load_dotenv()


# إعداد التسجيل
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)


# إعدادات البوت
BOT_TOKEN: str = os.getenv('BOT_TOKEN', 'YOUR_BOT_TOKEN_HERE')
GEMINI_API_KEY: str = os.getenv('GEMINI_API_KEY', 'YOUR_GEMINI_API_KEY_HERE')
ADMIN_IDS: list = [int(x) for x in os.getenv('ADMIN_IDS', '').split(',') if x]


# تأكد من وجود المفاتيح
if BOT_TOKEN == 'YOUR_BOT_TOKEN_HERE':
    logger.error("❌ يرجى إعداد BOT_TOKEN في ملف .env")
    exit(1)


if GEMINI_API_KEY == 'YOUR_GEMINI_API_KEY_HERE':
    logger.error("❌ يرجى إعداد GEMINI_API_KEY في ملف .env")
    exit(1)


# حدود الاستخدام
USER_LIMITS: Dict[str, Dict[str, int]] = {
    'free': {'daily': 100, 'weekly': 70, 'monthly': 300},
    'daily': {'daily': 500, 'weekly': 350, 'monthly': 1500},
    'weekly': {'daily': 50, 'weekly': 300, 'monthly': 1200},
    'monthly': {'daily': 50, 'weekly': 300, 'monthly': 1200}
}


# أسعار الاشتراكات بالنجوم (XTR)
SUBSCRIPTION_PRICES: Dict[str, int] = {
    'daily': 10,      # 10 نجوم
    'weekly': 50,     # 50 نجمة
    'monthly': 180    # 180 نجمة
}


WARNING_DAYS_BEFORE_EXPIRY: int = 2
try:
    genai.configure(api_key=GEMINI_API_KEY)
    logger.info("✅ تم تهيئة  API بنجاح")
except Exception as e:
    logger.error(f"❌ خطأ في تهيئة api: {e}")
    exit(1)

# قاعدة البيانات
class Database:
    def __init__(self):
        try:
            self.conn: sqlite3.Connection = sqlite3.connect('users.db', check_same_thread=False)
            self.create_tables()
            self.ensure_columns()
            logger.info("✅ تم الاتصال بقاعدة البيانات بنجاح")
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في قاعدة البيانات: {e}")
            raise

    def create_tables(self):
        cursor = self.conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                first_name TEXT,
                daily_used INTEGER DEFAULT 0,
                weekly_used INTEGER DEFAULT 0,
                monthly_used INTEGER DEFAULT 0,
                subscription_type TEXT DEFAULT 'free',
                subscription_expiry DATE,
                last_reset_date DATE,
                last_weekly_reset DATE,
                last_monthly_reset DATE,
                expiry_warning_sent INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS transactions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                subscription_type TEXT,
                stars_paid INTEGER,
                payment_id TEXT,
                transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                status TEXT DEFAULT 'completed',
                FOREIGN KEY (user_id) REFERENCES users (user_id)
            )
        ''')
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS question_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                question TEXT,
                answer TEXT,
                asked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users (user_id)
            )
        ''')
        self.conn.commit()
        logger.info("✅ تم إنشاء الجداول بنجاح")

    def ensure_columns(self):
        """ضمان وجود الأعمدة المطلوبة بعد إنشاء الجدول"""
        cursor = self.conn.cursor()
        cursor.execute("PRAGMA table_info(users)")
        existing_cols = [row[1] for row in cursor.fetchall()]
        columns = [
            ('last_reset_date', "DATE"),
            ('last_weekly_reset', "DATE"),
            ('last_monthly_reset', "DATE"),
            ('expiry_warning_sent', "INTEGER DEFAULT 0"),
        ]
        for col, definition in columns:
            if col not in existing_cols:
                cursor.execute(f"ALTER TABLE users ADD COLUMN {col} {definition}")
                logger.info(f"✅ تم إضافة العمود {col} بنجاح.")
        self.conn.commit()
        logger.info("✅ تم تحديث الأعمدة بنجاح")


    # باقي الدوال كما هي من النسخة السابقة (من غير تعديل)
    # فقط تأكد أن جميع عمليات جلب المستخدمين وتحديثهم تستدعي الأعمدة حسب الترتيب الصحيح
    # ولو زدت الأعمدة احرص على تعديل ترتيب الفهارس في الدالة get_user

    # ... (ضع كافة الدوال المساعدة والدوال الأخرى الخاصة بالكلاس هنا كما في كودك الأصلي) ...


# تابع كتابة بقية البرنامج واستخدم نفس مفهوم تأكيد الأعمدة والتعامل مع جلب وتحديث بيانات المستخدمين.


        
        # جدول المستخدمين
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                first_name TEXT,
                daily_used INTEGER DEFAULT 0,
                weekly_used INTEGER DEFAULT 0,
                monthly_used INTEGER DEFAULT 0,
                subscription_type TEXT DEFAULT 'free',
                subscription_expiry DATE,
                last_reset_date DATE DEFAULT CURRENT_DATE,
                last_weekly_reset DATE DEFAULT CURRENT_DATE,
                last_monthly_reset DATE DEFAULT CURRENT_DATE,
                expiry_warning_sent INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        # جدول المعاملات
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS transactions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                subscription_type TEXT,
                stars_paid INTEGER,
                payment_id TEXT,
                transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                status TEXT DEFAULT 'completed',
                FOREIGN KEY (user_id) REFERENCES users (user_id)
            )
        ''')
        
        # جدول سجل الأسئلة
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS question_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                question TEXT,
                answer TEXT,
                asked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users (user_id)
            )
        ''')
        
        self.conn.commit()
        logger.info("✅ تم إنشاء الجداول بنجاح")
    
    def get_user(self, user_id: int) -> Optional[Dict[str, Any]]:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
            user = cursor.fetchone()
            
            if user:
                return {
                    'user_id': user[0],
                    'username': user[1],
                    'first_name': user[2],
                    'daily_used': user[3],
                    'weekly_used': user[4],
                    'monthly_used': user[5],
                    'subscription_type': user[6],
                    'subscription_expiry': user[7],
                    'last_reset_date': user[8],
                    'last_weekly_reset': user[9],
                    'last_monthly_reset': user[10],
                    'expiry_warning_sent': user[11]
                }
            return None
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في جلب بيانات المستخدم: {e}")
            return None
    
    def create_user(self, user_id: int, username: Optional[str], first_name: Optional[str]) -> None:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            cursor.execute('''
                INSERT OR IGNORE INTO users (
                    user_id, username, first_name, 
                    last_reset_date, last_weekly_reset, last_monthly_reset
                )
                VALUES (?, ?, ?, DATE('now'), DATE('now'), DATE('now'))
            ''', (user_id, username, first_name))
            self.conn.commit()
            logger.info(f"✅ تم إنشاء/تحديث المستخدم: {user_id}")
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في إنشاء المستخدم: {e}")
    
    def update_usage(self, user_id: int) -> None:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            
            # إعادة تعيين العدادات حسب الحاجة
            cursor.execute('''
                UPDATE users 
                SET daily_used = CASE 
                        WHEN last_reset_date < DATE('now') THEN 0 
                        ELSE daily_used 
                    END,
                    weekly_used = CASE 
                        WHEN last_weekly_reset < DATE('now', '-7 days') THEN 0 
                        ELSE weekly_used 
                    END,
                    monthly_used = CASE 
                        WHEN last_monthly_reset < DATE('now', '-30 days') THEN 0 
                        ELSE monthly_used 
                    END,
                    last_reset_date = CASE 
                        WHEN last_reset_date < DATE('now') THEN DATE('now') 
                        ELSE last_reset_date 
                    END,
                    last_weekly_reset = CASE 
                        WHEN last_weekly_reset < DATE('now', '-7 days') THEN DATE('now') 
                        ELSE last_weekly_reset 
                    END,
                    last_monthly_reset = CASE 
                        WHEN last_monthly_reset < DATE('now', '-30 days') THEN DATE('now') 
                        ELSE last_monthly_reset 
                    END
                WHERE user_id = ?
            ''', (user_id,))
            
            # زيادة العدادات
            cursor.execute('''
                UPDATE users 
                SET daily_used = daily_used + 1,
                    weekly_used = weekly_used + 1, 
                    monthly_used = monthly_used + 1
                WHERE user_id = ?
            ''', (user_id,))
            
            self.conn.commit()
            logger.info(f"✅ تم تحديث استخدام المستخدم: {user_id}")
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في تحديث الاستخدام: {e}")
    
    def update_subscription(self, user_id: int, subscription_type: str, payment_id: str = None) -> None:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            
            # حساب تاريخ الانتهاء
            expiry_date: datetime.date = datetime.now().date()
            if subscription_type == 'daily':
                expiry_date += timedelta(days=1)
            elif subscription_type == 'weekly':
                expiry_date += timedelta(weeks=1)
            elif subscription_type == 'monthly':
                expiry_date += timedelta(days=30)
            elif subscription_type == 'free':
                expiry_date = None
            
            cursor.execute('''
                UPDATE users 
                SET subscription_type = ?, 
                    subscription_expiry = ?,
                    expiry_warning_sent = 0
                WHERE user_id = ?
            ''', (subscription_type, expiry_date, user_id))
            
            self.conn.commit()
            logger.info(f"✅ تم تحديث اشتراك المستخدم {user_id} إلى {subscription_type}")
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في تحديث الاشتراك: {e}")
    
    def add_transaction(self, user_id: int, subscription_type: str, stars_paid: int, payment_id: str) -> None:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            cursor.execute('''
                INSERT INTO transactions (user_id, subscription_type, stars_paid, payment_id)
                VALUES (?, ?, ?, ?)
            ''', (user_id, subscription_type, stars_paid, payment_id))
            self.conn.commit()
            logger.info(f"✅ تم إضافة معاملة للمستخدم {user_id}")
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في إضافة المعاملة: {e}")
    
    def log_question(self, user_id: int, question: str, answer: str) -> None:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            cursor.execute('''
                INSERT INTO question_log (user_id, question, answer)
                VALUES (?, ?, ?)
            ''', (user_id, question, answer))
            self.conn.commit()
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في تسجيل السؤال: {e}")
    
    def get_usage_history(self, user_id: int, days: int = 7) -> list:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            cursor.execute('''
                SELECT question, asked_at FROM question_log
                WHERE user_id = ? AND asked_at >= datetime('now', '-' || ? || ' days')
                ORDER BY asked_at DESC
                LIMIT 10
            ''', (user_id, days))
            return cursor.fetchall()
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في جلب سجل الاستخدام: {e}")
            return []
    
    def mark_expiry_warning_sent(self, user_id: int) -> None:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            cursor.execute('UPDATE users SET expiry_warning_sent = 1 WHERE user_id = ?', (user_id,))
            self.conn.commit()
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في تحديث حالة التحذير: {e}")
    
    def get_users_with_expiring_subscriptions(self) -> list:
        try:
            cursor: sqlite3.Cursor = self.conn.cursor()
            warning_date = (datetime.now() + timedelta(days=WARNING_DAYS_BEFORE_EXPIRY)).date()
            cursor.execute('''
                SELECT user_id, subscription_type, subscription_expiry 
                FROM users
                WHERE subscription_type != 'free' 
                AND subscription_expiry <= ? 
                AND expiry_warning_sent = 0
            ''', (warning_date,))
            return cursor.fetchall()
        except sqlite3.Error as e:
            logger.error(f"❌ خطأ في جلب المستخدمين ذوي الاشتراكات المنتهية: {e}")
            return []

# تهيئة قاعدة البيانات
try:
    db: Database = Database()
except Exception as e:
    logger.error(f"❌ فشل في تهيئة قاعدة البيانات: {e}")
    exit(1)

# وظائف المساعدة
def can_user_ask_question(user_data: Optional[Dict[str, Any]]) -> Tuple[bool, str]:
    """التحقق مما إذا كان يمكن للمستخدم طرح سؤال"""
    if not user_data:
        return False, "لم يتم العثور على بيانات المستخدم"
    
    # التحقق من انتهاء الاشتراك
    if user_data['subscription_expiry']:
        try:
            expiry_date: datetime.date = datetime.strptime(user_data['subscription_expiry'], '%Y-%m-%d').date()
            if datetime.now().date() > expiry_date:
                db.update_subscription(user_data['user_id'], 'free')
                user_data['subscription_type'] = 'free'
                logger.info(f"✅ تم تحويل المستخدم {user_data['user_id']} إلى الاشتراك المجاني")
        except ValueError as e:
            logger.error(f"❌ خطأ في تحويل التاريخ: {e}")
    
    subscription_type: str = user_data['subscription_type']
    limits = USER_LIMITS.get(subscription_type, USER_LIMITS['free'])
    
    # التحقق من الحدود
    if user_data['daily_used'] >= limits['daily']:
        return False, f"لقد استخدمت جميع أسئلتك اليومية ({limits['daily']} سؤال). 🎯\n\nيمكنك ترقية اشتراكك باستخدام /subscription للحصول على المزيد!"
    
    if user_data['weekly_used'] >= limits['weekly']:
        return False, f"لقد استخدمت جميع أسئلتك الأسبوعية ({limits['weekly']} سؤال). 🎯"
    
    if user_data['monthly_used'] >= limits['monthly']:
        return False, f"لقد استخدمت جميع أسئلتك الشهرية ({limits['monthly']} سؤال). 🎯"
    
    return True, ""

async def ask_gemini(question: str, user_name: str = "صديقي") -> str:
    """طرح سؤال على Google Gemini مع تخصيص الرد"""
    try:
        model = genai.GenerativeModel('gemini-2.0-flash')
        
        # تخصيص الـ prompt لتحسين تجربة المستخدم
        custom_prompt = f"""أنت مساعد ذكي ومفيد ولطيف. اسم المستخدم هو {user_name}.
        
قم بالإجابة على السؤال التالي بطريقة:
- واضحة ومباشرة
- ودية ومحترمة
- منظمة ومنسقة
- شاملة ومفيدة

السؤال: {question}

ملاحظة: إذا كان السؤال يتطلب تنسيقاً، استخدم رموز تعبيرية مناسبة ونقاط منظمة."""
        
        response = model.generate_content(custom_prompt)
        return response.text.strip()
    
    except Exception as e:
        logger.error(f"❌ Gemini Error: {e}")
        import random
        alternative_responses = [
            "عذراً، واجهت صعوبة في معالجة سؤالك. يرجى المحاولة مرة أخرى. 🙏",
            "حالياً نواجه بعض الصعوبات التقنية. جرب سؤالك بعد قليل! ⚙️",
            "خدمة الذكاء الاصطناعي غير متاحة مؤقتاً. نعمل على حل المشكلة. 🔧",
            "لم أتمكن من معالجة سؤالك الآن. يرجى المحاولة لاحقاً. ⏳"
        ]
        return random.choice(alternative_responses)

async def send_expiry_warnings(application: Application):
    """إرسال تحذيرات انتهاء الاشتراك"""
    while True:
        try:
            await asyncio.sleep(3600)  # كل ساعة
            
            users = db.get_users_with_expiring_subscriptions()
            for user_id, sub_type, expiry in users:
                try:
                    expiry_date = datetime.strptime(expiry, '%Y-%m-%d').date()
                    days_left = (expiry_date - datetime.now().date()).days
                    
                    warning_text = f"""
⚠️ تنبيه انتهاء الاشتراك

عزيزي المستخدم،
اشتراكك من نوع **{sub_type.upper()}** سينتهي خلال **{days_left} يوم**.

📅 تاريخ الانتهاء: {expiry_date.strftime('%Y-%m-%d')}

لتجديد اشتراكك والاستمرار في الاستمتاع بخدماتنا، استخدم الأمر /subscription

💡 نقدر دعمك المستمر! 🙏
                    """
                    
                    await application.bot.send_message(chat_id=user_id, text=warning_text)
                    db.mark_expiry_warning_sent(user_id)
                    logger.info(f"✅ تم إرسال تحذير انتهاء الاشتراك للمستخدم {user_id}")
                    
                except Exception as e:
                    logger.error(f"❌ خطأ في إرسال تحذير للمستخدم {user_id}: {e}")
            
        except Exception as e:
            logger.error(f"❌ خطأ في مهمة التحذيرات: {e}")



# 📨 دالة إرسال إشعار للأدمن
async def notify_admins(context: CallbackContext, message: str):
    """إرسال رسالة إلى جميع الأدمن"""
    ADMIN_IDS = [123456789, 987654321]  # ⚠️ غيّر هذه الأرقام إلى IDات الأدمن الحقيقيين
    for admin_id in ADMIN_IDS:
        try:
            await context.bot.send_message(chat_id=admin_id, text=message, parse_mode="Markdown")
        except Exception as e:
            logger.error(f"❌ فشل إرسال إشعار للأدمن {admin_id}: {e}")



































# 🧮 أمر لوحة تحكم الأدمن
async def admin_dashboard(update: Update, context: CallbackContext) -> None:
    """عرض لوحة الإحصائيات للأدمن"""
    admin_id = update.effective_user.id

    # ✅ ضع هنا ID الأدمن الحقيقي
    ADMIN_IDS = [7363617171]  # استبدل بالأرقام الصحيحة

    if admin_id not in ADMIN_IDS:
        await update.message.reply_text("🚫 ليس لديك صلاحية للوصول إلى لوحة التحكم.")
        return

    try:
        # الاتصال بقاعدة البيانات
        conn = sqlite3.connect("bot_data.db")
        cur = conn.cursor()

        # 👥 عدد المستخدمين
        cur.execute("SELECT COUNT(*) FROM users")
        total_users = cur.fetchone()[0]

        # 💰 إجمالي المبيعات
        cur.execute("SELECT COALESCE(SUM(stars_paid), 0) FROM transactions")
        total_stars = cur.fetchone()[0]

        # 📆 عدد الاشتراكات المفعّلة والمنتهية
        cur.execute("SELECT COUNT(*) FROM subscriptions WHERE active=1")
        active_subs = cur.fetchone()[0]
        cur.execute("SELECT COUNT(*) FROM subscriptions WHERE active=0")
        expired_subs = cur.fetchone()[0]

        # 🧠 المستخدمون النشطون اليوم
        today = datetime.now().strftime("%Y-%m-%d")
        cur.execute("SELECT COUNT(DISTINCT user_id) FROM messages WHERE date_sent LIKE ?", (f"{today}%",))
        active_today = cur.fetchone()[0]

        # 📊 عدد الأسئلة المرسلة اليوم
        cur.execute("SELECT COUNT(*) FROM messages WHERE date_sent LIKE ?", (f"{today}%",))
        questions_today = cur.fetchone()[0]

        conn.close()

        # 🧾 إنشاء النص
        dashboard_text = f"""
📊 **لوحة تحكم الأدمن**

👥 المستخدمون: {total_users}
💰 إجمالي النجوم المدفوعة: {total_stars}
📆 الاشتراكات:
   ✅ مفعّلة: {active_subs}
   ❌ منتهية: {expired_subs}
🧠 المستخدمون النشطون اليوم: {active_today}
💬 الأسئلة المرسلة اليوم: {questions_today}

⏰ آخر تحديث: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
        """

        await update.message.reply_text(dashboard_text, parse_mode="Markdown")

    except Exception as e:
        logger.error(f"❌ خطأ في لوحة الأدمن: {e}")
        await update.message.reply_text("⚠️ حدث خطأ أثناء تحميل لوحة التحكم.")












































# أوامر البوت
async def start(update: Update, context: CallbackContext) -> None:
    """أمر البدء"""
    user = update.effective_user
    db.create_user(user.id, user.username, user.first_name)
    
    welcome_text: str = f"""
مرحباً {user.first_name}! 👋

🤖 أنا بوت الذكاء الاصطناعي المدعوم من riyoai. يمكنني الإجابة على أسئلتك بذكاء!

🎯 **الخطة المجانية:**
• {USER_LIMITS['free']['daily']} أسئلة يومياً
• {USER_LIMITS['free']['weekly']} أسئلة أسبوعياً
• {USER_LIMITS['free']['monthly']} أسئلة شهرياً

💎 **خطط الاشتراك المميزة:**

🌙 يومي ({SUBSCRIPTION_PRICES['daily']} نجمة):
• {USER_LIMITS['daily']['daily']} سؤال يومياً

📅 أسبوعي ({SUBSCRIPTION_PRICES['weekly']} نجمة):
• {USER_LIMITS['weekly']['weekly']} سؤال أسبوعياً

📆 شهري ({SUBSCRIPTION_PRICES['monthly']} نجمة):
• {USER_LIMITS['monthly']['monthly']} سؤال شهرياً

📋 **الأوامر المتاحة:**
/subscription - شراء أو ترقية الاشتراك
/usage - عرض استخدامك الحالي
/history - عرض سجل أسئلتك
/cancel - إلغاء الاشتراك الحالي

💡 فقط اكتب سؤالك وسأجيب عليه باستخدام تقنية riyoaiالمتقدمة!
    """
    
    await update.message.reply_text(welcome_text, parse_mode='Markdown')

async def usage(update: Update, context: CallbackContext) -> None:
    """عرض الاستخدام الحالي"""
    user = update.effective_user
    user_data = db.get_user(user.id)
    
    if not user_data:
        await update.message.reply_text("❌ لم يتم العثور على بياناتك. استخدم /start للبدء.")
        return
    
    subscription_type = user_data['subscription_type']
    limits = USER_LIMITS[subscription_type]
    
    # حساب الأسئلة المتبقية
    daily_remaining = limits['daily'] - user_data['daily_used']
    weekly_remaining = limits['weekly'] - user_data['weekly_used']
    monthly_remaining = limits['monthly'] - user_data['monthly_used']
    
    # معلومات انتهاء الاشتراك
    expiry_info = ""
    if user_data['subscription_expiry']:
        try:
            expiry_date = datetime.strptime(user_data['subscription_expiry'], '%Y-%m-%d').date()
            days_left = (expiry_date - datetime.now().date()).days
            expiry_info = f"\n📅 ينتهي الاشتراك بعد: {days_left} يوم ({expiry_date.strftime('%Y-%m-%d')})"
        except:
            pass
    
    usage_text = f"""
📊 **استخدامك الحالي:**

🎯 نوع الاشتراك: **{subscription_type.upper()}**{expiry_info}

📝 **الاستخدام اليومي:**
• المستخدم: {user_data['daily_used']}/{limits['daily']}
• المتبقي: {daily_remaining} سؤال

📅 **الاستخدام الأسبوعي:**
• المستخدم: {user_data['weekly_used']}/{limits['weekly']}
• المتبقي: {weekly_remaining} سؤال

📆 **الاستخدام الشهري:**
• المستخدم: {user_data['monthly_used']}/{limits['monthly']}
• المتبقي: {monthly_remaining} سؤال

💎 لترقية اشتراكك، استخدم /subscription
📜 لعرض سجل أسئلتك، استخدم /history
    """
    
    await update.message.reply_text(usage_text, parse_mode='Markdown')

async def history(update: Update, context: CallbackContext) -> None:
    """عرض سجل الأسئلة"""
    user = update.effective_user
    questions = db.get_usage_history(user.id, days=7)
    
    if not questions:
        await update.message.reply_text("📭 لا يوجد لديك سجل أسئلة حتى الآن.")
        return
    
    history_text = "📜 **سجل أسئلتك الأخيرة:**\n\n"
    for idx, (question, asked_at) in enumerate(questions, 1):
        asked_date = datetime.strptime(asked_at, '%Y-%m-%d %H:%M:%S')
        history_text += f"{idx}. {question[:50]}...\n   ⏰ {asked_date.strftime('%Y-%m-%d %H:%M')}\n\n"
    
    await update.message.reply_text(history_text, parse_mode='Markdown')

async def subscription(update: Update, context: CallbackContext) -> None:
    """عرض خطط الاشتراك"""
    keyboard = [
        [InlineKeyboardButton(f"🌙 يومي - {SUBSCRIPTION_PRICES['daily']} ⭐", callback_data="sub_daily")],
        [InlineKeyboardButton(f"📅 أسبوعي - {SUBSCRIPTION_PRICES['weekly']} ⭐", callback_data="sub_weekly")],
        [InlineKeyboardButton(f"📆 شهري - {SUBSCRIPTION_PRICES['monthly']} ⭐", callback_data="sub_monthly")],
        [InlineKeyboardButton("🔄 تحديث الحالة", callback_data="refresh_status")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    text = f"""
💎 **خطط الاشتراك المتاحة:**

🌙 **اشتراك يومي:**
• السعر: {SUBSCRIPTION_PRICES['daily']} نجمة ⭐
• {USER_LIMITS['daily']['daily']} سؤال يومياً

📅 **اشتراك أسبوعي:**  
• السعر: {SUBSCRIPTION_PRICES['weekly']} نجمة ⭐
• {USER_LIMITS['weekly']['weekly']} سؤال أسبوعياً

📆 **اشتراك شهري:**
• السعر: {SUBSCRIPTION_PRICES['monthly']} نجمة ⭐
• {USER_LIMITS['monthly']['monthly']} سؤال شهرياً

💡 اختر الخطة المناسبة لك وادفع باستخدام نجوم تيليجرام!
    """
    
    await update.message.reply_text(text, reply_markup=reply_markup, parse_mode='Markdown')

async def cancel_subscription(update: Update, context: CallbackContext) -> None:
    """إلغاء الاشتراك"""
    user = update.effective_user
    user_data = db.get_user(user.id)
    
    if not user_data or user_data['subscription_type'] == 'free':
        await update.message.reply_text("❌ ليس لديك اشتراك مدفوع حالياً.")
        return
    
    # إلغاء الاشتراك
    db.update_subscription(user.id, 'free')
    
    cancel_text = f"""
✅ **تم إلغاء اشتراكك بنجاح**

تم تحويلك إلى الخطة المجانية.

🎯 يمكنك الآن استخدام:
• {USER_LIMITS['free']['daily']} أسئلة يومياً
• {USER_LIMITS['free']['weekly']} أسئلة أسبوعياً
• {USER_LIMITS['free']['monthly']} أسئلة شهرياً

💡 يمكنك الاشتراك مرة أخرى في أي وقت باستخدام /subscription

نأمل أن نراك قريباً! 👋
    """
    
    await update.message.reply_text(cancel_text, parse_mode='Markdown')

async def handle_callback(update: Update, context: CallbackContext) -> None:
    """معالجة الضغطات على الأزرار"""
    query = update.callback_query
    await query.answer()
    
    user = query.from_user
    user_data = db.get_user(user.id)
    
    if query.data == "refresh_status":
        if user_data:
            limits = USER_LIMITS[user_data['subscription_type']]
            remaining_daily = limits['daily'] - user_data['daily_used']
            remaining_weekly = limits['weekly'] - user_data['weekly_used']
            remaining_monthly = limits['monthly'] - user_data['monthly_used']
            
            status_text = f"""
🔄 **الحالة الحالية:**

📊 نوع الاشتراك: **{user_data['subscription_type'].upper()}**

الأسئلة المتبقية:
• يومياً: {remaining_daily}
• أسبوعياً: {remaining_weekly}
• شهرياً: {remaining_monthly}
            """
            await query.edit_message_text(status_text, parse_mode='Markdown')
        else:
            await query.edit_message_text("❌ لم يتم العثور على بياناتك. استخدم /start للبدء.")
        return
    
    # معالجة شراء الاشتراكات
    subscription_map = {
        'sub_daily': ('daily', SUBSCRIPTION_PRICES['daily'], 'يومي'),
        'sub_weekly': ('weekly', SUBSCRIPTION_PRICES['weekly'], 'أسبوعي'), 
        'sub_monthly': ('monthly', SUBSCRIPTION_PRICES['monthly'], 'شهري')
    }
    
    if query.data in subscription_map:
        sub_type, stars, sub_name = subscription_map[query.data]
        
        # إنشاء الفاتورة
        title = f"اشتراك {sub_name}"
        description = f"اشتراك {sub_name} - {USER_LIMITS[sub_type]['daily']} سؤال"
        payload = f"{sub_type}_subscription_{user.id}"
        prices = [LabeledPrice(label="XTR", amount=stars)]
        
        try:
            await context.bot.send_invoice(
                chat_id=query.message.chat_id,
                title=title,
                description=description,
                payload=payload,
                provider_token="",  # للنجوم، يترك فارغاً
                currency="XTR",
                prices=prices
            )
            await query.edit_message_text(f"✅ تم إرسال فاتورة الدفع. يرجى إكمال عملية الدفع.")
        except Exception as e:
            logger.error(f"❌ خطأ في إرسال الفاتورة: {e}")
            await query.edit_message_text("❌ عذراً، حدث خطأ في إنشاء الفاتورة. يرجى المحاولة لاحقاً.")

async def precheckout_callback(update: Update, context: CallbackContext) -> None:
    """معالجة استعلام ما قبل الدفع"""
    query = update.pre_checkout_query
    await query.answer(ok=True)


async def successful_payment_callback(update: Update, context: CallbackContext) -> None:
    """معالجة الدفع الناجح"""
    payment = update.message.successful_payment
    user = update.effective_user

    try:
        payload = payment.invoice_payload  # مثال: "daily_subscription_123456"
        parts = payload.split("_")
        subscription_type = parts[0] if len(parts) > 0 else "daily"
        user_id = int(parts[-1])
        stars_paid = payment.total_amount

        # تحديث الاشتراك
        db.update_subscription(user_id, subscription_type, payment_id=payment.provider_payment_charge_id)

        # تسجيل المعاملة
        db.add_transaction(
            user_id=user_id,
            subscription_type=subscription_type,
            stars_paid=stars_paid,
            payment_id=payment.provider_payment_charge_id
        )

        # رسالة تأكيد
        success_text = f"""
💎 **تم تفعيل اشتراكك بنجاح!**

🎯 نوع الاشتراك: **{subscription_type.upper()}**
⭐ عدد النجوم المدفوعة: {stars_paid}
📅 تاريخ الانتهاء سيتم تحديده تلقائياً حسب نوع الاشتراك.

استمتع بطرح المزيد من الأسئلة الذكية الآن! 🤖  
استخدم /usage لمتابعة استخدامك الحالي.
        """

        await update.message.reply_text(success_text, parse_mode='Markdown')
        logger.info(f"✅ تم تأكيد عملية الدفع للمستخدم {user_id} ({subscription_type})")
                # 🔔 إشعار الأدمن بالدفع
        admin_message = f"""
💰 *عملية دفع جديدة!*
👤 المستخدم: [{user.first_name}](tg://user?id={user_id})
🪪 ID: `{user_id}`
🎯 نوع الاشتراك: *{subscription_type.upper()}*
⭐ النجوم المدفوعة: {stars_paid}
📅 الوقت: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
        """
        await notify_admins(context, admin_message)

    except Exception as e:
        logger.error(f"❌ خطأ أثناء معالجة الدفع الناجح: {e}")
        await update.message.reply_text("❌ حدث خطأ أثناء معالجة عملية الدفع. يرجى التواصل مع الدعم.")



async def handle_message(update: Update, context: CallbackContext) -> None:
    """معالجة الرسائل العادية (الأسئلة)"""
    user = update.effective_user
    message_text = update.message.text
    
    # تخطي إذا كان الأمر
    if message_text.startswith('/'):
        return
    
    # إنشاء المستخدم إذا لم يكن موجوداً
    db.create_user(user.id, user.username, user.first_name)
    user_data = db.get_user(user.id)
    
    # التحقق من إمكانية السؤال
    can_ask, error_message = can_user_ask_question(user_data)
    if not can_ask:
        await update.message.reply_text(error_message)
        return
    
    # إظهار أن البوت يكتب
    await update.message.chat.send_action(action="typing")
    
    try:
        # طرح السؤال على Gemini
        response = await ask_gemini(message_text, user.first_name or "صديقي")
        
        # تسجيل السؤال والجواب
        db.log_question(user.id, message_text, response)
        
        # تحديث الاستخدام
        db.update_usage(user.id)
        user_data = db.get_user(user.id)
        
        limits = USER_LIMITS[user_data['subscription_type']]
        daily_remaining = limits['daily'] - user_data['daily_used']
        weekly_remaining = limits['weekly'] - user_data['weekly_used']
        monthly_remaining = limits['monthly'] - user_data['monthly_used']
        
        # إضافة معلومات الاستخدام للرد
        usage_info = f"\n\n━━━━━━━━━━━━━━━\n📊 **الأسئلة المتبقية:**\n• اليوم: {daily_remaining} | الأسبوع: {weekly_remaining} | الشهر: {monthly_remaining}"
        full_response = response + usage_info
        
        # تحديد طول الرد وتقسيمه إذا لزم الأمر
        if len(full_response) > 4096:
            # تقسيم الرد إلى أجزاء
            parts = [full_response[i:i+4096] for i in range(0, len(full_response), 4096)]
            for part in parts:
                await update.message.reply_text(part, parse_mode='Markdown')
        else:
            await update.message.reply_text(full_response, parse_mode='Markdown')
        
    except Exception as e:
        logger.error(f"❌ Error: {e}")
        await update.message.reply_text("❌ عذراً، حدث خطأ في معالجة سؤالك. يرجى المحاولة مرة أخرى.")

async def post_init(application: Application) -> None:
    """تهيئة ما بعد البدء - لإطلاق المهام الخلفية"""
    # إطلاق مهمة التحذيرات في الخلفية
    asyncio.create_task(send_expiry_warnings(application))
    logger.info("✅ تم إطلاق مهمة إشعارات انتهاء الاشتراك")

def main():
    """الدالة الرئيسية"""
    # إنشاء تطبيق البوت
    application = Application.builder().token(BOT_TOKEN).build()

    # أوامر البوت
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("admin_dashboard", admin_dashboard))

    application.add_handler(CommandHandler("usage", usage))
    application.add_handler(CommandHandler("history", history))
    application.add_handler(CommandHandler("subscription", subscription))
    application.add_handler(CommandHandler("cancel", cancel_subscription))

    # معالجة الدفع
    application.add_handler(PreCheckoutQueryHandler(precheckout_callback))
    application.add_handler(MessageHandler(filters.SUCCESSFUL_PAYMENT, successful_payment_callback))
    # المعالجة العامة
    application.add_handler(CallbackQueryHandler(handle_callback))
                        
    # تشغيل البوت والتحذيرات الدورية



    # بدء البوت
    logger.info("🤖 البوت يعمل الآن مع API ونظام الدفع بالنجوم...")
    print("🤖 البوت يعمل الآن مع  API ونظام الدفع بالنجوم...")
    
    # تشغيل البوت
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        logger.info("🛑 إيقاف البوت بواسطة المستخدم...")
        print("🛑 إيقاف البوت بواسطة المستخدم...")
    except Exception as e:
        logger.error(f"❌ خطأ غير متوقع: {e}")
        print(f"❌ خطأ غير متوقع: {e}")
    finally:
        logger.info("✅ تم إيقاف البوت")
        print("✅ تم إيقاف البوت")

