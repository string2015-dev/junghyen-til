# 파이썬 객체지향: 상속 (Inheritance) · IS-A vs HAS-A 정리

> 어제 배운 접근제어(`_`, `__`)·has-a(포함) 위에 **상속(is-a)** 을 얹었다.
> 중복 코드를 부모 클래스로 모으고 → 자식이 물려받고 → 필요한 것만 **오버라이딩** → 부모 기능은 **`super()`** 로 다시 부른다.
> 마지막엔 게임 캐릭터로 **상속 + 구성(has-a)을 함께** 설계했다.

## 📌 오늘의 키워드

| 키워드 | 한 줄 정리 |
|---|---|
| 상속 (Inheritance) | 부모 클래스의 속성·메서드를 자식 클래스가 **물려받아** 쓰는 것 |
| 슈퍼클래스 / 서브클래스 | 부모(상위) 클래스 / 자식(하위) 클래스 |
| `class 자식(부모):` | 상속 문법. 괄호 안에 부모 이름 |
| DRY | Don't Repeat Yourself — 같은 코드를 두 번 쓰지 마라 (상속이 필요한 이유) |
| 기능 추가 | 자식에만 있는 새 메서드 정의 (`bark()`, `honk()`) |
| 오버라이딩 (재정의) | 부모와 **같은 이름**의 메서드를 자식이 다시 정의 → 자식 것이 우선 |
| `super()` | "내 부모"를 가리킴. `super().__init__(...)`, `super().ask(...)` |
| 탐색 순서 / MRO | `자식 → 부모 → ... → object` 순으로 메서드·속성을 찾는다 (`클래스.__mro__`) |
| `_protected` vs `__private` | `_` 는 자식이 쓸 수 있고, `__` 는 이름 맹글링 때문에 못 쓴다 |
| `isinstance(객체, 클래스)` | 이 **객체**가 그 클래스(또는 그 자식)에서 만들어졌나? |
| `issubclass(자식, 부모)` | 이 **클래스**가 그 클래스를 물려받았나? |
| 다형성 (씨앗) | 같은 `speak()` 호출 → 객체마다 다른 결과 |
| IS-A / HAS-A | "~는 ~이다" → 상속 / "~가 ~를 갖는다" → 구성(composition) |

---

## 1. 왜 상속이 필요한가: 중복 제거

`OpenAIClient` 와 `GeminiClient` 를 따로 만들면 `__init__`, `_mask()`, 호출 횟수 증가 코드가 **두 클래스에 똑같이** 들어간다.

- 마스킹 규칙을 바꾸려면 **두 곳(나중엔 열 곳)** 을 모두 고쳐야 한다
- 한 곳만 고치면 → 버그가 조용히 숨는다
- 해결: 공통 부분을 **부모 클래스 하나**에 모으고 자식이 물려받는다 → 부모만 고치면 모든 자식에 반영

> 🏪 비유: 프랜차이즈 본사(부모)가 레시피를 정하면 모든 지점(자식)에 한 번에 적용, 지점은 한정 메뉴만 추가

---

## 2. 기본 상속 + 자식만의 기능 추가

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def eat(self):
        print(f"{self.name}: 밥을 먹는다 🍚")

class Dog(Animal):          # Animal 을 물려받음
    def bark(self):         # Dog 만의 메서드
        print(f"{self.name}: 멍멍! 🐶")

d = Dog("초코")             # 부모의 __init__ 이 그대로 쓰임
d.eat()                     # 부모 메서드
d.bark()                    # 자식 메서드
```

- 자식 안이 `pass` 뿐이어도 부모의 `__init__`, 메서드, 속성을 **모두** 쓸 수 있다
- **찾는 순서**: `d.eat()` → ① Dog 에서 찾고 → ② 없으면 Animal 로 올라간다 (속성도 동일)
- 형제 클래스끼리는 공유 X → `Dog` 객체로 `scratch()`(Cat 전용) 호출 시 `AttributeError`

---

## 3. 오버라이딩: 부모 메서드 덮어쓰기

```python
class Animal:
    def speak(self): print(f"{self.name}: ...")

class Dog(Animal):
    def speak(self): print(f"{self.name}: 멍멍!")    # 같은 이름 → 덮어씀

class Cat(Animal):
    def speak(self): print(f"{self.name}: 야옹~")
```

- 자식에 같은 이름이 있으면 **자식 것이 먼저 발견**되므로 부모 것은 실행되지 않는다
- **시그니처(메서드 이름 + 매개변수 구성)** 를 부모와 맞춰야 바깥에서 같은 방식으로 호출할 수 있다. 안의 로직은 자유롭게 바꾼다
- `for a in animals: a.speak()` → 같은 명령, 다른 결과 = **다형성의 씨앗**

---

## 4. `super()`: 부모 기능 다시 불러오기

### 문제: 자식이 `__init__` 을 새로 쓰면 부모 `__init__` 은 실행되지 않는다

```python
class Dog(Animal):
    def __init__(self, name, breed):
        self.breed = breed          # name 저장 안 됨!

Dog("초코", "푸들").name             # ❌ AttributeError: no attribute 'name'
```

### 해결: `super().__init__(...)` 를 **맨 윗줄**에

```python
class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)      # ① 부모 기초 공사 먼저 (name 세팅)
        self.breed = breed          # ② 자식만의 속성 추가

    def info(self):
        super().info()              # 부모 info 재사용
        print(f"품종: {self.breed}")  # + 자식 내용 추가
```

- `super().__init__()` 에는 **부모 생성자가 요구하는 인자**를 넘긴다
- 값을 고정하고 싶으면 자식에서 직접 넣는다 → `Car`: `super().__init__(name, 4)`, `Bicycle`: `super().__init__(name, 2)`
- 생성 순서: **부모 공간(속성)을 먼저 만들고 → 그 위에 자식 속성**을 더한다

---

## 5. 상속에서 `_` 와 `__` 의 차이

| 표기 | 실제 저장 이름 | 자식에서 `self.xxx` 로 접근 |
|---|---|---|
| `self.pub` | `pub` | ✅ |
| `self._prot` | `_prot` | ✅ |
| `self.__priv` | `_Parent__priv` | ❌ 자식은 `_Child__priv` 를 찾으므로 실패 |

```python
c.__dict__.keys()   # dict_keys(['pub', '_prot', '_Parent__priv'])
```

- `__` 는 **이름 맹글링**으로 `_클래스명__이름` 으로 바뀐다 → 자식 클래스 안에서 쓰면 자식 이름으로 바뀌어 못 찾음
- 그래서 **자식에게 물려줄 내부 데이터는 `_`**, 아무도 몰라야 할 것은 `__`
- 부모의 `__api_key` 는 부모 메서드(`get_masked_key()`)를 통해서만 쓴다

---

## 6. 관계 확인: `isinstance` / `issubclass` / `__mro__`

```python
isinstance(d, Dog)       # True
isinstance(d, Animal)    # True  ← 자식 객체는 부모 타입으로도 인정 ("초코는 동물이다")
isinstance(d, Cat)       # False
issubclass(Dog, Animal)  # True
issubclass(Animal, Dog)  # False ← 방향 주의

for cls in C.__mro__:    # C → B → A → object (메서드 탐색 순서)
    print(cls.__name__)
```

---

## 7. 실습: `ApiClient` 상속 구조 (중복 제거 리팩터링)

| 클래스 | 하는 일 |
|---|---|
| `ApiClient` (부모) | `service_name`(public), `__api_key`(private), `_call_count`(protected), `get_masked_key()`, `ask()`, `get_call_count()` |
| `OpenAIClient` | 서비스명 `"OpenAI"` 고정 + `model` 추가, `ask()` → `"[OpenAI/모델] 프롬프트"` |
| `GeminiClient` | 서비스명 `"Gemini"` 고정, `ask()` → `"[Gemini] 프롬프트 (안전필터 통과)"` |

**포인트**: 자식 `ask()` 를 오버라이딩해도 **호출 횟수는 늘어야 한다** → `super().ask(prompt)` 로 부모의 카운트 로직을 재사용하거나 `self._call_count += 1`(protected 라서 자식도 접근 가능)

---

## 8. 실습: 챗봇 상속 (부모 `say()` 한 번 + 자식 `_reply()` 오버라이딩)

```python
class ChatBot:
    def say(self, message):
        self.__history.append(message)
        print(f"[{self.name}] {self._reply(message)}")   # 👈 self._reply → 자식 것이 호출됨
    def _reply(self, message):
        return f"'{message}' 라고 하셨네요"

class SummaryBot(ChatBot):
    def _reply(self, message):
        return f"[요약] {message[:5]}..."
```

- `say()` 는 **부모에 한 번만** 작성 → 그 안의 `self._reply()` 는 **실제 객체(자식)의 메서드**를 찾아 실행
- 이 패턴(부모가 흐름, 자식이 세부)을 **템플릿 메서드**라고 부른다
- `__history` 는 객체마다 따로 → 번역봇 2개, 요약봇 1개처럼 각자 쌓인다

---

## 9. IS-A(상속) vs HAS-A(구성) — 게임 객체 설계

| 구분 | 상속 | 구성 |
|---|---|---|
| 관계 | IS-A: "~는 ~이다" | HAS-A: "~가 ~를 갖는다" |
| 예시 | Warrior is-a Character | Character has-a Weapon |
| 문법 | `class Warrior(Character):` | `self.weapon = Weapon(...)` |
| 언제 | 진짜 "종류" 관계일 때 (제한적으로) | 그 외 대부분 |

- **판별법**: 문장을 말해 보고 거꾸로도 말해 본다. "드래곤은 몬스터다" ✅ / "몬스터는 드래곤이다" ❌ → 드래곤이 자식
- 분류 정답: Slime–Monster, Wizard–Character, Monster–GameObject, Warrior–GameObject → **IS-A** / Character–HP·Skill·Armor, Dragon–Weapon → **HAS-A**

### 구조

```
GameObject (name, x, y, move, info)
├── Character  ──has-a──▶ HP, Weapon, Skill, Armor
│   ├── Warrior   (150, 롱소드 20, 강타 ×1.5, 판금갑옷 10)
│   └── Wizard    (90,  지팡이 8, 파이어볼 ×4.0, 로브 2)
└── Monster (+power)
    ├── Dragon
    └── Slime
```

### 잘못된 설계: `class Character(Weapon)`

- 이 코드는 "**캐릭터는 무기이다**"라는 뜻 → 말이 안 된다. "캐릭터는 무기를 **갖는다**"가 맞다
- 상속은 클래스를 정의할 때 고정되므로, 게임 중 **무기를 바꿀 수 없다** (캐릭터 자체가 무기)
- 고치기: 상속을 끊고 `self.weapon = weapon` 으로 **속성으로 보관** → `change_weapon()` 으로 갈아끼우기

> ⚠️ 상속 남용 금지: is-a 일 때만, 깊이는 2~3단까지 권장

---

## 10. 체크포인트 (실습 코드 ↔ 명세 비교)

| 항목 | 주의할 점 |
|---|---|
| `HP.damage()` | 명세는 "0 미만으로 내려가지 않음" → `max(0, current - amount)` 로 **0에 고정**. 피해가 체력보다 클 때 print만 하고 넘어가면 체력이 그대로 남는다 |
| `HP.current` | 남은 체력은 `max_hp` 를 깎지 말고 별도 `current` 로 관리 (최대 체력 정보 보존) |
| `HP.is_alive()` | `return self.current > 0` 까지 구현 |
| `take_damage()` | 방어력이 더 커도 **0을 return** 해야 출력이 `None` 이 안 된다 |
| `Character.__init__` | 명세는 `max_hp`(숫자)를 받아 안에서 `HP(max_hp)` 생성 |
| `GeminiClient.ask()` | 명세 문자열 끝 `(안전필터 통과)` 빠뜨리지 않기 |
| `SummaryBot` | 명세는 `앞5글자...` → `...` 포함 |
| 오버라이딩 설명 | "시그니처"는 로직이 아니라 **이름+매개변수**. 로직은 바꾸는 게 오버라이딩 |
| `try: d.scracth()` | 오타여도 AttributeError 는 나지만, 확인하려던 건 `scratch` |
| `super()` 없을 때 | 부모 `__init__` 이 **실행되지 않아** 부모 속성(`name`)이 생기지 않는다 |

---

## 11. 🗂️ 수업 중 메모: GitHub · 미니 프로젝트

- **GitHub 계정 분리**: 포트폴리오용(취업) / 제출·연습용
- 포트폴리오 계정은 **프로젝트 위주**, README 필수
  - README **맨 위에 소감**: 뭘 배웠고, 뭘 알게 됐고, 뭐가 어려웠고, 지금 어떻게 하고 있는지
  - DB 쓰는 프로젝트면 ERD 첨부
  - README는 compact하게, 길어지면 노션으로 빼서 URL 연결
- 오늘 workshop 실습도 GitHub에 정리하기
- **팀별 미니 프로젝트 (팀 세미나)**: Object 실습 4 이후 → 세부 기능 추가, RFP를 받아 **커맨드와 기능 연결**, 코드 작성 후 **다이어그램을 보고 설명**할 수 있게
- 추세: 작은 클래스(컴포넌트)를 모아 확장하는 **경량화** 구조 (모바일 생태계라 무거우면 안 됨)

---

## 12. 🔁 오늘 배운 전체 흐름 한눈에 보기

```python
# =====================================================================
# 오늘의 흐름
#   ① 중복 코드 → 부모 클래스로 모은다 (DRY)
#   ② 자식은 class 자식(부모): 로 물려받는다
#   ③ 다르게 동작할 메서드만 오버라이딩
#   ④ 부모 기능이 필요하면 super() 로 다시 부른다
#   ⑤ is-a 는 상속, has-a 는 구성(속성으로 보관)
# =====================================================================


# ---------------------------------------------------------------------
# PART 1. ApiClient : 상속으로 중복 제거 + 접근제어 복습
# ---------------------------------------------------------------------
class ApiClient:                                   # 부모(슈퍼클래스)
    def __init__(self, service_name, api_key):
        self.service_name = service_name           # public    : 누구나
        self.__api_key = api_key                   # private   : _ApiClient__api_key 로 저장됨
        self._call_count = 0                       # protected : 자식도 쓸 수 있음

    def get_masked_key(self):
        # private 은 "부모 메서드 안에서만" 다룬다 → 자식은 이 메서드를 물려받아 사용
        return self.__api_key[:4] + "*" * (len(self.__api_key) - 4)

    def ask(self, prompt):
        self._call_count += 1                      # 공통 로직: 호출 횟수 증가
        return f"[{self.service_name}] 기본 응답: {prompt}"

    def get_call_count(self):
        return self._call_count


class OpenAIClient(ApiClient):                     # 자식(서브클래스): ApiClient 를 물려받음
    def __init__(self, api_key, model="gpt-4"):
        super().__init__("OpenAI", api_key)        # ① 부모 생성자 먼저 (서비스명은 고정값으로)
        self.model = model                         # ② 자식만의 속성 추가

    def ask(self, prompt):                         # 오버라이딩 (이름·매개변수는 부모와 동일)
        super().ask(prompt)                        # 부모 ask 실행 → 호출 횟수 증가 재사용
        return f"[{self.service_name}/{self.model}] {prompt}"   # 반환값만 자식 방식으로


class GeminiClient(ApiClient):
    def __init__(self, api_key):
        super().__init__("Gemini", api_key)

    def ask(self, prompt):
        self._call_count += 1                      # 방법 2: protected 라서 자식이 직접 증가 가능
        return f"[{self.service_name}] {prompt} (안전필터 통과)"


gpt = OpenAIClient("sk-aaaa1111", model="gpt-4o")
gem = GeminiClient("gm-bbbb2222")

print(gpt.ask("파이썬이 뭐야?"))                    # [OpenAI/gpt-4o] 파이썬이 뭐야?
print(gpt.ask("한 줄 요약해줘"))
print(gem.ask("파이썬이 뭐야?"))                    # [Gemini] 파이썬이 뭐야? (안전필터 통과)
print("키 마스킹:", gpt.get_masked_key())           # sk-a*******  ← 부모 메서드를 그대로 사용
print("GPT 호출수:", gpt.get_call_count(),          # 2 ← 오버라이딩해도 카운트 유지
      "/ Gemini 호출수:", gem.get_call_count())      # 1 ← 객체마다 따로 센다
print("isinstance:", isinstance(gpt, ApiClient), isinstance(gem, ApiClient))  # True True

try:
    print(gpt.__api_key)                           # 바깥에서 private 직접 접근
except AttributeError as e:
    print("private 접근 실패:", e)                  # 이름이 _ApiClient__api_key 로 바뀌어 있음

print([c.__name__ for c in OpenAIClient.__mro__])  # ['OpenAIClient', 'ApiClient', 'object'] 탐색 순서


# ---------------------------------------------------------------------
# PART 2. 게임 객체 : 상속(IS-A) + 구성(HAS-A) 함께 쓰기
# ---------------------------------------------------------------------
class GameObject:                                  # 최상위 부모: 모든 게임 객체의 공통 부분
    def __init__(self, name):
        self.name = name
        self.x = 0
        self.y = 0

    def move(self, dx, dy):
        self.x += dx
        self.y += dy

    def info(self):
        return f"{self.name} ({self.x}, {self.y})"


# --- 부품 클래스들 (Character 가 "가지는" 것들 → HAS-A) ---
class HP:
    def __init__(self, max_hp):
        self.max_hp = max_hp                       # 최대 체력은 보존
        self.current = max_hp                      # 현재 체력은 따로 관리

    def damage(self, amount):
        self.current = max(0, self.current - amount)   # 0 미만으로 내려가지 않음

    def is_alive(self):
        return self.current > 0


class Weapon:
    def __init__(self, name, power):
        self.name = name
        self.power = power


class Skill:
    def __init__(self, name, multiplier):
        self.name = name
        self.multiplier = multiplier


class Armor:
    def __init__(self, name, defense):
        self.name = name
        self.defense = defense


class Character(GameObject):                       # IS-A : 캐릭터는 게임 객체다
    def __init__(self, name, max_hp, weapon, skill, armor):
        super().__init__(name)                     # name, x, y 는 부모가 세팅
        self.hp = HP(max_hp)                       # HAS-A : 부품 객체를 속성으로 보관
        self.weapon = weapon
        self.skill = skill
        self.armor = armor

    def attack(self):
        return self.weapon.power                   # 계산은 부품에게 맡긴다 (위임)

    def use_skill(self):
        return int(self.attack() * self.skill.multiplier)

    def take_damage(self, amount):
        real = max(0, amount - self.armor.defense) # 방어력만큼 감소, 음수 방지
        self.hp.damage(real)
        return real                                # 항상 숫자 반환 (None 방지)

    def change_weapon(self, weapon):
        self.weapon = weapon                       # HAS-A 라서 갈아끼우기가 쉽다


class Warrior(Character):                          # 직업 = 값만 고정한 자식 클래스
    def __init__(self, name):
        super().__init__(name, 150, Weapon("롱소드", 20), Skill("강타", 1.5), Armor("판금갑옷", 10))


class Wizard(Character):
    def __init__(self, name):
        super().__init__(name, 90, Weapon("지팡이", 8), Skill("파이어볼", 4.0), Armor("로브", 2))


class Monster(GameObject):                         # IS-A : 몬스터는 게임 객체다
    def __init__(self, name, power):
        super().__init__(name)
        self.power = power                         # 몬스터만의 속성 추가


class Dragon(Monster):                             # 드래곤은 몬스터다 (거꾸로는 X)
    pass


# --- 실행 ---
d = Dragon("드래곤", 50)
d.move(3, 4)                                       # Dragon → Monster → GameObject 에서 move 발견
print(d.info())                                    # 드래곤 (3, 4)
print(isinstance(d, Monster), isinstance(d, GameObject), isinstance(d, Character))  # True True False

w = Warrior("아서")
z = Wizard("멀린")
print(w.attack(), w.use_skill(), z.use_skill())    # 20 30 32
print(w.take_damage(25), w.hp.current)             # 15 135  (25 - 방어 10)
print(z.take_damage(25), z.hp.current)             # 23 67   (25 - 방어 2)
w.change_weapon(Weapon("전투도끼", 30))
print(w.attack(), w.use_skill())                   # 30 45
print(z.take_damage(999), z.hp.current, z.hp.is_alive())   # 997 0 False ← 0에서 멈춤
```

**실행 결과**

```
[OpenAI/gpt-4o] 파이썬이 뭐야?
[OpenAI/gpt-4o] 한 줄 요약해줘
[Gemini] 파이썬이 뭐야? (안전필터 통과)
키 마스킹: sk-a*******
GPT 호출수: 2 / Gemini 호출수: 1
isinstance: True True
private 접근 실패: 'OpenAIClient' object has no attribute '__api_key'
['OpenAIClient', 'ApiClient', 'object']
드래곤 (3, 4)
True True False
20 30 32
15 135
23 67
30 45
997 0 False
```
