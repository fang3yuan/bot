from instagrapi import Client
import time
from datetime import datetime, timedelta

# بيانات تسجيل الدخول إلى Instagram
INSTAGRAM_USERNAME = "au5ae"
INSTAGRAM_PASSWORD = "Qazwsx@12"

# تسجيل الدخول إلى حساب Instagram
def login_to_instagram(username, password):
    cl = Client()
    try:
        cl.login(username, password)
        print("تم تسجيل الدخول إلى Instagram بنجاح!")
        return cl
    except Exception as e:
        print(f"فشل تسجيل الدخول: {e}")
        return None

# بيانات اللاعبين
players_data = {}
admin_usernames = ["au5ae"]  # ضع أسماء الإدمن هنا
salary_cooldown = timedelta(minutes=10)  # مدة الانتظار بين الرواتب

# مستويات الوظائف
job_levels = {
    1: {"name": "متدرب", "salary": 100},
    2: {"name": "موظف", "salary": 200},
    3: {"name": "مدير", "salary": 400},
    4: {"name": "رئيس قسم", "salary": 800},
    5: {"name": "مدير تنفيذي", "salary": 1600},
}

# قائمة الأوامر
commands_list = """
الأوامر المتاحة:
1. إنشاء - لإنشاء حساب بنكي.
2. رصيد - لمعرفة رصيدك الحالي.
3. راتب - للمطالبة براتبك.
4. ترقيات - لمعرفة مستوى وظيفتك الحالي.
5. منح [@username] [المبلغ] - (للإدمن فقط) لمنح مبلغ لأي لاعب.
"""

# إنشاء حساب بنكي
def create_account(username):
    if username in players_data:
        return "لديك حساب بنكي بالفعل!"
    players_data[username] = {
        "balance": 0,
        "last_salary_time": datetime.min,
        "job_level": 1,
        "salary_claims": 0,
    }
    return f"تم إنشاء حسابك بنجاح! وظيفتك الحالية: {job_levels[1]['name']}"

# المطالبة بالراتب
def claim_salary(username):
    player = players_data.get(username)
    if not player:
        return "يجب عليك إنشاء حساب بنكي أولاً!"
    
    now = datetime.now()
    if now - player["last_salary_time"] < salary_cooldown:
        remaining_time = salary_cooldown - (now - player["last_salary_time"])
        minutes, seconds = divmod(remaining_time.seconds, 60)
        return f"يمكنك المطالبة براتبك بعد {minutes} دقيقة و {seconds} ثانية."

    # إضافة الراتب إلى الرصيد
    job_level = player["job_level"]
    salary = job_levels[job_level]["salary"]
    player["balance"] += salary
    player["last_salary_time"] = now
    player["salary_claims"] += 1

    # الترقية إذا طالب بالراتب 10 مرات
    if player["salary_claims"] >= 10:
        if job_level < len(job_levels):
            player["job_level"] += 1
            player["salary_claims"] = 0
            new_job = job_levels[player["job_level"]]["name"]
            return f"تمت ترقيتك إلى وظيفة: {new_job}! راتبك الجديد هو {job_levels[player['job_level']]['salary']}."

    return f"تم إضافة {salary} إلى حسابك. وظيفتك الحالية: {job_levels[job_level]['name']}."

# منح الأموال من قبل الإدمن
def grant_money(admin_username, target_username, amount):
    if admin_username not in admin_usernames:
        return "هذا الأمر مخصص للإدمن فقط!"
    
    if target_username not in players_data:
        return "اللاعب المستهدف ليس لديه حساب بنكي!"
    
    players_data[target_username]["balance"] += amount
    return f"تم منح {amount} إلى {target_username} من قبل {admin_username}."

# عرض الرصيد
def check_balance(username):
    player = players_data.get(username)
    if not player:
        return "يجب عليك إنشاء حساب بنكي أولاً!"
    return f"رصيدك الحالي هو: {player['balance']}."

# عرض الترقية
def show_job_level(username):
    player = players_data.get(username)
    if not player:
        return "يجب عليك إنشاء حساب بنكي أولاً!"
    job_level = player["job_level"]
    return f"وظيفتك الحالية: {job_levels[job_level]['name']}، راتبك: {job_levels[job_level]['salary']}."

# الرد على الرسائل
def handle_message(username, message):
    args = message.split()
    command = args[0].lower()
    
    if command == "إنشاء":
        return create_account(username)
    elif command == "راتب":
        return claim_salary(username)
    elif command == "رصيد":
        return check_balance(username)
    elif command == "ترقيات":
        return show_job_level(username)
    elif command == "منح":
        if len(args) < 3:
            return "صيغة الأمر غير صحيحة! استخدم: منح [@username] [المبلغ]"
        target_username = args[1].lstrip("@")
        try:
            amount = int(args[2])
            return grant_money(username, target_username, amount)
        except ValueError:
            return "المبلغ يجب أن يكون رقمًا صحيحًا!"
    else:
        return f"الأمر غير معروف! إليك قائمة بالأوامر:\n{commands_list}"

# تشغيل النظام
def main():
    # تسجيل الدخول إلى Instagram
    cl = login_to_instagram(INSTAGRAM_USERNAME, INSTAGRAM_PASSWORD)
    if not cl:
        return

    print("النظام يعمل... انتظر الرسائل!")

    while True:
        # جلب الرسائل الواردة
        messages = cl.direct_threads()
        for thread in messages:
            for message in thread.messages:
                sender = message.user.username
                text = message.text
                if sender and text:
                    print(f"رسالة من {sender}: {text}")
                    response = handle_message(sender, text)
                    cl.direct_send(response, thread.id)
        
        time.sleep(10)

if __name__ == "__main__":
    main()
