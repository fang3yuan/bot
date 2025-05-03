import requests
import telebot
import uuid
from uuid import uuid4
from user_agent import generate_user_agent


def rest1(user_id,message):

	headers = {
	    'authority': 'www.instagram.com',
	    'accept': '*/*',
	    'accept-language': 'ar-IQ,ar;q=0.9,en-CA;q=0.8,en;q=0.7,en-US;q=0.6',
	    'content-type': 'application/x-www-form-urlencoded',
	    'origin': 'https://www.instagram.com',
	    'referer': 'https://www.instagram.com/accounts/password/reset/',
	    'sec-ch-prefers-color-scheme': 'dark',
	    'sec-ch-ua': '"Not A(Brand";v="8", "Chromium";v="132"',
	    'sec-ch-ua-full-version-list': '"Not A(Brand";v="8.0.0.0", "Chromium";v="132.0.6961.0"',
	    'sec-ch-ua-mobile': '?0',
	    'sec-ch-ua-model': '""',
	    'sec-ch-ua-platform': '"Linux"',
	    'sec-ch-ua-platform-version': '""',
	    'sec-fetch-dest': 'empty',
	    'sec-fetch-mode': 'cors',
	    'sec-fetch-site': 'same-origin',
	    'user-agent': generate_user_agent(),
	    'x-asbd-id': '129477',
	    'x-csrftoken': 'MoKKhcy0MtQHyxInGtndUr',
	    'x-ig-app-id': '936619743392459',
	    'x-ig-www-claim': '0',
	    'x-instagram-ajax': '1019919946',
	    'x-mid': '1ujdauo3vgzs6u1r2s91ofgohk1h2rgm2kmhpc2132ncy3gny20f',
	    'x-requested-with': 'XMLHttpRequest',
	    'x-web-device-id': 'ECF42637-890D-492D-8B40-4CFA749DB5F5',
	    'x-web-session-id': '7ncrkr:ntw37b:pp1uq6',
	}
	
	data = {
	    'email_or_username': user_id,
	}
	
	try:
		res = requests.post(
		    'https://www.instagram.com/api/v1/web/accounts/account_recovery_send_ajax/',
		    headers=headers,
		    data=data,
		).json()
		
		if 'title' in res.text:
			we = res['contact_point']
			ms = f'''📩 تم إرسال رابط إعادة تعيين كلمة المرور إلى حسابك على إنستجرام!
✉️ البريد المرتبط: 
[{we}]
			'''
			bot.reply_to(message,ms)
		else:
			gh = '''⚠️ البريد الإلكتروني أو المستخدم غير موجود!
🛠️ يرجى التحقق من البيانات المدخلة
			'''
			bot.reply_to(message,gh)
	except:
		mn = '''❌ هناك خطأ في الاتصال أو الإنترنت!
🌐 تحقق من الاتصال وحاول مرة أخرى
		'''
		bot.reply_to(message,mn)

def rest(user_id,message):
	url = "https://i.instagram.com/api/v1/accounts/send_password_reset/"
	
	payload = {
	'ig_sig_key_version': "4",
	'user_email': user_id,
	'device_id': str(uuid.uuid4),
	}
	
	headers = {
	  'User-Agent': "Instagram 113.0.0.39.122 Android (30/11; 320dpi; 720x1339; realme; RMX3261; RMX3261; S19610AA1; en_CA)",
	  'Connection': "Keep-Alive",
	  'Accept-Encoding': "gzip",
	  'Cookie2': "$Version=1",
	  'Accept-Language': "en-CA, en-US",
	  'X-IG-Connection-Type': "WIFI",
	  'X-IG-Capabilities': "AQ==",
	  'Cookie': "mid=Z4pqeQABAAHARa5XXmMPD5DG3OUA; csrftoken=Fz0IDmyOhyfPHAinkGtwy5RjqpwCfDcK"
	}
	try:
		res = requests.post(url, data=payload, headers=headers).json()
		
		if 'obfuscated_email' in res:
			se = res['obfuscated_email']
			ms = f'''📩 تم إرسال رابط إعادة تعيين كلمة المرور إلى حسابك على إنستجرام!
✉️ البريد المرتبط: 
[{se}]
			'''
			bot.reply_to(message,ms)
		elif 'rate_limit_error' in res:
			md = '''انتظر 20 دقيقة ⏳ ثم حاول مجددًا.
🕒 الصبر مفتاح الفرج!
'''
			bot.reply_to(message,md)
		else:
			rest1(user_id,message)
	except:
		rest1(user_id,message)

TOKEN = "7278031174:AAHqJjLVrBGNFMmMnZvamKAQg8mWZNtabf0"  #توكن هنا 
bot = telebot.TeleBot(TOKEN)


@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "📬 أدخل البريد الإلكتروني أو اسم المستخدم لحسابك الآن!")

@bot.message_handler(func=lambda message: True)
def check_id(message):
    user_id = message.text.strip()
    rest(user_id,message)

bot.polling()
