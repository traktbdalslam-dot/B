import os
import logging
import asyncio
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
from openai import AsyncOpenAI

# إعداد التسجيل
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# التوكنات من متغيرات البيئة
TELEGRAM_TOKEN = os.environ.get('BOT_TOKEN')
OPENAI_API_KEY = os.environ.get('OPENAI_API_KEY')

# تهيئة عميل OpenAI
client = AsyncOpenAI(api_key=OPENAI_API_KEY)

# ذاكرة المحادثة لكل مستخدم
user_conversations = {}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    welcome = f"""مرحباً {user.first_name}! 👋

أنا **ناصر** - مطور بوتات ومبرمج ذكاء اصطناعي.

يمكنني:
• الإجابة على أسئلتك البرمجية
• مساعدتك في تطوير البوتات
• شرح مفاهيم الذكاء الاصطناعي
• كتابة أكواد بلغات مختلفة

اكتب لي أي سؤال! 💻"""
    
    await update.message.reply_text(welcome)
    # بدء محادثة جديدة
    user_conversations[user.id] = [
        {"role": "system", "content": "أنت ناصر، مطور بوتات ومبرمج محترف في الذكاء الاصطناعي. تتحدث بلهجة ودودة ومهنية. أنت عربي وتجاوب بالعربية ما لم يطلب غير ذلك."}
    ]

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = """🎯 **أوامر البوت:**
/start - بدء المحادثة
/help - عرض هذه المساعدة
/clear - مسح ذاكرة المحادثة
/code [اللغة] [الوصف] - توليد كود برمجي
/about - معلومات عني

💡 **مثال:**
`/code python حساب المعدل`
`/code javascript زر تفاعلي`

🔧 **المميزات:**
- ذاكرة محادثة ذكية
- دعم 50+ لغة برمجية
- إجابة على أسئلة تقنية
- كتابة توثيق وأكواد"""
    
    await update.message.reply_text(help_text, parse_mode='Markdown')

async def clear_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if user_id in user_conversations:
        user_conversations[user_id] = [
            {"role": "system", "content": "أنت ناصر، مطور بوتات ومبرمج محترف في الذكاء الاصطناعي."}
        ]
    await update.message.reply_text("✅ تم مسح ذاكرة المحادثة. ابدأ حديثاً جديداً!")

async def about_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    about_text = """👨‍💻 **نبذة عن ناصر:**
مطور بوتات تليجرام متخصص في:
- برمجة بوتات الذكاء الاصطناعي
- تطوير واجهات برمجية API
- استضافة وتنفيذ السيرفرات
- تعلم الآلة ومعالجة اللغات الطبيعية

📚 **المهارات:**
Python • JavaScript • AI/ML • APIs • Docker

🚀 **رؤيتي:**
تبسيط التقنية وجعل الذكاء الاصطناعي في متناول الجميع"""
    
    await update.message.reply_text(about_text)

async def generate_code(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args or len(context.args) < 2:
        await update.message.reply_text("⚠️ استخدم: `/code python وصف الكود`")
        return
    
    language = context.args[0]
    description = " ".join(context.args[1:])
    
    prompt = f"""اكتب كود {language} لـ: {description}

المتطلبات:
1. كود نظيف وواضح
2. تعليقات توضيحية بالعربية
3. معالجة الأخطاء
4. أمثلة على الاستخدام"""

    try:
        await update.message.reply_chat_action(action="typing")
        
        response = await client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "أنت مساعد برمجي محترف. اكتب أكواد عالية الجودة مع شرح عربي."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=1000
        )
        
        code = response.choices[0].message.content
        await update.message.reply_text(f"```{language}\n{code}\n```", parse_mode='Markdown')
        
    except Exception as e:
        logger.error(f"Error in generate_code: {e}")
        await update.message.reply_text("❌ حدث خطأ في توليد الكود. حاول مرة أخرى.")

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user_message = update.message.text
    
    # تجهيز المحادثة
    if user_id not in user_conversations:
        user_conversations[user_id] = [
            {"role": "system", "content": "أنت ناصر، مطور بوتات ومبرمج محترف في الذكاء الاصطناعي. ترد بلغة عربية واضحة ودقيقة."}
        ]
    
    # إضافة رسالة المستخدم
    user_conversations[user_id].append({"role": "user", "content": user_message})
    
    # تقليل المحادثة إذا طالت (آخر 10 رسائل)
    if len(user_conversations[user_id]) > 20:
        user_conversations[user_id] = [user_conversations[user_id][0]] + user_conversations[user_id][-10:]
    
    try:
        # إظهار حالة الكتابة
        await update.message.reply_chat_action(action="typing")
        
        # الحصول على رد من الذكاء الاصطناعي
        response = await client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=user_conversations[user_id],
            temperature=0.8,
            max_tokens=800,
            top_p=0.95,
            frequency_penalty=0.2,
            presence_penalty=0.3
        )
        
        ai_response = response.choices[0].message.content
        
        # إضافة رد الذكاء للذاكرة
        user_conversations[user_id].append({"role": "assistant", "content": ai_response})
        
        # إرسال الرد (بتقسيم إذا كان طويلاً)
        if len(ai_response) > 4000:
            parts = [ai_response[i:i+4000] for i in range(0, len(ai_response), 4000)]
            for part in parts:
                await update.message.reply_text(part)
                await asyncio.sleep(0.5)
        else:
            await update.message.reply_text(ai_response)
            
    except Exception as e:
        logger.error(f"Error in handle_message: {e}")
        await update.message.reply_text("⚠️ حدث خطأ في المعالجة. حاول مرة أخرى.")

async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    logger.error(f"Update {update} caused error {context.error}")
    if update and update.message:
        await update.message.reply_text("❌ حدث خطأ غير متوقع. الرجاء المحاولة لاحقاً.")

def main():
    # التحقق من التوكنات
    if not TELEGRAM_TOKEN or not OPENAI_API_KEY:
        logger.error("Missing tokens! Set BOT_TOKEN and OPENAI_API_KEY")
        return
    
    # إنشاء التطبيق
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    
    # إضافة handlers
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("clear", clear_chat))
    app.add_handler(CommandHandler("about", about_command))
    app.add_handler(CommandHandler("code", generate_code))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    # إضافة معالج الأخطاء
    app.add_error_handler(error_handler)
    
    # بدء البوت
    logger.info("🤖 بوت ناصر يعمل...")
    print("=" * 50)
    print("🎯 البوت يعمل باسم: ناصر - مطور بوتات")
    print("💡 المميزات: ذكاء اصطناعي - ذاكرة محادثة - توليد أكواد")
    print("=" * 50)
    
    app.run_polling(allowed_updates=Update.ALL_TYPES, drop_pending_updates=True)

if __name__ == '__main__':
    main()
