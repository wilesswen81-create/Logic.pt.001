##### STRUCTURE OVERVIEW #####
core/
 ├── config
 ├── player
 ├── stat_system
 ├── action_system
 ├── map_system
 ├── event_system
 ├── ExamSystem
 ├── job_system.py
 ├── university_system
 ├── turn_system
 ├── clothing_system
 └── data/
      ├── actions
      ├── jobs
      ├── events
      └── map
      
###### จำลองloop ########

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
        self.stats = {"health": 20, "stress": 10, "knowledge": 0, "money": 5000}
        
        # วิชาเรียน
        self.subjects = {s: 0 for s in [
            "math 1","physics","chemistry","biology","thai",
            "english","social","math 2","general science"
        ]}
        
        self.education = {"degree": "none", "major": None}
        self.active_subjects = []
        
        # Timers, Durability & Career States
        self.rent_timer = 0
        self.clothes_durability = 15  # เริ่มต้นด้วยชุด Basic
        self.work_done_count = 0      # จำนวนงานที่ทำสำเร็จในเทิร์นนี้
        self.part_time_history = 0    # สะสมเทิร์นที่ทำงานพาร์ทไทม์ครบโควตา
        self.is_shop_owner = False
        self.wait_exam_turns = 0      # เทิร์นที่ต้องรอถ้าสอบตก
        
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

    # คำนวณค่าที่จะอัปเดต โดยใช้ Multiplier ใหม่
        final_value = value
        if value > 0: # เฉพาะตอนที่ Stat กำลังจะเพิ่มขึ้น
            final_value *= StatSystem.get_custom_multiplier(player, stat)
        
        player.stats[stat] = StatSystem.clamp(player.stats[stat] + final_value)
        
        if player.stats[stat] > 25:
            player.visited_over_25[stat] = True

        StatSystem.refresh_abnormal(player, stat)

    @staticmethod
    def get_custom_multiplier(player, stat):
        v = player.stats[stat]
        
    #  Logic ใหม่สำหรับ "ความเครียด" เท่านั้น 
        if stat == "stress":
            if v >= 80: return 2.0   # เครียดหนักมาก พุ่งคูณสอง!
            if v >= 75: return 1.5   # เริ่มคุมไม่อยู่ คูณ 1.5
            return 1.0
        
     #  Logic สำหรับค่าอื่นๆ (Health, Knowledge) 
     # ยิ่งต่ำ ยิ่งเพิ่มยาก 
        if not player.visited_over_25.get(stat): return 1.0
        
        if v <= Config.ABNORMAL["critical"]: return 0.5
        if v <= Config.ABNORMAL["warning"]: return 0.75
        return 1.0

    @staticmethod
    def refresh_abnormal(player, stat):
    # ฟังก์ชันนี้เอาไว้คุมพวกระบบสี UI หรือ Alert แจ้งเตือนเฉยๆ
        v = player.stats[stat]
        if stat != "stress":
            if v <= Config.ABNORMAL["critical"]: player.abnormal[stat] = "CRITICAL"
            elif v <= Config.ABNORMAL["warning"]: player.abnormal[stat] = "WARNING"
            else: player.abnormal.pop(stat, None)
        else:
            if v >= 80: player.abnormal[stat] = "PANIC"
            elif v >= 75: player.abnormal[stat] = "HIGH_STRESS"
            else: player.abnormal.pop(stat, None)

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
        cost = action.get("cost", 0)
        time_needed = action.get("time", action.get("time_cost", 0))

        if player.time < time_needed: return False, "เวลาไม่พอ"
        if player.stats["money"] < cost: return False, "เงินไม่พอ"

        player.time -= time_needed
        player.stats["money"] -= cost
        eff = ActionSystem.efficiency(player)

     # ผลลัพธ์จาก Action
        for stat, base in action.get("effects", {}).items():
            val = base * eff if stat in ["knowledge", "social"] else base
            val *= StatSystem.multiplier(player, stat)
            StatSystem.update(player, stat, val)

    # Logic พิเศษ
        if action.get("is_clothes"):
            # เช็คว่าซื้อแบบไหน (ใน data mall มีแค่ตัวเดียว แต่เราสามารถขยายได้)
            player.clothes_durability = 60 if action["cost"] > 10000 else 15
            
        if "subject" in action:
            player.subjects[action["subject"]] += 1
            
        if action.get("is_stock"):
            res = random.choice([0.5, 0.8, 1.5, 2.5])
            player.stats["money"] += int(1000 * res)

        if action.get("is_job"):
            return JobSystem.do_job(player, action)

        return True, f"ทำ {action['name']} สำเร็จ!"

    # money
        if "money" in action:
            player.stats["money"] += action["money"]

    # subject
        if "subject" in action:
            player.subjects[action["subject"]] += 1

        return True

#### MAP SYSTEM ####
class MapSystem:
    TOTAL_DISTANCE = 40  # ระยะทางรวมทั้งหมดของแผนมี่

    @staticmethod
    def get_travel_time(start_node, end_node):
        if start_node == end_node: return 0
        
        pos1 = LOCATIONS_1D.get(start_node)
        pos2 = LOCATIONS_1D.get(end_node)
        
        if pos1 is None or pos2 is None: return 5
            
        # 1. คำนวณระยะทางแบบเดินหน้า-ถอยหลังปกติ
        direct_dist = abs(pos1 - pos2)
        
        # 2. คำนวณระยะทางแบบวนลูปไปอีกด้าน (ข้ามจุดตัด 0/40)
        circular_dist = MapSystem.TOTAL_DISTANCE - direct_dist
        
        # 3. เลือกทางที่สั้นที่สุด (Shortest Path)
        shortest_dist = min(direct_dist, circular_dist)
        
        # 4. แปลงเป็นเวลา (Time Unit) ใช้ตัวคูณเดิมเพื่อให้เลขดูสมจริง
        travel_time = round(shortest_dist * 0.4)
        
        return max(1, travel_time)
        
####  ExamSystem ####
class ExamSystem:

    @staticmethod
    def take_exam(player, choices):
        # คำนวณโอกาสสำเร็จจาก Knowledge และประสิทธิภาพร่างกายตอนนั้น
        eff = ActionSystem.efficiency(player)
        success_chance = (player.stats["knowledge"] / 100) * eff
        
        score = 0
        for _ in range(3): # ทำข้อสอบ 3 ชุด
            if random.random() < success_chance:
                score += 1
        
        # ผลลัพธ์ตามคะแนนที่ได้ (3=อันดับ1, 2=อันดับ2, 1=อันดับ3, 0=นก)
        if score == 0:
            player.wait_exam_turns = 2 # สอบตกต้องรอ 2 เทิร์น (เทิร์น 7, 8) สอบได้อีกทีเทิร์น 9
            player.education["major"] = None
            return None
        
        selected_major = choices[3 - score]
        player.education["major"] = selected_major
        player.education["degree"] = "bachelor"
        return selected_major
  
### JOB SYSTEM ###
class JobSystem:
    # ดึง Data จากหัวข้อถัดไปมาใช้
    @staticmethod
    def do_job(player, job_action):
    # ค้นหา Job Data จากความรู้/คณะ
        major = player.education["major"]
        job_info = None
        
    # หา JobInfo จาก JOB_DATA (ตามฟอร์แมตกลุ่มคณะ)
        for group in JOB_DATA.values():
            if major in group:
                job_info = group[major]
                break
        
        if not job_info: # พาร์ทไทม์
            job_info = {"job": "Part-time", "quota": 4, "time": 15, "pay": 2525}

    # การทำงาน 1 ครั้ง
        player.work_done_count += 1
        ClothingSystem.wear_tear(player)
        StatSystem.update(player, "stress", 5)
        StatSystem.update(player, "health", -2)
        
        return True, f"ทำงาน {job_info['job']} ({player.work_done_count}/{job_info['quota']})"

    @staticmethod
    def calculate_salary_payout(player):
        major = player.education["major"]
        job_info = None
        for group in JOB_DATA.values():
            if major in group:
                job_info = group[major]
                break
        
        if not job_info: job_info = {"pay": 2000, "quota": 4}

    # จ่ายตาม % ผลงาน
        perf = min(1.0, player.work_done_count / job_info["quota"])
        salary = job_info["pay"] * perf
        
        if job_info.get("license"): salary *= 2
        salary *= Config.DEGREE_MULTIPLIER.get(player.education["degree"], 1.0)
        
    # เช็คสะสมพาร์ทไทม์ (ถ้าทำครบโควตา)
        if perf >= 1.0 and not player.education["major"]:
            player.part_time_history += 1
            if player.part_time_history >= 12: player.is_shop_owner = True

        passive = 3000 if player.is_shop_owner else 0
        total = int(salary + passive)
        player.stats["money"] += total
        player.work_done_count = 0 # reset
        return total

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
    def get_random_event(player):
        """สุ่มอีเวนท์ตามช่วงอายุของตัวละคร"""
        import random
        # แยก Pool ตามเทิร์น
        if player.turn <= 6:
            pool = EVENT_DATA["high_school"]
        else:
            pool = EVENT_DATA["adult"]
            
        return random.choice(pool)

    @staticmethod
    def apply_choice(player, event_card, choice_index):
        """คำนวณผลลัพธ์หลังจากผู้เล่นเลือก Choice"""
        selected = event_card["choices"][choice_index]
        msg = selected["result_msg"]
        
        # อัปเดต Stats ตามที่เลือก
        for stat, val in selected.get("effects", {}).items():
            StatSystem.update(player, stat, val)
            
        return msg

  #### TURN SYSTEM ####
  class TurnSystem:
  
    @staticmethod
    def end_turn(player):
        messages = []
        
        # 1. จ่ายเงินเดือน (สรุปผลงานจากเทิร์นที่เพิ่งจบ)
        salary_payout = JobSystem.calculate_salary_payout(player)
        messages.append(f"💰 รายได้เทิร์นนี้: {salary_payout} บาท")

        # 2. ระบบค่าเช่าอัตโนมัติ (เช็คเมื่อจบเทิร์นคู่ เพื่อให้มีผลในเทิร์นคี่ถัดไป)
        player.rent_timer += 1
        if player.rent_timer >= 2:
            rent_amount = 7500
            player.stats["money"] -= rent_amount
            player.rent_timer = 0
            messages.append(f"🏠 [ระบบหักอัตโนมัติ] จ่ายค่าเช่าบ้านแล้ว {rent_amount} บาท")

        # 3. เช็คหนี้และสถานะความเครียด (Debt Logic ที่เราคุยกันไว้)
        if player.stats["money"] < 0:
            if not player.is_in_debt:
                StatSystem.update(player, "stress", 20)
                player.is_in_debt = True
                messages.append("⚠️ เงินติดลบ! คุณเริ่มเป็นหนี้แล้ว (Stress +20)")
            else:
                StatSystem.update(player, "stress", 10)
                messages.append("💸 ยังใช้หนี้ไม่หมด ความเครียดสะสมต่อเนื่อง (Stress +10)")
        else:
            if player.is_in_debt:
                player.is_in_debt = False
                messages.append("✅ ปลดหนี้เรียบร้อย! สุขภาพจิตดีขึ้น")

        # 4. เตรียมตัวสำหรับเทิร์นถัดไป
        player.turn += 1
        player.time = Config.TURN_TIME
        
        # แจ้งเตือนเรื่องสอบ (ถ้ามี)
        if player.wait_exam_turns > 0:
            player.wait_exam_turns -= 1
            if player.wait_exam_turns == 0:
                messages.append("🎓 คุณสามารถกลับไปสอบมหาวิทยาลัยได้แล้ว!")
            else:
                messages.append(f"⏳ อีก {player.wait_exam_turns} เทิร์น ถึงจะสอบใหม่ได้")

        return messages

        @staticmethod
    def is_everyone_ready_for_exam(player_list):
        """เช็คว่าทุกคนเล่นจบเทิร์น 6 พร้อมกันหรือยัง"""
        # ถ้าทุกคนอยู่ที่เทิร์น 7 แปลว่าทุกคนเพิ่งกด 'จบเทิร์น 6' มาพร้อมกัน
        return all(p.turn == 7 for p in player_list)

    @staticmethod
    def trigger_grand_exam(player_list):
        """ฟังก์ชันสำหรับรันอีเวนต์สอบใหญ่คั่นกลาง (เวียนกันสอบทีละคน)"""
        exam_results = []
        
        for p in player_list:
            # เช็คว่าผู้เล่นมีรายชื่อคณะที่เลือกไว้หรือยัง (เลือกไว้อันดับ 1, 2, 3)
            # ถ้าไม่มี ให้ระบบสุ่มคณะพื้นฐานให้ หรือเด้ง Pop-up ให้เลือก
            choices = getattr(p, 'exam_choices', ["Science", "Arts", "Sports"]) 
            
            # เรียกใช้ระบบสอบที่คำนวณจาก Knowledge + Efficiency
            major_result = ExamSystem.take_exam(p, choices)
            
            if major_result:
                exam_results.append(f"🌟 {p.name}: ยินดีด้วยคุณสอบติดคณะ {major_result}!")
            else:
                exam_results.append(f"💀 {p.name}: เสียใจด้วยสอบไม่ติดคณะใดเลย (สามารถรอสอบรอบแก้ตัวปีถัดไป)")
        
        return exam_results
        
#### ClothingSystem ####
class ClothingSystem:
    TYPES = {
        "basic": {"name": "เสื้อผ้าทั่วไป", "cost": 500, "durability": 15}, # ทนทำงานได้ 15 ครั้ง
        "premium": {"name": "ชุดแบรนด์เนม", "cost": 2500, "durability": 45} # ทนทำงานได้ 45 ครั้ง
    }

    @staticmethod
    def wear_tear(player):
        player.clothes_durability -= 1
        if player.clothes_durability <= 0:
            # ผลกระทบเมื่อเสื้อผ้าพัง
            StatSystem.update(player, "health", -10)
            StatSystem.update(player, "stress", 20)
            return "เสื้อผ้าของคุณขาดแล้ว! สุขภาพลดลงและความเครียดพุ่งสูง"
        return None





======================================================================






#### Data #####

### Actoin(for สถานที่) ###
ALL_ACTIONS = {

    "home": [
        {"id": "h_sleep", "name": "นอนหลับพักผ่อน", "time": 10, "effects": {"health": 15, "stress": -10}},
        {"id": "h_media", "name": "ดูหนัง/เล่นเกม", "time": 8, "effects": {"stress": -15, "health": -2}},
        {"id": "h_pay_rent", "name": "จ่ายค่าเช่าบ้าน", "time": 1, "cost": 500, "is_rent": True}
    ],
    "school": [
        {"id": "sch_study", "name": "เข้าเรียนวิชาพื้นฐาน", "time": 15, "cost": 100, "effects": {"knowledge": 8, "stress": 5}},
        # Jobs at School
        {"id": "sch_job_teacher", "name": "สอนหนังสือ (ครู)", "time": 12, "is_job": True, "req_major": "Education"},
        {"id": "sch_job_early", "name": "ดูแลเด็ก (ครูปฐมวัย)", "time": 10, "is_job": True, "req_major": "EarlyChild"},
        {"id": "sch_job_pe", "name": "สอนพละ (ครูพละ)", "time": 12, "is_job": True, "req_major": "Physical_Ed"}
    ],
    "fastfood": [
    #เปลี่นชื่อเมนูเอาเด้อ
        {"id": "ff_menu_1", "name": "เบอร์เกอร์ประหยัด", "time": 5, "cost": 120, "effects": {"health": 5, "stress": -2}},
        {"id": "ff_menu_2", "name": "ไก่ทอดชุดใหญ่", "time": 7, "cost": 240, "effects": {"health": 8, "stress": -10}},
        {"id": "ff_menu_3", "name": "สลัดสุขภาพ", "time": 6, "cost": 355, "effects": {"health": 12, "stress": 2}},
        {"id": "ff_menu_4", "name": "ชุดข้าวแกงกะหรี่", "time": 8, "cost": 310, "effects": {"health": 10, "stress": -5}},
        {"id": "ff_menu_5", "name": "ไอศกรีมซันเดย์", "time": 4, "cost": 67, "effects": {"health": -2, "stress": -15}},
        {"id": "ff_menu_6", "name": "น้ำอัดลมรีฟิล", "time": 3, "cost": 30, "effects": {"health": -5, "stress": -5}},
        # Jobs at Fastfood
        {"id": "ff_job_parttime", "name": "พนักงานพาร์ทไทม์", "time": 15, "is_job": True, "base_pay": 6565},
        {"id": "ff_owner", "name": "บริหารร้าน (เจ้าของ)", "time": 20, "is_job": True, "is_owner": True}
    ],
    "university": [
        {"id": "uni_major_select", "name": "เลือกคณะ/ลงทะเบียน", "time": 10, "is_event": True},
        {"id": "uni_study_major", "name": "เข้าเรียนวิชาภาค", "time": 18, "effects": {"knowledge": 12, "stress": 8}},
        # Jobs at University (Professors & Scientists)
        {"id": "uni_job_prof_ed", "name": "บรรยาย (อาจารย์ครู)", "time": 18, "is_job": True, "req_major": "Master_Ed"},
        {"id": "uni_job_prof_arts", "name": "บรรยาย (อาจารย์ภาษา)", "time": 18, "is_job": True, "req_major": "Master_Arts"},
        {"id": "uni_job_prof_sci", "name": "บรรยาย (อาจารย์วิทย์)", "time": 20, "is_job": True, "req_major": "Master_Sci"},
        {"id": "uni_job_bio", "name": "วิจัยชีววิทยา", "time": 12, "is_job": True, "req_major": "Biology"},
        {"id": "uni_job_phys", "name": "วิจัยฟิสิกส์", "time": 12, "is_job": True, "req_major": "Physics"}
    ],
    "gym": [
        {"id": "gym_cardio", "name": "วิ่งลู่วิ่ง", "time": 10, "cost": 500, "effects": {"health": 10, "stress": -2}},
        {"id": "gym_weight", "name": "ยกเวท", "time": 12, "cost": 600, "effects": {"health": 12, "stress": -2}},
        {"id": "gym_yoga", "name": "โยคะ", "time": 10, "cost": 800, "effects": {"health": 5, "stress": -12}},
        {"id": "gym_boxing", "name": "ต่อยมวย", "time": 15, "cost": 100, "effects": {"health": 15, "stress": -5}},
        # Jobs at Gym
        {"id": "gym_job_trainer", "name": "เทรนเนอร์", "time": 10, "is_job": True, "req_major": "Sports_Sci"},
        {"id": "gym_job_mgmt", "name": "ผู้จัดการทีม", "time": 12, "is_job": True, "req_major": "Sports_Mgmt"}
    ],
    "mall": [
        {"id": "mall_snack", "name": "ซื้อขนม/ของกินเล่น", "time": 5, "cost": 100, "effects": {"stress": -5}},
        {"id": "mall_electronics", "name": "ซื้อเครื่องใช้ไฟฟ้า", "time": 20, "cost": 1500, "effects": {"stress": -10, "health": 5}},
        {"id": "mall_books", "name": "ซื้อหนังสือ", "time": 15, "cost": 300, "effects": {"knowledge": 5, "stress": -5}},
        {"id": "mall_cosmetics", "name": "ซื้อเครื่องสำอาง", "time": 12, "cost": 600, "effects": {"stress": -8}},
        {"id": "mall_gacha", "name": "สุ่มกาชา/ของสะสม", "time": 10, "cost": 200, "effects": {"stress": -12, "health": -2}},
        {"id": "mall_clothes_basic", "name": "ซื้อชุดทั่วไป", "time": 10, "cost": 500, "is_clothes": True},
        {"id": "mall_clothes_rich", "name": "ซื้อชุดใหม่คนรวย", "time": 10, "cost": 28000, "is_clothes": True}
    ],
    "hospital": [
        {"id": "hos_checkup", "name": "รักษาน้อย", "time": 10, "cost": 500, "effects": {"health": 20}},
        {"id": "hos_vaccine", "name": "รักษากลาง", "time": 8, "cost": 1500, "effects": {"health": 40}},
        {"id": "hos_surgery", "name": "รักษามาก", "time": 30, "cost": 5000, "effects": {"health": 80, "stress": 10}},
        # Jobs at Hospital
        {"id": "hos_job_doc", "name": "ตรวจโรค (หมอ)", "time": 8, "is_job": True, "req_major": "Medicine"},
        {"id": "hos_job_dentist", "name": "ถอนฟัน (ทันตะ)", "time": 10, "is_job": True, "req_major": "Dentist"},
        {"id": "hos_job_pharm", "name": "จัดยา (เภสัช)", "time": 10, "is_job": True, "req_major": "Pharmacy"}
    ],
    "office": [
        # Jobs at Office
        {"id": "off_job_civil", "name": "เขียนแบบ (วิศวกรโยธา)", "time": 12, "is_job": True, "req_major": "Civil"},
        {"id": "off_job_com", "name": "เขียนโค้ด (Programmer)", "time": 15, "is_job": True, "req_major": "Computer"},
        {"id": "off_job_elec", "name": "วางระบบ (วิศวกรไฟฟ้า)", "time": 12, "is_job": True, "req_major": "Electrical"},
        {"id": "off_job_trans", "name": "แปลเอกสาร (ล่าม)", "time": 12, "is_job": True, "req_major": "Translation"},
        {"id": "off_job_ling", "name": "วิจัยภาษา (นักภาษาศาสตร์)", "time": 15, "is_job": True, "req_major": "Linguistics"}
    ],
    "bank": [
        {"id": "bank_deposit", "name": "ฝากเงิน", "time": 5},
        {"id": "bank_stock", "name": "เทรดหุ้น (High Risk)", "time": 10, "is_stock": True},
        # Jobs at Bank
        {"id": "bank_job_acc", "name": "ทำบัญชี (สมุห์บัญชี)", "time": 10, "is_job": True, "req_major": "Accounting"},
        {"id": "bank_job_fin", "name": "วิเคราะห์หุ้น (นักการเงิน)", "time": 12, "is_job": True, "req_major": "Finance"},
        {"id": "bank_job_mkt", "name": "วางแผนการตลาด", "time": 15, "is_job": True, "req_major": "Marketing"}
    ],
    "park": [
        {"id": "park_jog", "name": "วิ่งเหยาะๆ (ฟรี)", "time": 12, "effects": {"health": 5, "stress": -5}},
        {"id": "park_sit", "name": "นั่งชมสวน (ฟรี)", "time": 8, "effects": {"stress": -10}},
        {"id": "park_nap", "name": "งีบใต้ต้นไม้ (ฟรี)", "time": 15, "effects": {"health": 2, "stress": -5}}
    ]
}

#### JOB DATA ####
JOB_DATA = {
    "Health": {
    #ทำงานที่รพ.หมดเลย
        "Medicine": {"job": "หมอ", "pay": 60000, "quota": 8, "time": 8, "license": True},
        "Dentist": {"job": "ทันตแพทย์", "pay": 54000, "quota": 7, "time": 10, "license": True},
        "Pharmacy": {"job": "เภสัชกร", "pay": 39000, "quota": 6, "time": 10, "license": True}
    },
    "Engineering": {
    #ทำงานที่ออฟฟิซหมดเลย
        "Civil": {"job": "วิศวกรโยธา", "pay": 36000, "quota": 6, "time": 12, "license": True},
        "Computer": {"job": "Programmer", "pay": 42000, "quota": 5, "time": 15, "license": False},
        "Electrical": {"job": "วิศวกรไฟฟ้า", "pay": 37500, "quota": 6, "time": 12, "license": True}
},
    "Business": {
    #ทำงานที่ทธนาคารหมดเลย
        "Accounting": {"job": "สมุห์บัญชี", "pay": 33000, "quota": 7, "time": 10, "license": True},
        "Finance": {"job": "นักการเงิน", "pay": 36000, "quota": 6, "time": 12, "license": False},
        "Marketing": {"job": "นักการตลาด", "pay": 28500, "quota": 5, "time": 15, "license": False}
},
    "Education": {
        "Education": {"job": "ครู", "pay": 25500, "quota": 5, "time": 12, "license": False} #ทำงานที่โรงเรียน,
        "EarlyChild": {"job": "ครูปฐมวัย", "pay": 24000, "quota": 5, "time": 10, "license": False} #ทำงานที่โรงเรียน,
        "Master_Ed": {"job": "อาจารย์มหาลัย", "pay": 45000, "quota": 4, "time": 18, "license": False} #ทำงานที่มหาลัย
},
    "Sports": {
        "Sports_Sci": {"job": "เทรนเนอร์", "pay": 24000, "quota": 6, "time": 10, "license": False} #ทำงานที่ฟิซเนส,
        "Sports_Mgmt": {"job": "ผู้จัดการทีม", "pay": 30500, "quota": 5, "time": 12, "license": False} #ทำงานที่ฟิซเนส,
        "Physical_Ed": {"job": "ครูพละ", "pay": 24000, "quota": 5, "time": 12, "license": False} #ทำงานที่โรงเรียน
},
    "Arts": {
        "Translation": {"job": "ล่าม", "pay": 27000, "quota": 5, "time": 12, "license": False}  #ทำงานที่ออฟฟิซ,
        "Linguistics": {"job": "นักภาษาศาสตร์", "pay": 25500, "quota": 5, "time": 15, "license": False}  #ทำงานที่ออฟฟิซ,
        "Master_Arts": {"job": "อาจารย์ภาษา", "pay": 48500, "quota": 4, "time": 18, "license": False} #ทำงานที่มหาลัย
},
    "Science": {
     #ทำงานที่มหาลัยหมดเลย
        "Biology": {"job": "นักวิจัย", "pay": 27000, "quota": 6, "time": 12, "license": False},
        "Physics": {"job": "นักฟิสิกส์", "pay": 30000, "quota": 6, "time": 12, "license": False},
        "Master_Sci": {"job": "อาจารย์วิทย์", "pay": 49500, "quota": 4, "time": 20, "license": False}
}

#### EVENT DATA ####
EVENT_DATA = {

    ### 🏫 HIGH SCHOOL (มัธยม: เน้นการเรียน, เพื่อน, กิจกรรม)
    
    "high_school": [
        {
            "title": "เพื่อนชวนโดดเรียน", ## ด้านสังคม
            "desc": "เพื่อนสนิทชวนไปนั่งเล่นเกมตู้ที่ห้างในเวลาเรียน เอาไงดี?",
            "choices": [
                {"text": "ไปสิ เพื่อนสำคัญที่สุด", "result_msg": "สนุกมากแต่โดนฝ่ายปกครองหมายหัว (Stress -15, Knowledge -5)", "effects": {"stress": -15, "knowledge": -5}},
                {"text": "ไม่เอาหรอก จะตั้งใจเรียน", "result_msg": "เพื่อนล้อว่าเนิร์ด แต่ความรู้แน่นปึ้ก (Knowledge +10, Stress +5)", "effects": {"knowledge": 10, "stress": 5}}
            ]
        },
        {
            "title": "สอบย่อยที่ไม่ได้อ่านมา", ## ด้านการเรียน
            "desc": "อาจารย์ประกาศสอบ Pop Quiz ทันที! คุณยังไม่ได้อ่านบทนี้เลย",
            "choices": [
                {"text": "มั่วแม่นๆ (เน้นดวง)", "result_msg": "เดาถูกเฉย! คะแนนออกมาดี (Knowledge +8)", "effects": {"knowledge": 8}},
                {"text": "แอบดูโพยในมือถือ", "result_msg": "โดนจับได้! โดนทัณฑ์บนและเครียดหนัก (Stress +20, Knowledge -5)", "effects": {"stress": 20, "knowledge": -5}}
            ]
        },
        {
            "title": "แข่งกีฬาสี", ## ด้านสุขภาพ
            "desc": "รุ่นพี่บังคับให้ลงแข่งวิ่งผลัด ทั้งที่แดดร้อนมาก",
            "choices": [
                {"text": "วิ่งเต็มที่เพื่อสีของเรา", "result_msg": "ชนะเชียร์ลีดเดอร์สะใจ! แต่เพลียแดด (Health -10, Stress -10)", "effects": {"health": -10, "stress": -10}},
                {"text": "แกล้งเจ็บขาขอไปนั่งพัก", "result_msg": "โดนรุ่นพี่เขม่น แต่ร่างกายยังสดชื่น (Health +5, Stress +5)", "effects": {"health": 5, "stress": 5}}
            ]
        },
        {
            "title": "เจอเงินตกในโรงอาหาร", ## ด้านการเงิน/โชค
            "desc": "เจอเงิน 100 บาทตกอยู่ใต้โต๊ะ ไม่มีใครเห็นเลย",
            "choices": [
                {"text": "เอาไปซื้อชานมไข่มุก!", "result_msg": "อิ่มอร่อยฟรีๆ (Money +100, Stress -5)", "effects": {"money": 100, "stress": -5}},
                {"text": "ส่งคืนครูประชาสัมพันธ์", "result_msg": "ได้รับคำชมหน้าเสาธง (Stress -10)", "effects": {"stress": -10}}
            ]
        },
        {
            "title": "งานพรอม/งานเต้นรำ", ## ด้านความสัมพันธ์
            "desc": "คนที่แอบชอบมาชวนไปเดินงานโรงเรียนด้วยกัน!",
            "choices": [
                {"text": "ไปสิ โอกาสทอง!", "result_msg": "หัวใจพองโต มีความสุขที่สุด (Stress -20, Health -2)", "effects": {"stress": -20, "health": -2}},
                {"text": "ปฏิเสธเพราะต้องอ่านหนังสือ", "result_msg": "นกไปตามระเบียบ แต่เกรดน่าจะดี (Knowledge +10, Stress +15)", "effects": {"knowledge": 10, "stress": 15}}
            ]
        }
    ],


    ### 💼 ADULT (ผู้ใหญ่: เน้นงาน, หนี้, การลงทุน, สุขภาพ)
  
    ##  ด้านการเงิน (Money) 
        {
            "title": "คริปโตกำลังมา!",
            "desc": "กูรูบอกว่าเหรียญ 'หมาบิน' กำลังจะพุ่งไปดวงจันทร์ อัดหมดตัวเลยไหม?",
            "choices": [
                {"text": "All-in! รวยทางลัด", "result_msg": "โดน Rug Pull! เงินหายเกลี้ยง (Money -3000, Stress +25)", "effects": {"money": -3000, "stress": 25}},
                {"text": "อยู่เฉยๆ ทำงานวนไป", "result_msg": "รอดตัวไป เหรียญนั่นกลายเป็น 0 ในวันถัดมา (Stress -5)", "effects": {"stress": -5}}
            ]
        },
        {
            "title": "ซ่อมบ้านด่วน",
            "desc": "ท่อน้ำที่บ้านแตก ถ้าน้ำท่วมเฟอร์นิเจอร์จะพังหมดนะ!",
            "choices": [
                {"text": "จ้างช่างด่วน (แพงหน่อย)", "result_msg": "ซ่อมเสร็จทันเวลา แต่กระเป๋าเบา (Money -1500)", "effects": {"money": -1500}},
                {"text": "ลองซ่อมเองดู", "result_msg": "ยิ่งซ่อมยิ่งพัง แถมน้ำท่วมบ้าน! (Money -2500, Stress +15)", "effects": {"money": -2500, "stress": 15}}
            ]
        },

     ##  ด้านสุขภาพ (Health) 
        {
            "title": "โปรเจกต์เผาขน",
            "desc": "บอสบอกว่าถ้าทำโปรเจกต์นี้เสร็จทันเช้าวันจันทร์ จะพิจารณาเลื่อนตำแหน่งให้",
            "choices": [
                {"text": "โต้รุ่ง 2 วันติด!", "result_msg": "งานเสร็จ แต่ภาพตัดไปโรงพยาบาล (Health -30, Stress +15, Money +2000)", "effects": {"health": -30, "stress": 15, "money": 2000}},
                {"text": "ทำเท่าที่ไหว", "result_msg": "โดนบอสบ่น แต่ยังไม่ตาย (Health +5, Stress +10)", "effects": {"health": 5, "stress": 10}}
            ]
        },
        {
            "title": "บุฟเฟ่ต์บำบัดเครียด",
            "desc": "หลังเลิกงานเครียดๆ เพื่อนชวนไปกินปิ้งย่างจุกๆ",
            "choices": [
                {"text": "จัดหนัก เยียวยาจิตใจ", "result_msg": "หายเครียดแต่คอเลสเตอรอลพุ่ง (Stress -20, Health -10, Money -599)", "effects": {"stress": -20, "health": -10, "money": -599}},
                {"text": "กลับไปกินสลัดที่บ้าน", "result_msg": "หิวแต่สุขภาพดี (Health +10, Stress +5)", "effects": {"health": 10, "stress": 5}}
            ]
        },

    ##  ด้านการงาน/ความรู้ (Career & Knowledge) 
        {
            "title": "คอร์สเรียนอัปสกิล",
            "desc": "มีคอร์สสอนใช้ AI ช่วยทำงานให้เร็วขึ้น 10 เท่า สนใจไหม?",
            "choices": [
                {"text": "ซื้อเลยเพื่ออนาคต", "result_msg": "เก่งขึ้นเยอะ ทำงานสบายขึ้นในระยะยาว (Knowledge +20, Money -1000)", "effects": {"knowledge": 20, "money": -1000}},
                {"text": "หาดูฟรีใน YouTube เอา", "result_msg": "ได้ความรู้กระปิดกระปอย (Knowledge +5)", "effects": {"knowledge": 5}}
            ]
        },
        {
            "title": "ความผิดพลาดในงาน",
            "desc": "คุณส่งอีเมลผิดกลุ่ม! ข้อมูลความลับบริษัทหลุดไปหาลูกค้า!",
            "choices": [
                {"text": "รีบแจ้งไอทีให้ดึงเมลกลับ", "result_msg": "แก้ปัญหาได้ทันแต่โดนด่ายับ (Stress +20)", "effects": {"stress": 20}},
                {"text": "แกล้งเนียนทำเป็นไม่รู้", "result_msg": "เรื่องแดงขึ้นมาทีหลัง โดนหักโบนัส! (Money -2000, Stress +30)", "effects": {"money": -2000, "stress": 30}}
            ]
        },

    ##  ด้านสังคม/ความสัมพันธ์ (Social)
        {
            "title": "ซองผ้าป่า/งานแต่ง",
            "desc": "เดือนนี้ซองมาแล้ว 4 ซอง! ทั้งงานบวช งานแต่ง งานศพ...",
            "choices": [
                {"text": "ใส่ตามมารยาททุกงาน", "result_msg": "สังคมดีเด่นแต่กินมาม่า (Money -2000, Stress -5)", "effects": {"money": -2000, "stress": -5}},
                {"text": "เลือกไปเฉพาะงานสำคัญ", "result_msg": "ประหยัดเงินได้บ้าง (Money -500, Stress +5)", "effects": {"money": -500, "stress": 5}}
            ]
        },
        {
            "title": "ดราม่าในออฟฟิศ",
            "desc": "เพื่อนร่วมงานสองคนทะเลาะกันอย่างหนักและพยายามดึงคุณเข้าพวก",
            "choices": [
                {"text": "เข้าข้างฝ่ายที่ดูมีเหตุผล", "result_msg": "ได้มิตรเพิ่ม 1 แต่ได้ศัตรูเพิ่ม 1 (Stress +10)", "effects": {"stress": 10}},
                {"text": "บอกว่า 'ไม่รู้ครับ ผมมาทำงาน'", "result_msg": "โดนมองว่าไร้น้ำใจ แต่ชีวิตสงบ (Stress -5)", "effects": {"stress": -5}}
            ]
        },

    ##  ด้านโชคลาภ (Luck/Random) 
        {
            "title": "รางวัลใหญ่จากใบเสร็จ",
            "desc": "จู่ๆ มี SMS แจ้งว่าคุณถูกรางวัลจากการช้อปปิ้ง!",
            "choices": [
                {"text": "กดลิงก์ไปรับรางวัล", "result_msg": "โดนแก๊งคอลเซ็นเตอร์ดูดเงิน! (Money -5000, Stress +40)", "effects": {"money": -5000, "stress": 40}},
                {"text": "ลบทิ้งไป มิจฉาชีพชัวร์", "result_msg": "ฉลาดมาก รอดตัวไป (Knowledge +2)", "effects": {"knowledge": 2}}
            ]
        },
        {
            "title": "สลากกินแบ่งรัฐบาล",
            "desc": "แม่ค้าเดินมาขายหวยชุดสุดท้าย 5 ใบ เลขสวยซะด้วย",
            "choices": [
                {"text": "เหมามาเลย เผื่อรวย!", "result_msg": "ถูกกินตามระเบียบ... (Money -600, Stress +5)", "effects": {"money": -600, "stress": 5}},
                {"text": "เดินหนีอย่างใจแข็ง", "result_msg": "ประหยัดเงินไปได้ (Money +0)", "effects": {}}
            ]
        }
    ]
}


#### EVENT DATA ####
    # Reset ใหม่ให้ House เป็นจุด 0
LOCATIONS_1D = {
    "home": 0,          # จุดเริ่มต้น (ขวาบน)
    "hospital": 6,      # ถัดลงมานิดเดียว
    "bank": 12,         # ขวาล่าง
    "university": 16,   # ล่างขวา
    "park": 18,         # ล่างกลาง
    "school": 20,       # ล่างซ้าย
    "office": 24,       # ซ้ายล่าง
    "mall": 30,         # ซ้ายบน
    "fastfood": 34,     # KFD (กลางบนซ้าย)
    "gym": 38           # GYM (กลางบนขวา)
}
