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

        self.abnormal = {}
        self.visited_over_25 = {k: False for k in self.stats}

        self.rent_timer = 0
        self.clothes_timer = 0    

#### STAT SYSTEM ####
  class StatSystem:

    @staticmethod
    def clamp(v):
        return max(Config.MIN_STAT, min(v, Config.MAX_STAT))

    @staticmethod
    def update(player, stat, value):
        player.stats[stat] += value
        player.stats[stat] = StatSystem.clamp(player.stats[stat])

        if player.stats[stat] > 25:
            player.visited_over_25[stat] = True

        StatSystem.update_abnormal(player, stat)

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
        cur = player.location

        if dest not in MapSystem.DISTANCE[cur]:
            return False

        cost = MapSystem.DISTANCE[cur][dest]

        if player.time < cost:
            return False

        player.time -= cost
        player.location = dest
        return True

### JOB SYSTEM ###
class JobSystem:

    @staticmethod
    def salary(player, job):
        base = job["base_salary"]
        mult = Config.DEGREE_MULTIPLIER[player.education["degree"]]

        salary = base * mult

        if job.get("license_bonus"):
            if player.education["major"] in ["medicine","engineering"]:
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
  class TurnSystem:

    @staticmethod
    def end_turn(player):

        # passive
        if player.stats["stress"] >= 60:
            StatSystem.update(player, "health", -5)

        # rent
        player.rent_timer += 1
        if player.rent_timer >= 2:
            player.stats["money"] -= 500
            player.rent_timer = 0

        # clothes
        player.clothes_timer += 1
        if player.clothes_timer >= 4:
            StatSystem.update(player, "health", -10)

        player.turn += 1
        player.time = Config.TURN_TIME
