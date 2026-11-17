# 파이썬 객체지향 실습: ERD 기반 클래스 설계 (has-a · has-many) 정리

> 어제 배운 class · 생성자 복습 + **ERD(설계도)만 보고 클래스를 직접 짜는** 실습.
> 클래스 1개 → 2개(has-a) → 3개(has-many) → 5개(회사·부서까지)로 단계적으로 확장했다.

## 📌 오늘의 키워드

| 키워드 | 한 줄 정리 |
|---|---|
| ERD / 클래스 다이어그램 | 클래스의 **속성·메서드·관계**를 그린 설계도 → 보고 그대로 코드로 옮긴다 |
| 하드코딩 | 입력 없이 데이터를 코드에 직접 적어 객체를 만드는 방식 |
| 메서드 호출 체인 | `get_net()` → `get_tax()` → `get_total()` 처럼 메서드가 **같은 객체의 다른 메서드를 `self.`로 재사용** |
| 객체 리스트 | `employees = [Employee(...), ...]` → `for` 로 돌며 같은 메서드 호출 |
| has-a (포함, 1:1) | 객체가 **다른 객체 1개를 속성으로** 가짐 (`self.salary = Salary(...)`) |
| has-many (1:N) | 객체가 **다른 객체 여러 개를 리스트로** 가짐 (`self.histories = [WorkHistory, ...]`) |
| Composition (구성) | 작은 클래스를 조립해 큰 클래스를 만드는 설계 방식 |
| 위임 (delegation) | 계산은 직접 하지 않고 **그 일을 맡은 객체에게** 시킨다 (`self.salary.get_total()`) |
| 단일 책임 | 클래스 하나는 **책임 하나만** (Salary는 급여 계산만) |
| `None` 기본값 패턴 | `histories=None` → 안에서 `[]` 생성. 기본값에 `[]` 직접 쓰면 객체끼리 리스트 공유 버그 |
| 이중 반복 | 부서 순회 → 그 안에서 사원 순회 (`print_report`) |
| 집계 | `sum(제너레이터)` 로 하위 객체 값 합산, 평균은 **0으로 나누기 방지** |

---

## 1. 단일 클래스: Employee (사원 급여 명세서)

- ERD: 속성 `name, base_pay, bonus, overtime` / 메서드 `get_total, get_tax, get_net, to_string`
- 핵심은 **메서드 호출 체인** → 같은 계산을 다시 쓰지 않고 `self.메서드()` 로 재사용

```python
def get_total(self): return self.base_pay + self.bonus + self.overtime
def get_tax(self):   return self.get_total() * 0.1          # get_total 재사용
def get_net(self):   return self.get_total() - self.get_tax()
```

- 객체 5개를 리스트에 담고 `for emp in employees: print(emp.to_string())`
- `to_string()` 은 **출력용 문자열을 return** (print는 바깥에서) → `\t` 로 열 맞춤

---

## 2. has-a 관계: Employee ─▶ Salary (1:1)

- 급여 계산을 `Salary` 클래스로 **분리** → Employee는 사원 정보만, Salary는 계산만 (단일 책임)
- **포함되는 클래스(Salary)를 먼저 정의**해야 한다

```python
class Employee:
    def __init__(self, name, dept, salary):
        self.salary = salary                 # Salary 객체를 속성으로 보관

    def to_string(self):
        return f"{self.name}\t{self.salary.get_total()}"   # 객체.속성.메서드()
```

- 생성: `Employee('김민수', '개발팀', Salary(300, 50, 20))` → 객체 안에 객체
- `emp.salary.get_total()` = "사원**의** 급여**의** 총액"

---

## 3. has-many 관계: Employee ─▶ WorkHistory (1:N)

- 사원은 근무이력을 **0건 이상** 가짐 → **리스트** 속성
- 추가 메서드 `add_history()` / 합산 메서드 `get_career()`

```python
def __init__(self, name, dept, salary, histories=None):
    self.histories = histories if histories is not None else []   # ✅ None 패턴

def add_history(self, history):
    self.histories.append(history)

def get_career(self):
    return sum(h.get_years() for h in self.histories)   # 이력 없으면 0
```

- **왜 `histories=[]` 가 아니라 `None`?** 기본값은 함수 정의 시 **한 번만** 만들어져서, `[]` 를 쓰면 모든 사원이 **같은 리스트**를 공유한다
- 신입(이력 없음)은 `len(emp.histories) == 0` 으로 구분 가능

---

## 4. 확장: Company ─▶ Department ─▶ Employee (5단 구조)

| 관계 | 속성 | 종류 |
|---|---|---|
| Company → Department | `departments` | 1 : N (리스트) |
| Department → Employee | `employees` | 1 : N (리스트) |
| Employee → WorkHistory | `histories` | 1 : N (리스트) |
| Employee → Salary | `salary` | 1 : 1 (객체) |

- **Department**: `add_employee(emp)` 하는 순간 `emp.dept = self.name` 으로 **부서 자동 지정**, `get_payroll()`(급여 총액), `get_avg_career()`(평균 경력)
- **Company**: `get_employee_count()`, `get_payroll()` → 부서의 `get_payroll()` 을 다시 합산 (위임의 연쇄)
- **Employee 변경점**: 생성 시 부서 없이 `Employee(이름, Salary(...))`, `dept` 는 빈 문자열로 시작

### 생성은 아래에서 위로, 호출은 위에서 아래로

```
생성: Salary → Employee → add_history → Department.add_employee → Company.add_department
호출: Company.print_report → Department.print_members → Employee.to_string
                                                       ├ get_career → WorkHistory.get_years
                                                       └ salary     → Salary.get_total/tax/net
```

### 예상 결과 (명세 기준)

| 부서 | 인원 | 급여 총액 | 평균 경력 |
|---|---|---|---|
| 개발팀 | 2 | 775 | 9.0 |
| 인사팀 | 1 | 390 | 4.0 |
| 영업팀 | 1 | 350 | 0.0 |
| 마케팅팀 | 1 | 385 | 3.0 |
| 합계 | 5 | 1900 | – |

---

## 5. 체크포인트 (실습 코드 ↔ 명세 비교)

| 항목 | 주의할 점 |
|---|---|
| `get_career()` 를 for문으로 쓸 때 | `return total` 은 **for문 밖**. 안에 두면 첫 이력만 합산, 이력 없으면 `None` |
| `get_avg_career()` | 사원 0명이면 `return 0` (ZeroDivisionError 방지) |
| 부서 `get_payroll()` | 명세는 `salary.get_total()` 합계. `get_net()` 으로 합하면 개발팀이 775가 아니라 697.5 |
| 부서 지정 | 명세는 `add_employee()` 가 `dept` 를 지정. 생성자에 dept를 넣고 이름 비교로 배치하는 방식도 동작은 하지만, 부서가 사원을 "소유"하는 관계가 코드에 드러나지 않음 |
| 메서드 이름 | ERD는 `get_years()` → ERD와 이름을 **똑같이** 맞추는 습관 |
| 예외 처리 범위 | 음수 급여, `end_year < start_year`, 한 사원을 두 부서에 추가 → 현재는 검사하지 않음 |

---

## 6. 🔁 오늘 배운 전체 흐름 한눈에 보기

```python
# =====================================================================
# 0) 설계 원칙: "포함되는 쪽(아래 단계) 클래스부터" 정의한다
#    Salary, WorkHistory → Employee → Department → Company
#    각 클래스는 책임 1개만, 계산은 아래 단계 객체에게 맡긴다(위임)
# =====================================================================


# ---------------------------------------------------------------------
# 1) Salary (급여) : 급여 항목 보관 + 총액/세금/실수령액 계산만 담당
#    Employee → Salary  =  1 : 1 (has-a, 객체 1개)
# ---------------------------------------------------------------------
class Salary:
    def __init__(self, base_pay, bonus, overtime):
        self.base_pay = base_pay      # 기본급
        self.bonus = bonus            # 상여금
        self.overtime = overtime      # 야근수당

    def get_total(self):
        """세 항목 합"""
        return self.base_pay + self.bonus + self.overtime

    def get_tax(self):
        """총액 × 0.1  → 내 다른 메서드를 self.로 재사용"""
        return self.get_total() * 0.1

    def get_net(self):
        """총액 - 세금 (메서드 호출 체인: get_net → get_tax → get_total)"""
        return self.get_total() - self.get_tax()


# ---------------------------------------------------------------------
# 2) WorkHistory (근무이력) : 이전 직장 1건 + 근무 연수 계산
#    Employee → WorkHistory  =  1 : N (has-many, 리스트)
# ---------------------------------------------------------------------
class WorkHistory:
    def __init__(self, company, position, start_year, end_year):
        self.company = company
        self.position = position
        self.start_year = start_year
        self.end_year = end_year

    def get_years(self):
        return self.end_year - self.start_year

    def to_string(self):
        return f"{self.company}\t{self.position}\t{self.start_year}~{self.end_year}\t{self.get_years()}년"


# ---------------------------------------------------------------------
# 3) Employee (사원) : Salary 1개 + WorkHistory 여러 개를 "가진다"
#    - dept 는 생성 시 빈 문자열 → 부서에 배치될 때 정해진다
#    - histories=None 패턴: 기본값으로 []를 직접 쓰면 모든 객체가
#      같은 리스트를 공유하는 버그 → None 으로 받고 안에서 새로 만든다
# ---------------------------------------------------------------------
class Employee:
    def __init__(self, name, salary, histories=None):
        self.name = name
        self.dept = ""                       # add_employee() 에서 지정
        self.salary = salary                 # Salary 객체 (has-a)
        self.histories = histories if histories is not None else []

    def add_history(self, history):
        self.histories.append(history)       # WorkHistory 객체 추가

    def get_career(self):
        total = 0
        for h in self.histories:
            total += h.get_years()           # 이력마다 get_years() 호출
        return total                         # ⚠️ for 문 "밖"에서 return

    def to_string(self):
        s = self.salary                      # 계산은 Salary 에게 위임
        return f"{self.name}\t{self.get_career()}년\t{s.get_total()}\t{s.get_tax()}\t{s.get_net()}"


# ---------------------------------------------------------------------
# 4) Department (부서) : 사원 목록 관리 + 부서 단위 집계
#    Department → Employee  =  1 : N
# ---------------------------------------------------------------------
class Department:
    def __init__(self, name):
        self.name = name
        self.employees = []

    def add_employee(self, emp):
        emp.dept = self.name                 # 부서에 넣는 순간 사원의 dept 자동 지정
        self.employees.append(emp)

    def get_payroll(self):
        return sum(e.salary.get_total() for e in self.employees)

    def get_avg_career(self):
        if not self.employees:               # 사원 0명 → 0으로 나누기 방지
            return 0
        return sum(e.get_career() for e in self.employees) / len(self.employees)

    def print_members(self):
        print(f"[{self.name}] {len(self.employees)}명 | 급여 총액 {self.get_payroll()} | 평균 경력 {self.get_avg_career():.1f}년")
        print("  이름\t경력\t총액\t세금\t실수령액")
        for e in self.employees:             # 안쪽 반복: 사원 순회
            print("  " + e.to_string())


# ---------------------------------------------------------------------
# 5) Company (회사) : 부서 목록 관리 + 회사 전체 집계/보고서
#    Company → Department  =  1 : N
# ---------------------------------------------------------------------
class Company:
    def __init__(self, name):
        self.name = name
        self.departments = []

    def add_department(self, dept):
        self.departments.append(dept)

    def get_employee_count(self):
        return sum(len(d.employees) for d in self.departments)

    def get_payroll(self):
        return sum(d.get_payroll() for d in self.departments)   # 부서에게 위임

    def print_report(self):
        print(f"===== {self.name} 사원 현황 =====")
        for d in self.departments:           # 바깥 반복: 부서 순회
            d.print_members()                #   → 안에서 사원 순회 (이중 반복)
        print(f"전체 사원 {self.get_employee_count()}명 | 전체 급여 총액 {self.get_payroll()}")


# =====================================================================
# 6) 객체 생성은 "아래 단계부터" : Salary → Employee → 이력 → 부서 → 회사
# =====================================================================
kim  = Employee("김민수", Salary(300, 50, 20))
choi = Employee("최서연", Salary(350, 30, 25))
lee  = Employee("이지은", Salary(320, 40, 30))
park = Employee("박준호", Salary(280, 60, 10))      # 신입 → 이력 없음
han  = Employee("한도윤", Salary(300, 70, 15))

kim.add_history(WorkHistory("네오소프트", "사원", 2015, 2018))
kim.add_history(WorkHistory("한빛시스템", "대리", 2018, 2022))
choi.add_history(WorkHistory("데이터랩", "연구원", 2012, 2016))
choi.add_history(WorkHistory("클라우드원", "선임", 2016, 2020))
choi.add_history(WorkHistory("넥스트AI", "책임", 2020, 2023))
lee.add_history(WorkHistory("그린HR", "사원", 2017, 2021))
han.add_history(WorkHistory("브랜드웍스", "사원", 2019, 2022))

dev, hr, sales, mkt = Department("개발팀"), Department("인사팀"), Department("영업팀"), Department("마케팅팀")
dev.add_employee(kim)
dev.add_employee(choi)
hr.add_employee(lee)
sales.add_employee(park)
mkt.add_employee(han)

company = Company("블루웨이브")
for d in [dev, hr, sales, mkt]:
    company.add_department(d)

# =====================================================================
# 7) 보고서 호출은 "위에서 아래로"
#    Company.print_report → Department.print_members → Employee.to_string
#                                                     ├ get_career → WorkHistory.get_years
#                                                     └ salary     → Salary.get_total/tax/net
# =====================================================================
company.print_report()
```

**실행 결과**

```
===== 블루웨이브 사원 현황 =====
[개발팀] 2명 | 급여 총액 775 | 평균 경력 9.0년
  이름	경력	총액	세금	실수령액
  김민수	7년	370	37.0	333.0
  최서연	11년	405	40.5	364.5
[인사팀] 1명 | 급여 총액 390 | 평균 경력 4.0년
  이름	경력	총액	세금	실수령액
  이지은	4년	390	39.0	351.0
[영업팀] 1명 | 급여 총액 350 | 평균 경력 0.0년
  이름	경력	총액	세금	실수령액
  박준호	0년	350	35.0	315.0
[마케팅팀] 1명 | 급여 총액 385 | 평균 경력 3.0년
  이름	경력	총액	세금	실수령액
  한도윤	3년	385	38.5	346.5
전체 사원 5명 | 전체 급여 총액 1900
```
