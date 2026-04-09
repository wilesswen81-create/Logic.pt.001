##### STRUCTURE OVERVIEW #####
core/
 ├── config
 ├── player
 ├── stat_system
 ├── action_system
 ├── map_system
 ├── event_system
 ├── job_system.py
 ├── university_system
 ├── turn_system
 └── data/
      ├── actions
      ├── jobs
      ├── programs
      ├── events
      └── map
##### จำลองloop #####
# 1. สร้างตัวละคร
p = Player()

# 2. จำลองการเดินทางไปห้างเพื่อซื้อเสื้อผ้า
if MapSystem.move(p, "mall"):
    action_data = {"id": "buy_cl", "name": "ซื้อชุด", "time": 10, "cost": 800, "is_clothes": True}
    success, msg = ActionSystem.execute(p, action_data)
    print(f"Action: {msg} | Time left: {p.time}")

# 3. จบเทิร์น
logs = TurnSystem.end_turn(p)
for l in logs: print(f"Event: {l}")

#### CONFIG ####
class Config:
    MAX_STAT = 100
    MIN_STAT = 0
    MAX_MONEY = 1_000_000

    TURN_TIME = 90
    TOTAL_TURNS = 30

    ABNORMAL = {
        "warning": 25,
        "critical": 20
    }

    DEGREE_MULTIPLIER = {
        "none": 1.0,
        "bachelor": 1.2,
        "master": 1.6,
        "phd": 2.0
    }

#### PLAYER (state กลาง) ####
class Player:
    def __init__(self):

        self.time = Config.TURN_TIME
        self.turn = 1
        self.location = "home"

        self.stats = {
            "health": 0,
            "stress": 0,
            "knowledge": 0,
            "money": 0
        }
# วิชาเรียน (สะสมจำนวนครั้งที่เรียน)
        self.subjects = {
            s: 0 for s in [
                "math 1","physics","chemistry","biology",
                "thai","english","social","math 2","general science"
            ]
        }

        self.education = {
            "degree": "none",
            "major": None
        }
        self.active_subjects = [] # วิชาทีโชว์บนจอตามคณะที่เลือก
      
# Timers & States
        self.rent_timer = 0
        self.clothes_timer = 0    
        self.visited_over_25 = {k: False for k in ["health", "stress", "knowledge"]}
        self.abnormal = {}
        
#### STAT SYSTEM ####
  class StatSystem:

    @staticmethod
    def clamp(v):
        return max(Config.MIN_STAT, min(v, Config.MAX_STAT))

    @staticmethod
    def update(player, stat, value):
        if stat == "money":
            player.stats[stat] = max(-10000, min(player.stats[stat] + value, Config.MAX_MONEY))
            return

        player.stats[stat] = StatSystem.clamp(player.stats[stat] + value)
        
   # เช็คขีดจำกัด 25 เพื่อเริ่มระบบ Abnormal
        if player.stats[stat] > 25:
            player.visited_over_25[stat] = True
            
        player.stats[stat] = StatSystem.clamp(player.stats[stat] + value)

        if player.stats[stat] > 25:
            player.visited_over_25[stat] = True

        StatSystem.refresh_abnormal(player, stat)
        
    @staticmethod
    def update_abnormal(player, stat):
        v = player.stats[stat]

        if not player.visited_over_25[stat]:
            return

        if v <= Config.ABNORMAL["critical"]:
            player.abnormal[stat] = 0.5
        elif v <= Config.ABNORMAL["warning"]:
            player.abnormal[stat] = 0.75
        else:
            player.abnormal.pop(stat, None)

    @staticmethod
    def multiplier(player, stat):
        return player.abnormal.get(stat, 1.0)

 #### ACTION SYSTEM (กรุณารอการปรับแต่งค่าสักครู่ ใส่เท่าที่มีไปก่อน) ####
 class ActionSystem:

    @staticmethod
    def efficiency(player):
        h = player.stats["health"]
        s = player.stats["stress"]

        e = 1 + (h - 50)/100 - (s / 150)
        return max(0.3, e)

    @staticmethod
    def execute(player, action):

        if player.time < action["time_cost"]:
            return False

        player.time -= action["time_cost"]

        eff = ActionSystem.efficiency(player)

        for stat, base in action.get("effects", {}).items():

            val = base * eff if stat == "knowledge" else base
            val *= StatSystem.multiplier(player, stat)

            StatSystem.update(player, stat, val)

   # money
        if "money" in action:
            player.stats["money"] += action["money"]

   # subject
        if "subject" in action:
            player.subjects[action["subject"]] += 1

        return True

#### MAP SYSTEM ####
class MapSystem:

    DISTANCE = {
        "home": {"school": 6, "fastfood": 5, "park": 4},
        "school": {"home": 6, "fastfood": 4},
        "fastfood": {"home": 5},
        "park": {"home": 4}
    }

    @staticmethod
    def move(player, dest):
        current = player.location
        if dest == current: return True
        
   # ถ้าไม่มีในแมพ ให้คิดค่าเฉลี่ย 10
        cost = MapSystem.DISTANCE.get(current, {}).get(dest, 10)
        
        if player.time < cost: return False
        
        player.time -= cost
        player.location = dest
        return True

### JOB SYSTEM ###
class JobSystem:

    @staticmethod
    def salary(player, job):
        base = job.get["base_salary",6565]
        mult = Config.DEGREE_MULTIPLIER[player.education["degree"]]

        salary = base * mult

   # โบนัสใบประกอบวิชาชีพ x2
        license_majors = ["medicine", "engineering", "accounting"]
        if player.education["major"] in license_majors:
            salary *= 2
        return int(salary)

    @staticmethod
    def do_job(player, job):
        pay = JobSystem.salary(player, job)

        player.stats["money"] += pay

        StatSystem.update(player, "stress", job["stress"])
        StatSystem.update(player, "health", job["health"])

#### UNIVERSITY SYSTEM ####
class UniversitySystem:

    @staticmethod
    def can_apply(player, program):

        if player.stats["knowledge"] < program["knowledge"]:
            return False

        for s, req in program["subjects"].items():
            if player.subjects[s] < req:
                return False

        return True

    @staticmethod
    def exam(player):
        import random

        score = 0
        eff = ActionSystem.efficiency(player)

        for _ in range(3):
            chance = min(0.9, (player.stats["knowledge"]/100) * eff)
            if random.random() < chance:
                score += 1

        return score

    @staticmethod
    def admission(player, choices):

        valid = [c if UniversitySystem.can_apply(player, c) else None for c in choices]

        score = UniversitySystem.exam(player)

        if score == 0:
            return None

        selected = valid[score - 1]

        if not selected:
            return None

        player.education["degree"] = "bachelor"
        player.education["major"] = selected["name"]

        return selected["name"]

#### EVENT SYSTEM ####
class EventSystem:

    @staticmethod
    def get_events(player, events):

        triggered = []

        for e in events:
            if e["type"] != "random" and e["condition"](player):
                triggered.append(e)

        if player.turn % 4 == 0:
            triggered.append(EventSystem.random_event(events))

        triggered.sort(key=lambda e: EventSystem.priority(player, e))
        return triggered

    @staticmethod
    def priority(player, e):
        if e["type"] == "random":
            return 999
        return min(player.stats.values())

    @staticmethod
    def random_event(events):
        import random
        pool = [e for e in events if e["type"] == "random"][0]["pool"]
        return {"type": "random", "effect": random.choice(pool)}

  #### TURN SYSTEM ####
  @staticmethod
    def end_turn(player):
        messages = []
        
   # จ่ายค่าเช่า (ทุก 2 เทิร์น)
        player.rent_timer += 1
        if player.rent_timer >= 2:
            player.stats["money"] -= 500
            player.rent_timer = 0
            messages.append("จ่ายค่าเช่า 500 บาท")

   # เสื้อผ้า (ทุก 4 เทิร์น)
        player.clothes_timer += 1
        if player.clothes_timer >= 4:
            StatSystem.update(player, "health", -10)
            StatSystem.update(player, "stress", 5)
            messages.append("เสื้อผ้าขาด! Health -10")

   # หนี้สิน
        if player.stats["money"] < 0:
            StatSystem.update(player, "stress", 20)
            messages.append("เป็นหนี้! Stress +20")

   # Reset Turn
        player.turn += 1
        player.time = Config.TURN_TIME
        return messages

#### Data #####
### Actoin ###
ALL_ACTIONS = {
    "home": [
        {"id": "h_sleep", "name": "นอนหลับพักผ่อน", "time": 10, "effects": {"health": 15, "stress": -10}},
        {"id": "h_media", "name": "ดูหนัง/เล่นเกม", "time": 8, "effects": {"stress": -15, "health": -2}},
        {"id": "h_pay_rent", "name": "จ่ายค่าเช่าบ้าน", "time": 1, "cost": 500, "is_rent": True}
    ],
    "school": [
   # เรียน 9 วิชาหลัก (ปรับค่าตามความเหมาะสม)
        {"id": "sch_study", "name": "เข้าเรียนวิชาพื้นฐาน", "time": 15, "cost": 100, "effects": {"knowledge": 8, "stress": 5}},
        {"id": "sch_job_teacher", "name": "สอนหนังสือ (ครู)", "time": 20, "is_job": True, "req_degree": "bachelor"}
    ],
    "fastfood": [
   # ร้านตามสั่ง ซื้อข้าว 6 แบบ (สุ่มค่าสารอาหาร/ความแซบ)
        {"id": "ff_menu_1", "name": "เบอร์เกอร์ประหยัด", "time": 5, "cost": 120, "effects": {"health": 5, "stress": -2}},
        {"id": "ff_menu_2", "name": "ไก่ทอดชุดใหญ่", "time": 7, "cost": 240, "effects": {"health": 8, "stress": -10}},
        {"id": "ff_menu_3", "name": "สลัดสุขภาพ", "time": 6, "cost": 355, "effects": {"health": 12, "stress": 2}},
        {"id": "ff_menu_4", "name": "ชุดข้าวแกงกะหรี่", "time": 8, "cost": 310, "effects": {"health": 10, "stress": -5}},
        {"id": "ff_menu_5", "name": "ไอศกรีมซันเดย์", "time": 4, "cost": 67, "effects": {"health": -2, "stress": -15}},
        {"id": "ff_menu_6", "name": "น้ำอัดลมรีฟิล", "time": 3, "cost": 30, "effects": {"health": -5, "stress": -5}},
        {"id": "ff_job_parttime", "name": "พนักงานพาร์ทไทม์", "time": 15, "is_job": True, "base_pay": 6565},
        {"id": "ff_owner", "name": "บริหารร้าน (เจ้าของ)", "time": 20, "is_job": True, "req_degree": "bachelor"}
    ],
    "university": [
        {"id": "uni_major_select", "name": "เลือกคณะ/ลงทะเบียน", "time": 10, "is_event": True},
        {"id": "uni_study_major", "name": "เข้าเรียนวิชาภาค", "time": 18, "effects": {"knowledge": 12, "stress": 8}},
        {"id": "uni_job_prof", "name": "อาจารย์มหาวิทยาลัย", "time": 20, "is_job": True, "req_degree": "master"}
    ],
    "gym": [
        {"id": "gym_cardio", "name": "วิ่งลู่วิ่ง", "time": 10, "cost": 500, "effects": {"health": 10, "stress": -2}},
        {"id": "gym_weight", "name": "ยกเวท", "time": 12, "cost": 500, "effects": {"health": 12, "stress": -2}},
        {"id": "gym_yoga", "name": "โยคะ", "time": 10, "cost": 800, "effects": {"health": 5, "stress": -12}},
        {"id": "gym_boxing", "name": "ต่อยมวย", "time": 15, "cost": 100, "effects": {"health": 15, "stress": -5}},
        {"id": "gym_job_trainer", "name": "เทรนเนอร์", "time": 20, "is_job": True, "req_major": "sports_science"}
    ],
    "mall": [
        {"id": "mall_snack", "name": "ซื้อขนม/ของกินเล่น", "time": 5, "cost": 100, "effects": {"stress": -5}},
        {"id": "mall_electronics", "name": "ซื้อเครื่องใช้ไฟฟ้า", "time": 20, "cost": 1500, "effects": {"stress": -10}},
        {"id": "mall_books", "name": "ซื้อหนังสือ", "time": 15, "cost": 300, "effects": {"knowledge": 5, "stress": -5}},
        {"id": "mall_cosmetics", "name": "ซื้อเครื่องสำอาง", "time": 12, "cost": 600, "effects": {"stress": -8}},
        {"id": "mall_gacha", "name": "สุ่มกาชา/ของสะสม", "time": 10, "cost": 200, "effects": {"stress": -12, "health": -2}},
        {"id": "mall_clothes", "name": "ซื้อชุดใหม่คนรวย", "time": 10, "cost": 28000, "is_clothes": True}
    ],
    "hospital": [
        {"id": "hos_checkup", "name": "รักษาน้อย", "time": 10, "cost": 500, "effects": {"health": 20}},
        {"id": "hos_vaccine", "name": "รักษากลาง", "time": 8, "cost": 1200, "effects": {"health": 40}},
        {"id": "hos_surgery", "name": "รักษามาก", "time": 30, "cost": 5000, "effects": {"health": 80, "stress": 10}},
        {"id": "hos_job_doc", "name": "ปฏิบัติงานแพทย์", "time": 25, "is_job": True, "req_major": "medicine"}
    ],
    "office": [
        {"id": "off_job_staff", "name": "พนักงานออฟฟิศ", "time": 20, "is_job": True, "req_degree": "bachelor"}
    ],
    "bank": [
        {"id": "bank_deposit", "name": "ฝากเงิน", "time": 5},
        {"id": "bank_stock", "name": "เทรดหุ้น (High Risk)", "time": 10, "is_stock": True},
        {"id": "bank_job_acc", "name": "พนักงานบัญชี", "time": 20, "is_job": True, "req_major": "accounting"}
    ],
    "park": [
        {"id": "park_jog", "name": "วิ่งเหยาะๆ (ฟรี)", "time": 12, "effects": {"health": 5, "stress": -5}},
        {"id": "park_sit", "name": "นั่งชมสวน (ฟรี)", "time": 8, "effects": {"stress": -10}},
        {"id": "park_nap", "name": "งีบใต้ต้นไม้ (ฟรี)", "time": 15, "effects": {"health": 2, "stress": -5}}
    ]
}

