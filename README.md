import os
import json
import random
import time
from datetime import datetime, timedelta
from instagrapi import Client
from instagrapi.exceptions import LoginRequired

# Import games
from games import games_manager

# Constants
BASE_SALARY = 10000
POINTS_TO_MONEY_RATE = 50  # Each point is worth 50 currency
MIN_TIP = 3000
MAX_TIP = 15000
MIN_TREASURE = 20000
MAX_TREASURE = 200000
MIN_LOAN = 50000
MAX_LOAN = 200000
WHEEL_COST = 5000000
CAR_PRICE = 1000000
STOCK_PRICE = 1000000
COOLDOWNS = {
    'salary': 600,
    'tip': 600,
    'steal': 600,
    'invest': 600,
    'luck': 1200,
    'treasure': 1800,
    'wheel': 300
}

# Data paths
DATA_DIR = "data"
USERS_FILE = os.path.join(DATA_DIR, "users.json")
GROUPS_FILE = os.path.join(DATA_DIR, "groups.json")
CUSTOM_RESPONSES_FILE = os.path.join(DATA_DIR, "custom_responses.json")

# Create data directory if not exists
os.makedirs(DATA_DIR, exist_ok=True)

class BankBot:
    def __init__(self):
        self.cl = Client()
        self.load_data()
        self.admin_username = "fpg.x"
        self.processed_messages = set()  # Track processed message IDs
        self.device_settings = {
            "app_version": "219.0.0.12.117",
            "android_version": 29,
            "android_release": "10",
            "dpi": "480dpi",
            "resolution": "1080x2268",
            "manufacturer": "Samsung",
            "device": "SM-G988U",
            "model": "Galaxy S20 Ultra",
            "cpu": "qcom",
            "version_code": "314665256"
        }

    def load_data(self):
        """Load all data from JSON files"""
        try:
            with open(USERS_FILE, 'r') as f:
                self.users = json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            self.users = {}

        try:
            with open(GROUPS_FILE, 'r') as f:
                self.groups = json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            self.groups = {}

        try:
            with open(CUSTOM_RESPONSES_FILE, 'r') as f:
                self.custom_responses = json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            self.custom_responses = {}

    def save_data(self):
        """Save all data to JSON files"""
        with open(USERS_FILE, 'w') as f:
            json.dump(self.users, f, indent=4)

        with open(GROUPS_FILE, 'w') as f:
            json.dump(self.groups, f, indent=4)

        with open(CUSTOM_RESPONSES_FILE, 'w') as f:
            json.dump(self.custom_responses, f, indent=4)

    def login(self, username, password):
        """Login to Instagram with device emulation"""
        try:
            # Set device settings for real device emulation
            self.cl.set_device(self.device_settings)

            # Load session if exists
            session_file = f"{username}_session.json"
            if os.path.exists(session_file):
                self.cl.load_settings(session_file)

            self.cl.login(username, password)

            # Save session
            self.cl.dump_settings(session_file)
            return True
        except Exception as e:
            print(f"Login failed: {e}")
            return False

    def listen_to_messages(self):
        """Listen to group messages and process commands"""
        while True:
            try:
                print("Checking for new messages...")
                # Get recent activity (including group messages)
                threads = self.cl.direct_threads()
                print(f"Found {len(threads)} threads")

                for thread in threads:
                    if not thread.is_group:  # Skip private chats
                        continue

                    thread_id = thread.id
                    try:
                        messages = self.cl.direct_messages(thread_id=thread_id, amount=10)
                        print(f"Processing {len(messages)} messages in thread {thread_id}")

                        # Process each message in reverse order (newest first)
                        for message in reversed(messages):
                            if message.user_id == self.cl.user_id or message.id in self.processed_messages:
                                continue
                            
                            print(f"Processing message {message.id}")
                            self.processed_messages.add(message.id)
                            
                            if len(self.processed_messages) > 1000:
                                self.processed_messages = set(list(self.processed_messages)[-1000:])

                            try:
                                user_info = self.cl.user_info(message.user_id)
                                is_reply = False
                                reply_message = None
                                
                                if hasattr(message, 'text') and message.text:
                                    text = message.text.strip()
                                    print(f"Received command: {text} from {user_info.username}")
                                else:
                                    continue
                                    
                                if hasattr(message, 'replied_to'):
                                    reply_message = message.replied_to
                                    is_reply = True

                                self.process_message(
                                    text=text,
                                    username=user_info.username,
                                    thread_id=thread_id,
                                    is_reply=is_reply,
                                    reply_message=reply_message
                                )
                            except LoginRequired:
                                print("Session expired, attempting to relogin...")
                                self.login(username, password)
                            except Exception as e:
                                print(f"Error processing individual message: {e}")
                                continue
                    except Exception as e:
                        print(f"Error processing thread {thread_id}: {e}")
                        continue

                time.sleep(2)

            except LoginRequired:
                print("Session expired, attempting to relogin...")
                self.login(username, password)
            except Exception as e:
                print(f"Major error in message loop: {e}")
                time.sleep(10)

    def process_message(self, text, username, thread_id, is_reply, reply_message=None):
        """Process a received message and execute commands"""
        # Balance command
        if text == "رصيدي" or text == "حسابي":
            self.show_balance(username, thread_id)
            return
            
        # Check for دارين responses
        if "دارين" in text:
            if username not in self.darin_count:
                self.darin_count[username] = 0
            self.darin_count[username] += 1
            
            if self.darin_count[username] > 3:
                self.send_message(thread_id, "وجع شتريد صرعتني")
            else:
                responses = [
                    "عيون دارين العسليات",
                    "ها شتريد",
                    "قلب دارين شو تريد"
                ]
                self.send_message(thread_id, random.choice(responses))
            return

        # Process add response state
        if username in self.adding_response_state:
            if self.process_adding_response(username, text, thread_id):
                return

        # Check for custom responses
        if text in self.custom_responses:
            self.send_message(thread_id, self.custom_responses[text])
            return

        # Normalize text for command processing
        text = text.strip().lower()

        # Track group activity
        if thread_id not in self.groups:
            self.groups[thread_id] = {
                "name": "",
                "command_count": 0,
                "users": {}
            }

        self.groups[thread_id]["command_count"] += 1

        if username not in self.groups[thread_id]["users"]:
            self.groups[thread_id]["users"][username] = 0
        self.groups[thread_id]["users"][username] += 1

        # Create user account if not exists
        if username not in self.users:
            self.users[username] = {
                "balance": 0,
                "properties": {"cars": 0, "diamonds": 0},
                "last_used": {},
                "stocks": 0,
                "debt": 0,
                "in_jail": False,
                "x2": False,
                "x2_expiry": None,
                "rank": "عضو",
                "group_name": "",
                "created_at": datetime.now().isoformat()
            }

        # Check if user is in jail
        if self.users[username]["in_jail"] and not text.startswith(("سداد ديوني", "سداد ديونه بالرد")):
            self.send_message(thread_id, f"@{username} أنت في السجن! ادفع ديونك أولاً باستخدام 'سداد ديوني'.")
            return

        # Process commands
        if text == "انشاء حساب بنكي":
            self.send_message(thread_id, f"@{username} لديك حساب بنكي بالفعل!")

        elif text == "راتب":
            self.give_salary(username, thread_id)

        elif text == "بخشيش":
            self.give_tip(username, thread_id)

        elif text.startswith("زرف @"):
            target = text.split("@")[1].strip()
            self.steal_money(username, target, thread_id)

        elif text.startswith("استثمار"):
            amount = text.split()[1]
            try:
                amount = int(amount)
                self.invest(username, amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: استثمار [المبلغ]")

        elif text.startswith("حظ"):
            amount = text.split()[1]
            try:
                amount = int(amount)
                self.luck(username, amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: حظ [المبلغ]")

        elif text == "كنز":
            self.give_treasure(username, thread_id)

        elif text == "العجله":
            self.spin_wheel(username, thread_id)

        elif text == "ممتلكاتي":
            self.show_properties(username, thread_id)

        elif text.startswith("شراء") and "سيارة" in text:
            try:
                amount = int(text.split()[1])
                self.buy_property(username, "cars", amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: شراء [العدد] سيارة")

        elif text.startswith("بيع") and "سيارة" in text:
            try:
                amount = int(text.split()[1])
                self.sell_property(username, "cars", amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: بيع [العدد] سيارة")

        elif text.startswith("اهداء") and "سيارة" in text and is_reply:
            try:
                amount = int(text.split()[1])
                # Get target user from replied message
                target_msg = self.cl.direct_message(thread_id=thread_id, message_id=message.id)
                replied_to = getattr(target_msg, 'replied_to', None)
                if replied_to and hasattr(replied_to, 'id'):
                    replied_msg = self.cl.direct_message(thread_id=thread_id, message_id=replied_to.id)
                    if hasattr(replied_msg, 'user') and hasattr(replied_msg.user, 'username'):
                        target = replied_msg.user.username
                        self.gift_property(username, target, "cars", amount, thread_id)
                    else:
                        self.send_message(thread_id, "خطأ في العثور على المستخدم المستهدف")
                else:
                    self.send_message(thread_id, "لم يتم الرد على الرسالة")

            except (ValueError, IndexError, Exception) as e:
                print(f"Error in gift_property: {e}")
                self.send_message(thread_id, "استخدم: اهداء [العدد] سيارة بالرد على رسالة الشخص")


        elif text.startswith("شراء اسهم"):
            try:
                amount = int(text.split()[2])
                self.buy_stocks(username, amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: شراء اسهم [العدد]")

        elif text.startswith("بيع اسهم"):
            try:
                amount = int(text.split()[2])
                self.sell_stocks(username, amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: بيع اسهم [العدد]")

        elif text == "سعر الاسهم":
            self.show_stock_price(username, thread_id)

        elif text == "قرض":
            self.take_loan(username, thread_id)

        elif text == "سجني":
            self.check_jail(username, thread_id)

        elif text == "ديوني":
            self.check_debt(username, thread_id)

        elif text == "سداد ديوني":
            self.pay_debt(username, thread_id)

        elif text.startswith("سداد ديونه @"):
            try:
                target = text.split("@")[1].strip()
                self.pay_debt_for(username, target, thread_id)
            except Exception as e:
                print(f"Error in pay_debt_for: {e}")
                self.send_message(thread_id, "استخدم: سداد ديونه @username")

        elif text in ["توب القروبات", "توب المجموعات"]:
            self.show_group_top(thread_id)

        elif text in ["توب المتفاعلين", "توب الاعضاء"]:
            self.show_user_top(thread_id)

        elif text.startswith("تحويل @"):
            try:
                parts = text.split("@")[1].split()
                target = parts[0].strip()
                amount = int(parts[1])
                self.transfer_money(username, target, amount, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: تحويل @username المبلغ")

        elif text == "اضف رد":
            self.start_adding_response(username, thread_id)

        elif text == "الردود":
            self.show_responses(thread_id)

        elif text.startswith("حذف رد"):
            try:
                response_num = int(text.split("حذف رد")[1].strip())
                self.delete_response(response_num, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: حذف رد [رقم الرد]")

        elif text.startswith("طرد @") and self.is_group_admin(username, thread_id):
            try:
                target = text.split("@")[1].strip()
                self.kick_user(target, thread_id, username)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: طرد @username")

        elif text.startswith("ترقية @") and self.is_group_admin(username, thread_id):
            try:
                target = text.split("@")[1].strip()
                self.promote_user(target, thread_id, username)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: ترقية @username")

        elif text.startswith("تنزيل @") and self.is_group_admin(username, thread_id):
            try:
                target = text.split("@")[1].strip()
                self.demote_user(target, thread_id, username)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: تنزيل @username")

        elif text.startswith("رفع رتبة") and username == self.admin_username:
            try:
                rank = text.split()[2]
                target = text.split("@")[1].strip()
                self.set_rank(target, rank, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: رفع رتبة [الرتبة] @username")

        elif text.startswith("رتبته @") and username == self.admin_username:
            try:
                target = text.split("@")[1].strip()
                self.show_rank(target, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: رتبته @username")

        elif text.startswith("حذف رتبة @") and username == self.admin_username:
            try:
                target = text.split("@")[1].strip()
                self.remove_rank(target, thread_id)
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: حذف رتبة @username")

        # Game commands
        elif text == "المختلف":
            msg = games_manager.start_different_game(thread_id)
            self.send_message(thread_id, msg)
            
        elif text == "العكس":
            msg = games_manager.start_opposite_game(thread_id)
            self.send_message(thread_id, msg)
            
        elif text == "اعلام":
            msg = games_manager.start_flag_game(thread_id)
            self.send_message(thread_id, msg)
            
        elif text == "الكلمات":
            msg = games_manager.start_word_game(thread_id)
            self.send_message(thread_id, msg)
            
        elif text == "جمع":
            msg = games_manager.start_singular_plural_game(thread_id)
            self.send_message(thread_id, msg)
            
        elif text == "دين":
            msg = games_manager.start_religious_game(thread_id)
            self.send_message(thread_id, msg)
            
        elif text == "نقاطي":
            points = games_manager.get_points(username)
            self.send_message(thread_id, f"@{username} نقاطك: {points} 🎮")
            
        elif text.startswith("بيع نقاطي"):
            try:
                points = int(text.split()[2])
                if games_manager.deduct_points(username, points):
                    money = points * POINTS_TO_MONEY_RATE
                    self.users[username]["balance"] += money
                    self.send_message(thread_id, f"@{username} تم بيع {points} نقطة مقابل {money} 💵")
                else:
                    self.send_message(thread_id, f"@{username} لا تملك نقاط كافية!")
            except (ValueError, IndexError):
                self.send_message(thread_id, "استخدم: بيع نقاطي [العدد]")
                
        # Check game answers
        elif thread_id in games_manager.active_games:
            if games_manager.check_answer(thread_id, text):
                games_manager.add_points(username)
                self.send_message(thread_id, f"@{username} إجابة صحيحة! +1 نقطة 🎯")
            
        elif text in ["اوامر", "الأوامر", "الاوامر"]:
            self.show_commands(thread_id)

        # Save data after processing
        self.save_data()

    def send_message(self, thread_id, text):
        """Send a message to a group thread"""
        try:
            self.cl.direct_send(text, thread_ids=[thread_id])
            time.sleep(1)  # Rate limiting
        except Exception as e:
            print(f"Failed to send message: {e}")

    def check_cooldown(self, username, command):
        """Check if a command is on cooldown for a user"""
        user = self.users[username]
        last_used = user["last_used"].get(command)

        if not last_used:
            return False

        cooldown = COOLDOWNS[command]
        elapsed = (datetime.now() - datetime.fromisoformat(last_used)).total_seconds()

        if elapsed < cooldown:
            remaining = int(cooldown - elapsed)
            return remaining

        return False

    def update_last_used(self, username, command):
        """Update the last used time for a command"""
        self.users[username]["last_used"][command] = datetime.now().isoformat()

    # Banking commands implementation
    def give_salary(self, username, thread_id):
        cooldown = self.check_cooldown(username, "salary")
        if cooldown:
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل الحصول على الراتب التالي!")
            return

        amount = BASE_SALARY
        if self.users[username]["x2"] and datetime.now() < datetime.fromisoformat(self.users[username]["x2_expiry"]):
            amount *= 2

        self.users[username]["balance"] += amount
        self.update_last_used(username, "salary")
        self.send_message(thread_id, f"@{username} لقد حصلت على راتبك {amount} 💵! الرصيد الحالي: {self.users[username]['balance']}")

    def give_tip(self, username, thread_id):
        cooldown = self.check_cooldown(username, "tip")
        if cooldown:
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل الحصول على بخشيش جديد!")
            return

        amount = random.randint(MIN_TIP, MAX_TIP)
        if self.users[username]["x2"] and datetime.now() < datetime.fromisoformat(self.users[username]["x2_expiry"]):
            amount *= 2

        self.users[username]["balance"] += amount
        self.update_last_used(username, "tip")
        self.send_message(thread_id, f"@{username} لقد حصلت على بخشيش {amount} 💵! الرصيد الحالي: {self.users[username]['balance']}")

    def steal_money(self, username, target, thread_id):
        cooldown = self.check_cooldown(username, "steal")
        if cooldown:
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل محاولة السرقة التالية!")
            return

        if target not in self.users:
            self.send_message(thread_id, f"@{username} المستخدم @{target} ليس لديه حساب بنكي!")
            return

        if self.users[target]["balance"] < 1000:
            self.send_message(thread_id, f"@{username} @{target} لا يملك ما يكفي من المال لسرقته!")
            return

        success_chance = random.random()
        if success_chance > 0.7:  # 30% chance of success
            stolen_amount = min(random.randint(1000, 50000), self.users[target]["balance"])
            self.users[target]["balance"] -= stolen_amount
            self.users[username]["balance"] += stolen_amount
            self.update_last_used(username, "steal")
            self.send_message(thread_id, f"@{username} لقد سرقت بنجاح {stolen_amount} 💵 من @{target}! الرصيد الحالي: {self.users[username]['balance']}")
        else:
            self.update_last_used(username, "steal")
            self.send_message(thread_id, f"@{username} لقد فشلت في سرقة @{target}! الشرطة تلاحقك!")

    def invest(self, username, amount, thread_id):
        cooldown = self.check_cooldown(username, "invest")
        if cooldown and text.startswith("استثمار"):
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل الاستثمار التالي!")
            return

        if amount < 1000:
            self.send_message(thread_id, f"@{username} الحد الأدنى للاستثمار هو 1000 💵!")
            return

        if self.users[username]["balance"] < amount:
            self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال!")
            return

        profit_percent = random.randint(10, 100)
        profit = int(amount * (profit_percent / 100))

        self.users[username]["balance"] += profit
        self.update_last_used(username, "invest")
        self.send_message(thread_id, f"@{username} استثمارك {amount} 💵 ربح {profit_percent}% ({profit} 💵)! الرصيد الحالي: {self.users[username]['balance']}")

    def luck(self, username, amount, thread_id):
        cooldown = self.check_cooldown(username, "luck")
        if cooldown and text.startswith("حظ"):
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل محاولة الحظ التالية!")
            return

        if amount < 1000:
            self.send_message(thread_id, f"@{username} الحد الأدنى للحظ هو 1000 💵!")
            return

        if self.users[username]["balance"] < amount:
            self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال!")
            return

        if random.random() > 0.5:  # 50% chance to win
            win_amount = amount * 2
            self.users[username]["balance"] += win_amount
            self.update_last_used(username, "luck")
            self.send_message(thread_id, f"@{username} لقد ربحت {win_amount} 💵! الرصيد الحالي: {self.users[username]['balance']}")
        else:
            self.users[username]["balance"] -= amount
            self.update_last_used(username, "luck")
            self.send_message(thread_id, f"@{username} لقد خسرت {amount} 💵! الرصيد الحالي: {self.users[username]['balance']}")

    def give_treasure(self, username, thread_id):
        cooldown = self.check_cooldown(username, "treasure")
        if cooldown and text == "كنز":
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل الحصول على كنز جديد!")
            return

        amount = random.randint(MIN_TREASURE, MAX_TREASURE)
        if self.users[username]["x2"] and datetime.now() < datetime.fromisoformat(self.users[username]["x2_expiry"]):
            amount *= 2

        self.users[username]["balance"] += amount
        self.update_last_used(username, "treasure")
        self.send_message(thread_id, f"@{username} لقد وجدت كنزًا {amount} 💵! الرصيد الحالي: {self.users[username]['balance']}")

    def spin_wheel(self, username, thread_id):
        cooldown = self.check_cooldown(username, "wheel")
        if cooldown and text == "العجله":
            self.send_message(thread_id, f"@{username} يجب الانتظار {cooldown} ثانية قبل تدوير العجلة التالية!")
            return

        if self.users[username]["balance"] < WHEEL_COST:
            self.send_message(thread_id, f"@{username} تحتاج إلى {WHEEL_COST} 💵 لتدوير العجلة!")
            return

        self.users[username]["balance"] -= WHEEL_COST
        prizes = [
            ("سيارة", "لقد ربحت سيارة جديدة!"),
            ("ماسة", "لقد ربحت ماسة!"),
            ("x2", "لقد ربحت مضاعفة الأرباح لمدة 3 دقائق!"),
            ("500,000", "لقد ربحت 500,000 💵!"),
            ("لاشيء", "لم تربح شيئًا هذه المرة!")
        ]
        prize = random.choice(prizes)

        if prize[0] == "سيارة":
            self.users[username]["properties"]["cars"] += 1
        elif prize[0] == "ماسة":
            self.users[username]["properties"]["diamonds"] += 1
        elif prize[0] == "x2":
            self.users[username]["x2"] = True
            self.users[username]["x2_expiry"] = (datetime.now() + timedelta(minutes=3)).isoformat()
        elif prize[0] == "500,000":
            self.users[username]["balance"] += 500000

        self.update_last_used(username, "wheel")
        self.send_message(thread_id, f"@{username} {prize[1]} الرصيد الحالي: {self.users[username]['balance']}")

    def show_balance(self, username, thread_id):
        """Show user's balance"""
        self.send_message(thread_id, f"@{username} رصيدك هو: {self.users[username]['balance']} 💵")

    def show_properties(self, username, thread_id):
        props = self.users[username]["properties"]
        msg = f"@{username} ممتلكاتك:\n"
        msg += f"💰 الرصيد: {self.users[username]['balance']}\n"
        msg += f"🚗 السيارات: {props['cars']}\n"
        msg += f"💎 الماسات: {props['diamonds']}\n"
        msg += f"📈 الأسهم: {self.users[username]['stocks']}\n"
        self.send_message(thread_id, msg)

    def buy_property(self, username, prop_type, amount, thread_id):
        if prop_type == "cars":
            total_cost = amount * CAR_PRICE
            if self.users[username]["balance"] < total_cost:
                self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال! تحتاج {total_cost} 💵")
                return

            self.users[username]["balance"] -= total_cost
            self.users[username]["properties"]["cars"] += amount
            self.send_message(thread_id, f"@{username} لقد اشتريت {amount} سيارة! الرصيد الحالي: {self.users[username]['balance']}")

    def sell_property(self, username, prop_type, amount, thread_id):
        if prop_type == "cars":
            if self.users[username]["properties"]["cars"] < amount:
                self.send_message(thread_id, f"@{username} لا تملك هذا العدد من السيارات!")
                return

            total_earned = amount * (CAR_PRICE // 2)
            self.users[username]["balance"] += total_earned
            self.users[username]["properties"]["cars"] -= amount
            self.send_message(thread_id, f"@{username} لقد بعت {amount} سيارة وحصلت على {total_earned} 💵! الرصيد الحالي: {self.users[username]['balance']}")

    def gift_property(self, username, target, prop_type, amount, thread_id):
        if target not in self.users:
            self.send_message(thread_id, f"@{username} المستخدم @{target} ليس لديه حساب بنكي!")
            return

        if prop_type == "cars":
            if self.users[username]["properties"]["cars"] < amount:
                self.send_message(thread_id, f"@{username} لا تملك هذا العدد من السيارات!")
                return

            self.users[username]["properties"]["cars"] -= amount
            self.users[target]["properties"]["cars"] += amount
            self.send_message(thread_id, f"@{username} لقد أهديت {amount} سيارة إلى @{target}!")

    def buy_stocks(self, username, amount, thread_id):
        total_cost = amount * STOCK_PRICE
        if self.users[username]["balance"] < total_cost:
            self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال! تحتاج {total_cost} 💵")
            return

        self.users[username]["balance"] -= total_cost
        self.users[username]["stocks"] += amount
        self.send_message(thread_id, f"@{username} لقد اشتريت {amount} سهم! الرصيد الحالي: {self.users[username]['balance']}")

    def sell_stocks(self, username, amount, thread_id):
        if self.users[username]["stocks"] < amount:
            self.send_message(thread_id, f"@{username} لا تملك هذا العدد من الأسهم!")
            return

        # Calculate profit/loss based on random stock price change
        price_change = random.uniform(0.8, 1.5)  # 80% to 150% of original price
        total_earned = int(amount * STOCK_PRICE * price_change)

        self.users[username]["balance"] += total_earned
        self.users[username]["stocks"] -= amount
        self.send_message(thread_id, f"@{username} لقد بعت {amount} سهم وحصلت على {total_earned} 💵! الرصيد الحالي: {self.users[username]['balance']}")

    def show_stock_price(self, username, thread_id):
        price_change = random.uniform(0.8, 1.5)
        current_price = int(STOCK_PRICE * price_change)
        change_percent = int((price_change - 1) * 100)

        if change_percent >= 0:
            msg = f"📈 سعر السهم الحالي: {current_price} 💵 (+{change_percent}%)"
        else:
            msg = f"📉 سعر السهم الحالي: {current_price} 💵 ({change_percent}%)"

        self.send_message(thread_id, msg)

    def take_loan(self, username, thread_id):
        if self.users[username]["debt"] > 0:
            self.send_message(thread_id, f"@{username} لديك دين بالفعل! يجب سداده أولاً.")
            return

        loan_amount = random.randint(MIN_LOAN, MAX_LOAN)
        self.users[username]["balance"] += loan_amount
        self.users[username]["debt"] = loan_amount
        self.users[username]["in_jail"] = True
        self.send_message(thread_id, f"@{username} لقد أخذت قرضًا بقيمة {loan_amount} 💵! أنت الآن في السجن حتى تسدد دينك.")

    def check_jail(self, username, thread_id):
        if self.users[username]["in_jail"]:
            self.send_message(thread_id, f"@{username} أنت في السجن! دينك الحالي: {self.users[username]['debt']} 💵")
        else:
            self.send_message(thread_id, f"@{username} أنت حر ولست في السجن!")

    def check_debt(self, username, thread_id):
        self.send_message(thread_id, f"@{username} دينك الحالي: {self.users[username]['debt']} 💵")

    def pay_debt(self, username, thread_id):
        if not self.users[username]["in_jail"]:
            self.send_message(thread_id, f"@{username} ليس لديك ديون أو لست في السجن!")
            return

        debt = self.users[username]["debt"]
        if self.users[username]["balance"] < debt:
            self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال لسداد دينك! تحتاج {debt} 💵")
            return

        self.users[username]["balance"] -= debt
        self.users[username]["debt"] = 0
        self.users[username]["in_jail"] = False
        self.send_message(thread_id, f"@{username} لقد سددت دينك! أنت الآن حر. الرصيد الحالي: {self.users[username]['balance']}")

    def pay_debt_for(self, username, target, thread_id):
        if target not in self.users:
            self.send_message(thread_id, f"@{username} المستخدم @{target} ليس لديه حساب بنكي!")
            return

        if not self.users[target]["in_jail"]:
            self.send_message(thread_id, f"@{username} @{target} ليس في السجن!")
            return

        debt = self.users[target]["debt"]
        if self.users[username]["balance"] < debt:
            self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال لسداد دين @{target}! تحتاج {debt} 💵")
            return

        self.users[username]["balance"] -= debt
        self.users[target]["debt"] = 0
        self.users[target]["in_jail"] = False
        self.send_message(thread_id, f"@{username} لقد سددت دين @{target}! @{target} أنت الآن حر.")

    def show_group_top(self, thread_id):
        sorted_groups = sorted(self.groups.items(), key=lambda x: x[1]["command_count"], reverse=True)[:20]
        msg = "🏆 توب 20 قروب:\n"
        for i, (group_id, data) in enumerate(sorted_groups, 1):
            msg += f"{i}. {data.get('name', group_id)}: {data['command_count']} أمر\n"
        self.send_message(thread_id, msg)

    def show_user_top(self, thread_id):
        if thread_id not in self.groups:
            self.send_message(thread_id, "لا يوجد بيانات تفاعل لهذه المجموعة بعد!")
            return

        sorted_users = sorted(self.groups[thread_id]["users"].items(), key=lambda x: x[1], reverse=True)[:10]
        msg = "🏆 توب 10 أعضاء متفاعلين:\n"
        for i, (user, count) in enumerate(sorted_users, 1):
            msg += f"{i}. @{user}: {count} أمر\n"
        self.send_message(thread_id, msg)

    def __init__(self):
        self.cl = Client()
        self.load_data()
        self.admin_username = "fpg.x"
        self.processed_messages = set()
        self.adding_response_state = {}  # Track state for each user
        self.darin_count = {}  # Track دارين mentions per user
        self.device_settings = {
            "app_version": "219.0.0.12.117",
            "android_version": 29,
            "android_release": "10",
            "dpi": "480dpi",
            "resolution": "1080x2268",
            "manufacturer": "Samsung",
            "device": "SM-G988U",
            "model": "Galaxy S20 Ultra",
            "cpu": "qcom",
            "version_code": "314665256"
        }

    def start_adding_response(self, username, thread_id):
        self.adding_response_state[username] = {"state": "waiting_keyword", "thread_id": thread_id}
        self.send_message(thread_id, "ارسل الكلمة التي تريد أن يتم الرد عليها")

    def process_adding_response(self, username, text, thread_id):
        if username not in self.adding_response_state:
            return False

        state = self.adding_response_state[username]
        if state["state"] == "waiting_keyword":
            state["keyword"] = text
            state["state"] = "waiting_response"
            self.send_message(thread_id, f"ارسل الرد الذي تريدني أن أقوله عندما يكتب أحد كلمة {text}")
            return True
        elif state["state"] == "waiting_response":
            keyword = state["keyword"]
            self.custom_responses[keyword] = text
            self.save_data()
            self.send_message(thread_id, f"تم إضافة الرد بنجاح! عندما يكتب أحد '{keyword}'، سأرد: {text}")
            del self.adding_response_state[username]
            return True
        return False

    def set_rank(self, target, rank, thread_id):
        if target not in self.users:
            self.send_message(thread_id, f"المستخدم @{target} ليس لديه حساب بنكي!")
            return

        self.users[target]["rank"] = rank
        self.send_message(thread_id, f"تم تعيين رتبة @{target} إلى {rank}")

    def show_rank(self, target, thread_id):
        if target not in self.users:
            self.send_message(thread_id, f"المستخدم @{target} ليس لديه حساب بنكي!")
            return

        self.send_message(thread_id, f"رتبة @{target} هي: {self.users[target]['rank']}")

    def remove_rank(self, target, thread_id):
        if target not in self.users:
            self.send_message(thread_id, f"المستخدم @{target} ليس لديه حساب بنكي!")
            return

        self.users[target]["rank"] = "عضو"
        self.send_message(thread_id, f"تم حذف رتبة @{target}")

    def show_responses(self, thread_id):
        if not self.custom_responses:
            self.send_message(thread_id, "لا توجد ردود مخصصة حالياً!")
            return
            
        msg = "📝 الردود المخصصة:\n\n"
        for i, (trigger, response) in enumerate(self.custom_responses.items(), 1):
            msg += f"{i}. {trigger} ➜ {response}\n"
        self.send_message(thread_id, msg)

    def delete_response(self, response_num, thread_id):
        if not self.custom_responses:
            self.send_message(thread_id, "لا توجد ردود مخصصة للحذف!")
            return
            
        if response_num < 1 or response_num > len(self.custom_responses):
            self.send_message(thread_id, "رقم الرد غير صحيح!")
            return
            
        trigger = list(self.custom_responses.keys())[response_num - 1]
        response = self.custom_responses.pop(trigger)
        self.save_data()
        self.send_message(thread_id, f"تم حذف الرد: {trigger} ➜ {response}")

    def is_group_admin(self, username, thread_id):
        """Check if user is a group admin"""
        try:
            thread_info = self.cl.direct_thread(thread_id)
            # Check if user is thread creator (always admin)
            if hasattr(thread_info, 'thread_title') and username == self.admin_username:
                return True
                
            # For now, only allow the bot owner to perform admin actions
            return username == self.admin_username
        except Exception as e:
            print(f"Error checking admin status: {e}")
            return False

    def kick_user(self, target, thread_id, admin_username):
        """Kick a user from the group"""
        try:
            thread_info = self.cl.direct_thread(thread_id)
            target_id = None
            for user in thread_info.users:
                if user.username == target:
                    target_id = user.pk
                    break
            
            if target_id:
                self.cl.direct_thread_remove_user(thread_id, [target_id])
                self.send_message(thread_id, f"@{admin_username} قام بطرد @{target} من المجموعة")
            else:
                self.send_message(thread_id, f"لم يتم العثور على المستخدم @{target}")
        except Exception as e:
            print(f"Error kicking user: {e}")
            self.send_message(thread_id, "حدث خطأ أثناء محاولة طرد المستخدم")

    def promote_user(self, target, thread_id, admin_username):
        """Promote a user to admin"""
        try:
            thread_info = self.cl.direct_thread(thread_id)
            target_id = None
            for user in thread_info.users:
                if user.username == target:
                    target_id = user.pk
                    break
            
            if target_id:
                self.cl.direct_thread_add_admin(thread_id, [target_id])
                self.send_message(thread_id, f"@{admin_username} قام بترقية @{target} إلى منصب الإدارة")
            else:
                self.send_message(thread_id, f"لم يتم العثور على المستخدم @{target}")
        except Exception as e:
            print(f"Error promoting user: {e}")
            self.send_message(thread_id, "حدث خطأ أثناء محاولة ترقية المستخدم")

    def demote_user(self, target, thread_id, admin_username):
        """Demote an admin to regular user"""
        try:
            thread_info = self.cl.direct_thread(thread_id)
            target_id = None
            for user in thread_info.users:
                if user.username == target:
                    target_id = user.pk
                    break
            
            if target_id:
                self.cl.direct_thread_remove_admin(thread_id, [target_id])
                self.send_message(thread_id, f"@{admin_username} قام بتنزيل @{target} من منصب الإدارة")
            else:
                self.send_message(thread_id, f"لم يتم العثور على المستخدم @{target}")
        except Exception as e:
            print(f"Error demoting user: {e}")
            self.send_message(thread_id, "حدث خطأ أثناء محاولة تنزيل المستخدم")

    def transfer_money(self, username, target, amount, thread_id):
        """Transfer money from one user to another"""
        if amount < 1:
            self.send_message(thread_id, f"@{username} المبلغ يجب أن يكون أكبر من 0!")
            return

        if target not in self.users:
            self.send_message(thread_id, f"@{username} المستخدم @{target} ليس لديه حساب بنكي!")
            return

        if self.users[username]["balance"] < amount:
            self.send_message(thread_id, f"@{username} لا تملك ما يكفي من المال!")
            return

        if self.users[username]["in_jail"]:
            self.send_message(thread_id, f"@{username} لا يمكنك التحويل وأنت في السجن!")
            return

        self.users[username]["balance"] -= amount
        self.users[target]["balance"] += amount
        self.send_message(thread_id, f"@{username} تم تحويل {amount} 💵 إلى @{target}! رصيدك الحالي: {self.users[username]['balance']}")

    def show_commands(self, thread_id):
        commands = [
            "🏦 الأوامر المالية:",
            "- تحويل @username المبلغ: تحويل مبلغ لمستخدم آخر",
            "- راتب: احصل على راتب كل 10 دقائق",
            "- بخشيش: احصل على بخشيش عشوائي كل 10 دقائق",
            "- زرف @username: حاول سرقة مستخدم آخر",
            "- استثمار [المبلغ]: استثمر المال واحصل على ربح",
            "- حظ [المبلغ]: اختبر حظك مع فرصة ربح أو خسارة المبلغ",
            "- كنز: احصل على كنز عشوائي كل 30 دقيقة",
            "",
            "🎡 العجلة:",
            "- العجله: تدوير العجلة (تكلفة 5 مليون)",
            "",
            "🚗 الممتلكات:",
            "- ممتلكاتي: عرض ممتلكاتك",
            "- شراء [عدد] سيارة: شراء سيارات (1 مليون لكل سيارة)",
            "- بيع [عدد] سيارة: بيع سيارات (500 ألف لكل سيارة)",
            "- اهداء [عدد] سيارة بالرد: إهداء سيارات لمستخدم آخر",
            "",
            "📈 الأسهم:",
            "- شراء اسهم [عدد]: شراء أسهم (1 مليون لكل سهم)",
            "- بيع اسهم [عدد]: بيع الأسهم",
            "- سعر الاسهم: عرض سعر الأسهم الحالي",
            "",
            "💰 القروض والسجن:",
            "- قرض: أخذ قرض (يضعك في السجن)",
            "- سجني: التحقق من حالة السجن",
            "- ديوني: عرض ديونك",
            "- سداد ديوني: سداد ديونك والخروج من السجن",
            "- سداد ديونه @username: سداد ديون مستخدم آخر",
            "",
            "🏆 التوب:",
            "- توب القروبات: أعلى 20 مجموعة تفاعلاً",
            "- توب المتفاعلين: أعلى 10 أعضاء تفاعلاً في المجموعة",
            "",
            "⚙️ الأوامر الإدارية (للمالك فقط):",
            "- رفع رتبة [الرتبة] @username: تعيين رتبة",
            "- رتبته @username: عرض رتبة مستخدم",
            "- حذف رتبة @username: حذف رتبة مستخدم",
            "",
            "💬 الردود المخصصة:",
            "- اضف رد: إضافة رد مخصص",
            "",
            "👮‍♂️ أوامر الإدارة (للمشرفين فقط):",
            "- طرد @username: طرد عضو من المجموعة",
            "- ترقية @username: ترقية عضو إلى مشرف",
            "- تنزيل @username: تنزيل مشرف إلى عضو",
            "",
            "🎮 الألعاب:",
            "- المختلف: ابحث عن الإيموجي المختلف",
            "- العكس: اكتب عكس الكلمة",
            "- اعلام: تخمين اسم العلم",
            "- الكلمات: اكتب كلمة تبدأ بالحرف",
            "- جمع: حول بين المفرد والجمع",
            "- دين: أسئلة دينية",
            "- نقاطي: عرض نقاطك",
            "- بيع نقاطي [العدد]: بيع نقاط مقابل رصيد"
        ]

        self.send_message(thread_id, "\n".join(commands))

# Main execution
if __name__ == "__main__":
    bot = BankBot()

    # Login (replace with your credentials)
    username = "li_vihann"
    password = "Qazwsx@12"

    if bot.login(username, password):
        print("Logged in successfully!")
        bot.listen_to_messages()
    else:
        print("Failed to login!")
