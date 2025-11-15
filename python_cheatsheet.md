# Полная шпаргалка по Python

## 📋 Содержание
- [Соглашения об именовании](#соглашения-об-именовании)
- [Типы данных](#типы-данных)
- [Переменные и константы](#переменные-и-константы)
- [Операторы](#операторы)
- [Условия](#условия)
- [Циклы](#циклы)
- [Функции](#функции)
- [Классы и ООП](#классы-и-ооп)
- [Работа со строками](#работа-со-строками)
- [Коллекции](#коллекции)
- [Работа с файлами](#работа-с-файлами)
- [Обработка исключений](#обработка-исключений)
- [Модули и пакеты](#модули-и-пакеты)
- [Comprehensions](#comprehensions)
- [Декораторы](#декораторы)
- [Контекстные менеджеры](#контекстные-менеджеры)
- [Работа с датой и временем](#работа-с-датой-и-временем)
- [Регулярные выражения](#регулярные-выражения)
- [Асинхронное программирование](#асинхронное-программирование)
- [Лучшие практики](#лучшие-практики)

---

## Соглашения об именовании

### PEP 8 - Style Guide

```python
# ПЕРЕМЕННЫЕ: snake_case (строчные буквы с подчеркиванием)
user_name = "John"
total_count = 42
is_active = True

# КОНСТАНТЫ: SCREAMING_SNAKE_CASE (заглавные буквы с подчеркиванием)
MAX_SIZE = 100
DEFAULT_TIMEOUT = 30
API_KEY = "secret_key"

# ФУНКЦИИ: snake_case
def calculate_total(items):
    pass

def get_user_by_id(user_id):
    pass

# КЛАССЫ: PascalCase (каждое слово с заглавной буквы)
class UserProfile:
    pass

class HTTPRequestHandler:
    pass

# МОДУЛИ И ПАКЕТЫ: snake_case (короткие имена)
# user_manager.py
# database_connector.py

# ПРИВАТНЫЕ переменные/методы: начинаются с _
class MyClass:
    def __init__(self):
        self._internal_value = 0  # защищенный атрибут
        self.__private_value = 1  # приватный атрибут (name mangling)
    
    def _helper_method(self):  # защищенный метод
        pass

# Избегайте использования:
# - одиночных букв (кроме счетчиков: i, j, k)
# - l, O (легко спутать с 1 и 0)
# - зарезервированных слов Python
```

---

## Типы данных

### Базовые типы

```python
# Числа
integer_num = 42                    # int
float_num = 3.14                    # float
complex_num = 1 + 2j                # complex
binary_num = 0b1010                 # 10 в десятичной (бинарный)
octal_num = 0o12                    # 10 в десятичной (восьмеричный)
hex_num = 0xA                       # 10 в десятичной (шестнадцатеричный)

# Строки
single_quote = 'Hello'
double_quote = "World"
multiline = """Многострочная
строка"""
raw_string = r"C:\new\path"         # сырая строка (игнорирует escape)
f_string = f"Value: {integer_num}"  # форматированная строка

# Логический тип
is_true = True
is_false = False

# None (отсутствие значения)
empty_value = None

# Проверка типа
type(integer_num)                   # <class 'int'>
isinstance(integer_num, int)        # True
```

### Преобразование типов

```python
# В строку
str(42)                 # "42"
str(3.14)               # "3.14"

# В число
int("42")               # 42
int(3.14)               # 3 (отбрасывает дробную часть)
int("FF", 16)           # 255 (из шестнадцатеричной)
float("3.14")           # 3.14

# В логический тип
bool(1)                 # True
bool(0)                 # False
bool("")                # False
bool("text")            # True
bool([])                # False
bool([1, 2])            # True

# Проверки
isinstance(value, (int, float))  # несколько типов
```

---

## Переменные и константы

```python
# Простое присваивание
x = 10
name = "Alice"

# Множественное присваивание
a, b, c = 1, 2, 3
x = y = z = 0

# Обмен значений
a, b = b, a

# Распаковка
first, *rest = [1, 2, 3, 4, 5]      # first=1, rest=[2,3,4,5]
*start, last = [1, 2, 3, 4, 5]      # start=[1,2,3,4], last=5
first, *middle, last = [1, 2, 3, 4] # first=1, middle=[2,3], last=4

# Аннотации типов (type hints)
age: int = 25
name: str = "Bob"
scores: list[int] = [95, 87, 92]
user_data: dict[str, str] = {"name": "Alice"}
optional_value: int | None = None   # Python 3.10+

# Глобальные и локальные переменные
global_var = "I'm global"

def my_function():
    local_var = "I'm local"
    global global_var
    global_var = "Modified global"
    
# Константы (по соглашению, не enforced в Python)
MAX_CONNECTIONS = 100
PI = 3.14159
```

---

## Операторы

### Арифметические операторы

```python
a, b = 10, 3

a + b       # 13 - сложение
a - b       # 7  - вычитание
a * b       # 30 - умножение
a / b       # 3.333... - деление (всегда float)
a // b      # 3  - целочисленное деление
a % b       # 1  - остаток от деления (модуло)
a ** b      # 1000 - возведение в степень
-a          # -10 - унарный минус
+a          # 10  - унарный плюс
abs(-10)    # 10  - модуль числа

# Составное присваивание
x = 5
x += 3      # x = x + 3 → 8
x -= 2      # x = x - 2 → 6
x *= 4      # x = x * 4 → 24
x /= 3      # x = x / 3 → 8.0
x //= 2     # x = x // 2 → 4.0
x %= 3      # x = x % 3 → 1.0
x **= 2     # x = x ** 2 → 1.0
```

### Операторы сравнения

```python
a, b = 5, 3

a == b      # False - равно
a != b      # True  - не равно
a > b       # True  - больше
a < b       # False - меньше
a >= b      # True  - больше или равно
a <= b      # False - меньше или равно

# Цепочки сравнений
1 < x < 10  # True если x между 1 и 10
a == b == c # True если все равны

# is vs ==
a = [1, 2, 3]
b = [1, 2, 3]
c = a

a == b      # True  (одинаковые значения)
a is b      # False (разные объекты)
a is c      # True  (тот же объект)

# Сравнение с None
x = None
x is None       # ✅ Правильно
x == None       # ❌ Не рекомендуется
```

### Логические операторы

```python
# and, or, not
x = True
y = False

x and y     # False
x or y      # True
not x       # False

# Короткое замыкание (short-circuit)
def expensive_check():
    print("Вызвана дорогая функция")
    return True

False and expensive_check()  # Не вызовет функцию
True or expensive_check()    # Не вызовет функцию

# Тернарный оператор
result = "Yes" if condition else "No"
max_value = a if a > b else b
```

### Операторы принадлежности и идентичности

```python
# in, not in
fruits = ["apple", "banana", "cherry"]
"apple" in fruits           # True
"orange" not in fruits      # True

# Работает со строками
"ell" in "hello"            # True

# Работает со словарями (проверяет ключи)
user = {"name": "Alice", "age": 30}
"name" in user              # True
"Alice" in user             # False (не значение)

# is, is not
a = 256
b = 256
a is b          # True (для маленьких чисел Python кеширует)

x = 1000
y = 1000
x is y          # False (для больших чисел создаются разные объекты)
```

### Битовые операторы

```python
a = 60  # 0011 1100
b = 13  # 0000 1101

a & b   # 12 (0000 1100) - AND
a | b   # 61 (0011 1101) - OR
a ^ b   # 49 (0011 0001) - XOR
~a      # -61            - NOT (инверсия)
a << 2  # 240            - сдвиг влево
a >> 2  # 15             - сдвиг вправо
```

---

## Условия

```python
# Базовый if-elif-else
age = 18

if age < 13:
    print("Ребенок")
elif age < 18:
    print("Подросток")
elif age < 65:
    print("Взрослый")
else:
    print("Пенсионер")

# Условие в одну строку
status = "active" if user.is_logged_in else "inactive"

# Множественные условия
if x > 0 and x < 100:
    print("x в диапазоне 0-100")

if username and password:  # проверка на "truthy"
    login(username, password)

# Match-case (Python 3.10+)
def http_status(status):
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500 | 502 | 503:  # несколько значений
            return "Server Error"
        case _:  # default
            return "Unknown"

# Pattern matching со структурами
def process_point(point):
    match point:
        case (0, 0):
            print("Начало координат")
        case (0, y):
            print(f"На оси Y: {y}")
        case (x, 0):
            print(f"На оси X: {x}")
        case (x, y):
            print(f"Точка: ({x}, {y})")

# Walrus operator := (Python 3.8+)
if (n := len(data)) > 10:
    print(f"Слишком много данных: {n}")
```

---

## Циклы

### Цикл for

```python
# Итерация по последовательности
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit)

# range() для чисел
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)

for i in range(1, 6):       # 1, 2, 3, 4, 5
    print(i)

for i in range(0, 10, 2):   # 0, 2, 4, 6, 8 (шаг 2)
    print(i)

for i in range(10, 0, -1):  # 10, 9, 8, ..., 1 (обратный порядок)
    print(i)

# enumerate() для индекса и значения
for index, fruit in enumerate(fruits):
    print(f"{index}: {fruit}")

for index, fruit in enumerate(fruits, start=1):  # начать с 1
    print(f"{index}: {fruit}")

# Итерация по словарю
user = {"name": "Alice", "age": 30, "city": "NYC"}

for key in user:                        # только ключи
    print(key)

for value in user.values():             # только значения
    print(value)

for key, value in user.items():         # ключи и значения
    print(f"{key}: {value}")

# zip() для параллельной итерации
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]

for name, age in zip(names, ages):
    print(f"{name} is {age} years old")

# Вложенные циклы
for i in range(3):
    for j in range(3):
        print(f"({i}, {j})")
```

### Цикл while

```python
# Базовый while
count = 0
while count < 5:
    print(count)
    count += 1

# while с else
i = 0
while i < 5:
    print(i)
    i += 1
else:
    print("Цикл завершен")  # выполнится, если не было break

# Бесконечный цикл
while True:
    user_input = input("Введите 'q' для выхода: ")
    if user_input == 'q':
        break
    print(f"Вы ввели: {user_input}")
```

### Управление циклами

```python
# break - прерывает цикл
for i in range(10):
    if i == 5:
        break
    print(i)  # выведет 0, 1, 2, 3, 4

# continue - пропускает текущую итерацию
for i in range(10):
    if i % 2 == 0:
        continue
    print(i)  # выведет только нечетные: 1, 3, 5, 7, 9

# pass - ничего не делает (заглушка)
for i in range(5):
    pass  # TODO: добавить логику позже

# else в циклах (выполняется, если не было break)
for i in range(5):
    if i == 10:
        break
else:
    print("Цикл завершился нормально")
```

---

## Функции

### Определение функций

```python
# Простая функция
def greet():
    print("Hello!")

greet()  # вызов функции

# Функция с параметрами
def greet_user(name):
    print(f"Hello, {name}!")

greet_user("Alice")

# Функция с возвращаемым значением
def add(a, b):
    return a + b

result = add(3, 5)  # 8

# Множественный возврат (tuple)
def get_user():
    return "Alice", 30, "NYC"

name, age, city = get_user()

# Значения по умолчанию
def greet_user(name, greeting="Hello"):
    print(f"{greeting}, {name}!")

greet_user("Alice")              # Hello, Alice!
greet_user("Bob", "Hi")          # Hi, Bob!

# ВАЖНО: не используйте изменяемые объекты как значения по умолчанию
# ❌ Плохо
def add_item(item, items=[]):
    items.append(item)
    return items

# ✅ Хорошо
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### Типы аргументов

```python
# Позиционные аргументы
def power(base, exponent):
    return base ** exponent

power(2, 3)  # 8

# Именованные аргументы (keyword arguments)
power(base=2, exponent=3)
power(exponent=3, base=2)  # порядок не важен

# *args - произвольное количество позиционных аргументов
def sum_all(*numbers):
    return sum(numbers)

sum_all(1, 2, 3, 4, 5)  # 15

# **kwargs - произвольное количество именованных аргументов
def print_info(**info):
    for key, value in info.items():
        print(f"{key}: {value}")

print_info(name="Alice", age=30, city="NYC")

# Комбинация всех типов (порядок важен!)
def complex_function(pos_arg, *args, keyword_arg="default", **kwargs):
    print(f"Позиционный: {pos_arg}")
    print(f"*args: {args}")
    print(f"Именованный: {keyword_arg}")
    print(f"**kwargs: {kwargs}")

complex_function(1, 2, 3, keyword_arg="test", extra1="a", extra2="b")

# Только позиционные аргументы (Python 3.8+)
def divide(a, b, /):  # / означает только позиционные
    return a / b

divide(10, 2)        # ✅ OK
divide(a=10, b=2)    # ❌ Error

# Только именованные аргументы
def create_user(*, name, age):  # * означает только именованные
    return {"name": name, "age": age}

create_user(name="Alice", age=30)  # ✅ OK
create_user("Alice", 30)           # ❌ Error
```

### Type hints для функций

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add_numbers(a: int, b: int) -> int:
    return a + b

def process_data(data: list[dict[str, str]]) -> None:
    for item in data:
        print(item)

# Optional и Union
from typing import Optional, Union

def find_user(user_id: int) -> Optional[dict]:  # может вернуть None
    # ...
    return None

def process_value(value: Union[int, str]) -> str:  # int или str
    return str(value)

# Python 3.10+ синтаксис
def process_value(value: int | str) -> str:
    return str(value)
```

### Лямбда-функции

```python
# Анонимная функция
square = lambda x: x ** 2
print(square(5))  # 25

# Лямбда с несколькими аргументами
add = lambda x, y: x + y
print(add(3, 5))  # 8

# Использование с map, filter, sorted
numbers = [1, 2, 3, 4, 5]

squared = list(map(lambda x: x ** 2, numbers))
# [1, 4, 9, 16, 25]

evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4]

users = [{"name": "Bob", "age": 30}, {"name": "Alice", "age": 25}]
sorted_users = sorted(users, key=lambda u: u["age"])
```

### Области видимости (Scope)

```python
# LEGB Rule: Local, Enclosing, Global, Built-in

global_var = "I'm global"

def outer():
    enclosing_var = "I'm enclosing"
    
    def inner():
        local_var = "I'm local"
        print(local_var)       # Local
        print(enclosing_var)   # Enclosing
        print(global_var)      # Global
        print(len([]))         # Built-in (функция len)
    
    inner()

# global - изменение глобальной переменной
counter = 0

def increment():
    global counter
    counter += 1

# nonlocal - изменение переменной из внешней функции
def outer():
    count = 0
    
    def inner():
        nonlocal count
        count += 1
        return count
    
    return inner
```

---

## Классы и ООП

### Основы классов

```python
# Определение класса
class Dog:
    # Атрибут класса (общий для всех экземпляров)
    species = "Canis familiaris"
    
    # Конструктор
    def __init__(self, name, age):
        # Атрибуты экземпляра
        self.name = name
        self.age = age
    
    # Метод экземпляра
    def bark(self):
        return f"{self.name} says Woof!"
    
    # Метод с параметром
    def info(self):
        return f"{self.name} is {self.age} years old"
    
    # Специальный метод (magic method)
    def __str__(self):
        return f"Dog({self.name}, {self.age})"

# Создание экземпляра
my_dog = Dog("Buddy", 3)

# Доступ к атрибутам
print(my_dog.name)      # Buddy
print(my_dog.species)   # Canis familiaris

# Вызов методов
print(my_dog.bark())    # Buddy says Woof!
print(my_dog)           # Dog(Buddy, 3)
```

### Наследование

```python
# Базовый класс
class Animal:
    def __init__(self, name):
        self.name = name
    
    def speak(self):
        raise NotImplementedError("Subclass must implement this method")

# Наследование
class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof!"

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow!"

# Использование
dog = Dog("Buddy")
cat = Cat("Whiskers")
print(dog.speak())  # Buddy says Woof!
print(cat.speak())  # Whiskers says Meow!

# super() для вызова методов родителя
class Vehicle:
    def __init__(self, brand):
        self.brand = brand
    
    def info(self):
        return f"Vehicle: {self.brand}"

class Car(Vehicle):
    def __init__(self, brand, model):
        super().__init__(brand)  # вызов конструктора родителя
        self.model = model
    
    def info(self):
        parent_info = super().info()
        return f"{parent_info}, Model: {self.model}"

# Множественное наследование
class Flyable:
    def fly(self):
        return "Flying..."

class Swimmable:
    def swim(self):
        return "Swimming..."

class Duck(Animal, Flyable, Swimmable):
    def speak(self):
        return f"{self.name} says Quack!"

duck = Duck("Donald")
print(duck.speak())  # Donald says Quack!
print(duck.fly())    # Flying...
print(duck.swim())   # Swimming...
```

### Инкапсуляция и свойства

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        self.owner = owner
        self._balance = balance  # protected (по соглашению)
        self.__pin = "1234"      # private (name mangling)
    
    # Геттер
    @property
    def balance(self):
        return self._balance
    
    # Сеттер
    @balance.setter
    def balance(self, amount):
        if amount < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = amount
    
    # Метод
    def deposit(self, amount):
        if amount > 0:
            self._balance += amount
    
    def withdraw(self, amount):
        if 0 < amount <= self._balance:
            self._balance -= amount
            return True
        return False

# Использование
account = BankAccount("Alice", 1000)
print(account.balance)      # 1000 (через геттер)
account.deposit(500)
account.balance = 2000      # через сеттер
# account.balance = -100    # ValueError
```

### Специальные методы (Magic methods)

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    # Строковое представление для пользователя
    def __str__(self):
        return f"Point({self.x}, {self.y})"
    
    # Строковое представление для разработчика
    def __repr__(self):
        return f"Point(x={self.x}, y={self.y})"
    
    # Сложение
    def __add__(self, other):
        return Point(self.x + other.x, self.y + other.y)
    
    # Вычитание
    def __sub__(self, other):
        return Point(self.x - other.x, self.y - other.y)
    
    # Сравнение
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
    
    # Длина/размер
    def __len__(self):
        return int((self.x**2 + self.y**2) ** 0.5)
    
    # Доступ по индексу
    def __getitem__(self, index):
        if index == 0:
            return self.x
        elif index == 1:
            return self.y
        raise IndexError("Index out of range")
    
    # Вызов как функция
    def __call__(self, scale):
        return Point(self.x * scale, self.y * scale)

# Использование
p1 = Point(1, 2)
p2 = Point(3, 4)

print(p1)           # Point(1, 2)
print(p1 + p2)      # Point(4, 6)
print(p1 == p2)     # False
print(p1[0])        # 1
scaled = p1(2)      # Point(2, 4)
```

### Методы класса и статические методы

```python
class DateUtils:
    # Атрибут класса
    date_format = "%Y-%m-%d"
    
    def __init__(self, date_string):
        self.date_string = date_string
    
    # Обычный метод (требует экземпляр)
    def parse(self):
        from datetime import datetime
        return datetime.strptime(self.date_string, self.date_format)
    
    # Метод класса (получает класс как первый аргумент)
    @classmethod
    def from_timestamp(cls, timestamp):
        from datetime import datetime
        date_str = datetime.fromtimestamp(timestamp).strftime(cls.date_format)
        return cls(date_str)
    
    # Статический метод (не получает ни self, ни cls)
    @staticmethod
    def is_valid_format(date_string):
        from datetime import datetime
        try:
            datetime.strptime(date_string, DateUtils.date_format)
            return True
        except ValueError:
            return False

# Использование
date = DateUtils("2025-11-15")
parsed = date.parse()

# Альтернативный конструктор
date_from_ts = DateUtils.from_timestamp(1700000000)

# Статический метод
is_valid = DateUtils.is_valid_format("2025-11-15")  # True
```

### Абстрактные классы

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass
    
    @abstractmethod
    def perimeter(self):
        pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    
    def area(self):
        return self.width * self.height
    
    def perimeter(self):
        return 2 * (self.width + self.height)

# shape = Shape()  # ❌ Error: нельзя создать экземпляр абстрактного класса
rectangle = Rectangle(5, 3)  # ✅ OK
```

### Dataclasses (Python 3.7+)

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    email: str = ""  # значение по умолчанию
    is_active: bool = True
    tags: list[str] = field(default_factory=list)  # для изменяемых объектов
    
    def __post_init__(self):
        # Вызывается после __init__
        if self.age < 0:
            raise ValueError("Age cannot be negative")

# Автоматически создаются __init__, __repr__, __eq__
user = User(name="Alice", age=30)
print(user)  # User(name='Alice', age=30, email='', is_active=True, tags=[])

# Замороженный dataclass (immutable)
@dataclass(frozen=True)
class Point:
    x: float
    y: float

point = Point(1.0, 2.0)
# point.x = 3.0  # ❌ Error: dataclass is frozen
```

---

## Работа со строками

```python
# Создание строк
s = "Hello, World!"
s2 = 'Single quotes'
s3 = """Multi
line
string"""

# Конкатенация
greeting = "Hello" + " " + "World"
repeated = "Ha" * 3  # "HaHaHa"

# Форматирование
dt_str = now.strftime("%Y-%m-%d %H:%M:%S")  # "2025-11-15 14:30:00"
date_str = now.strftime("%d/%m/%Y")          # "15/11/2025"
time_str = now.strftime("%H:%M")             # "14:30"

# Парсинг строки в datetime
dt = datetime.strptime("2025-11-15", "%Y-%m-%d")
dt = datetime.strptime("15/11/2025 14:30", "%d/%m/%Y %H:%M")

# Компоненты datetime
now.year                 # 2025
now.month                # 11
now.day                  # 15
now.hour                 # 14
now.minute               # 30
now.second               # 0
now.weekday()            # 0-6 (понедельник-воскресенье)
now.isoweekday()         # 1-7 (понедельник-воскресенье)

# Арифметика с датами
tomorrow = now + timedelta(days=1)
yesterday = now - timedelta(days=1)
week_ago = now - timedelta(weeks=1)
hour_later = now + timedelta(hours=1)

# Разница между датами
diff = datetime(2025, 12, 31) - datetime(2025, 1, 1)
diff.days                # количество дней
diff.total_seconds()     # общее количество секунд

# Unix timestamp
timestamp = now.timestamp()              # в секундах с 1970-01-01
dt_from_timestamp = datetime.fromtimestamp(timestamp)

# Сравнение дат
dt1 = datetime(2025, 1, 1)
dt2 = datetime(2025, 12, 31)
dt1 < dt2                # True
dt1 == dt2               # False

# Форматы strftime/strptime:
# %Y - год (4 цифры)       2025
# %y - год (2 цифры)       25
# %m - месяц               11
# %d - день                15
# %H - час (24ч)           14
# %I - час (12ч)           02
# %M - минута              30
# %S - секунда             00
# %p - AM/PM               PM
# %A - день недели         Saturday
# %a - день недели кратко  Sat
# %B - месяц               November
# %b - месяц кратко        Nov
# %w - день недели (0-6)   6
# %j - день года (1-366)   319

# Таймзоны (требуется библиотека pytz или zoneinfo)
from zoneinfo import ZoneInfo  # Python 3.9+

utc_time = datetime.now(ZoneInfo("UTC"))
ny_time = datetime.now(ZoneInfo("America/New_York"))
tokyo_time = datetime.now(ZoneInfo("Asia/Tokyo"))

# Конвертация между таймзонами
utc_dt = datetime.now(ZoneInfo("UTC"))
ny_dt = utc_dt.astimezone(ZoneInfo("America/New_York"))

# Измерение времени выполнения
import time

start = time.time()
# какой-то код
time.sleep(1)  # пауза на 1 секунду
end = time.time()
elapsed = end - start
print(f"Время выполнения: {elapsed:.4f} сек")

# Более точное измерение
start = time.perf_counter()
# код
end = time.perf_counter()
```

---

## Регулярные выражения

```python
import re

# Основные функции
text = "The quick brown fox jumps over the lazy dog"

# Поиск первого совпадения
match = re.search(r"quick", text)
if match:
    print(match.group())      # "quick"
    print(match.start())      # 4 (начальный индекс)
    print(match.end())        # 9 (конечный индекс)

# Поиск в начале строки
match = re.match(r"The", text)   # найдет только если в начале

# Поиск всех совпадений
matches = re.findall(r"\b\w{4}\b", text)  # все 4-буквенные слова
# ['quick', 'brown', 'jumps', 'over', 'lazy']

# Итератор по совпадениям
for match in re.finditer(r"\b\w+\b", text):
    print(match.group(), match.start())

# Замена
new_text = re.sub(r"fox", "cat", text)
new_text = re.sub(r"\d+", "NUM", "abc 123 def 456")  # заменить числа

# Разделение строки
parts = re.split(r"\s+", text)       # по пробелам
parts = re.split(r"[,;]", "a,b;c")   # по запятой или точке с запятой

# Основные паттерны регулярных выражений:

# Символы
# .     - любой символ (кроме новой строки)
# \d    - цифра [0-9]
# \D    - не цифра
# \w    - буква, цифра или _ [a-zA-Z0-9_]
# \W    - не \w
# \s    - пробельный символ [ \t\n\r\f\v]
# \S    - не пробельный символ
# ^     - начало строки
# $     - конец строки
# \b    - граница слова
# \B    - не граница слова

# Квантификаторы
# *     - 0 или более раз
# +     - 1 или более раз
# ?     - 0 или 1 раз
# {n}   - ровно n раз
# {n,}  - n или более раз
# {n,m} - от n до m раз

# Группы и классы символов
# [abc]     - a, b или c
# [a-z]     - любая строчная буква
# [^abc]    - любой символ кроме a, b, c
# (abc)     - группа
# (?:abc)   - не захватывающая группа
# a|b       - a или b

# Примеры паттернов:

# Email
email_pattern = r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}"
emails = re.findall(email_pattern, text)

# Телефон
phone_pattern = r"\+?\d{1,3}?[-.\s]?\(?\d{1,4}\)?[-.\s]?\d{1,4}[-.\s]?\d{1,9}"
phones = re.findall(phone_pattern, text)

# URL
url_pattern = r"https?://[^\s]+"
urls = re.findall(url_pattern, text)

# IP адрес
ip_pattern = r"\b(?:\d{1,3}\.){3}\d{1,3}\b"
ips = re.findall(ip_pattern, text)

# Дата (DD/MM/YYYY или DD-MM-YYYY)
date_pattern = r"\b\d{2}[/-]\d{2}[/-]\d{4}\b"
dates = re.findall(date_pattern, text)

# Группы захвата
pattern = r"(\w+)@(\w+)\.(\w+)"
match = re.search(pattern, "user@example.com")
if match:
    print(match.group(0))  # "user@example.com" (всё совпадение)
    print(match.group(1))  # "user"
    print(match.group(2))  # "example"
    print(match.group(3))  # "com"
    print(match.groups())  # ("user", "example", "com")

# Именованные группы
pattern = r"(?P<user>\w+)@(?P<domain>\w+)\.(?P<tld>\w+)"
match = re.search(pattern, "user@example.com")
if match:
    print(match.group("user"))    # "user"
    print(match.group("domain"))  # "example"
    print(match.groupdict())      # {'user': 'user', 'domain': 'example', 'tld': 'com'}

# Компиляция паттерна для многократного использования
pattern = re.compile(r"\d+")
matches = pattern.findall("abc 123 def 456")
match = pattern.search("abc 123")

# Флаги
# re.IGNORECASE или re.I  - игнорировать регистр
# re.MULTILINE или re.M   - ^ и $ для каждой строки
# re.DOTALL или re.S      - . включает новую строку
# re.VERBOSE или re.X     - разрешить пробелы и комментарии

pattern = re.compile(r"hello", re.IGNORECASE)
matches = pattern.findall("Hello HELLO hello")  # ['Hello', 'HELLO', 'hello']

# Lookahead и lookbehind
# (?=...)   - positive lookahead
# (?!...)   - negative lookahead
# (?<=...)  - positive lookbehind
# (?<!...)  - negative lookbehind

# Найти слова, за которыми следует запятая
pattern = r"\w+(?=,)"
matches = re.findall(pattern, "apple, banana, cherry")  # ['apple', 'banana']

# Найти числа, перед которыми НЕТ буквы
pattern = r"(?<![a-zA-Z])\d+"
matches = re.findall(pattern, "abc123 456 def789")  # ['456']
```

---

## Асинхронное программирование

```python
import asyncio

# Базовая асинхронная функция
async def greet(name):
    print(f"Hello, {name}!")
    await asyncio.sleep(1)  # асинхронная пауза
    print(f"Goodbye, {name}!")

# Запуск асинхронной функции
asyncio.run(greet("Alice"))

# Множественные задачи
async def task1():
    print("Task 1 starting")
    await asyncio.sleep(2)
    print("Task 1 completed")
    return "Result 1"

async def task2():
    print("Task 2 starting")
    await asyncio.sleep(1)
    print("Task 2 completed")
    return "Result 2"

async def main():
    # Запуск параллельно
    result1, result2 = await asyncio.gather(task1(), task2())
    print(f"Results: {result1}, {result2}")

asyncio.run(main())

# Создание задач
async def main():
    # Создаем задачи
    task_1 = asyncio.create_task(task1())
    task_2 = asyncio.create_task(task2())
    
    # Ждем завершения
    await task_1
    await task_2

# Таймауты
async def long_operation():
    await asyncio.sleep(10)
    return "Done"

async def main():
    try:
        result = await asyncio.wait_for(long_operation(), timeout=2.0)
    except asyncio.TimeoutError:
        print("Операция заняла слишком много времени")

# Асинхронный контекстный менеджер
class AsyncResource:
    async def __aenter__(self):
        print("Acquiring resource")
        await asyncio.sleep(0.1)
        return self
    
    async def __aexit__(self, exc_type, exc, tb):
        print("Releasing resource")
        await asyncio.sleep(0.1)

async def main():
    async with AsyncResource() as resource:
        print("Using resource")

# Асинхронный итератор
class AsyncCounter:
    def __init__(self, stop):
        self.current = 0
        self.stop = stop
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.current < self.stop:
            await asyncio.sleep(0.1)
            self.current += 1
            return self.current
        raise StopAsyncIteration

async def main():
    async for number in AsyncCounter(5):
        print(number)

# Асинхронный генератор
async def async_range(count):
    for i in range(count):
        await asyncio.sleep(0.1)
        yield i

async def main():
    async for i in async_range(5):
        print(i)

# Работа с HTTP (требуется aiohttp)
# import aiohttp
# 
# async def fetch_url(url):
#     async with aiohttp.ClientSession() as session:
#         async with session.get(url) as response:
#             return await response.text()
# 
# async def main():
#     html = await fetch_url("https://example.com")
#     print(html)

# Запуск в фоновом режиме
async def background_task():
    while True:
        print("Background task running")
        await asyncio.sleep(5)

async def main():
    # Создать фоновую задачу
    bg_task = asyncio.create_task(background_task())
    
    # Основная работа
    await asyncio.sleep(15)
    
    # Отменить фоновую задачу
    bg_task.cancel()
    try:
        await bg_task
    except asyncio.CancelledError:
        print("Background task cancelled")

# Синхронизация
async def worker(lock, resource):
    async with lock:
        # Критическая секция
        print(f"Worker accessing resource: {resource}")
        await asyncio.sleep(1)

async def main():
    lock = asyncio.Lock()
    await asyncio.gather(
        worker(lock, "data"),
        worker(lock, "data"),
        worker(lock, "data")
    )

# Семафор (ограничение количества одновременных операций)
async def limited_operation(semaphore, name):
    async with semaphore:
        print(f"{name} started")
        await asyncio.sleep(1)
        print(f"{name} finished")

async def main():
    semaphore = asyncio.Semaphore(2)  # максимум 2 одновременно
    await asyncio.gather(
        limited_operation(semaphore, "Op1"),
        limited_operation(semaphore, "Op2"),
        limited_operation(semaphore, "Op3"),
        limited_operation(semaphore, "Op4")
    )
```

---

## Лучшие практики

### PEP 8 - Стиль кода

```python
# Отступы: 4 пробела (не табуляция)
def my_function():
    if True:
        print("4 пробела")

# Длина строки: максимум 79 символов для кода, 72 для комментариев
# Перенос длинных строк:
long_variable_name = (
    some_function(arg1, arg2) +
    another_function(arg3, arg4)
)

# Пустые строки:
# - 2 пустые строки между функциями/классами верхнего уровня
# - 1 пустая строка между методами класса

# Импорты:
# 1. Стандартная библиотека
import os
import sys

# 2. Сторонние библиотеки
import numpy as np
import pandas as pd

# 3. Локальные модули
from myapp import models
from myapp.utils import helpers

# Избегайте:
# from module import *  # не рекомендуется

# Пробелы вокруг операторов
x = 1
y = 2
result = x + y

# Без пробелов в вызовах функций и индексации
func(arg1, arg2)
my_list[0]
my_dict['key']

# Комментарии на той же строке - 2 пробела перед #
x = x + 1  # Увеличить x

# Docstrings
def my_function(arg1, arg2):
    """
    Краткое описание функции.
    
    Подробное описание того, что делает функция.
    
    Args:
        arg1 (int): Описание первого аргумента.
        arg2 (str): Описание второго аргумента.
    
    Returns:
        bool: Описание возвращаемого значения.
    
    Raises:
        ValueError: Описание исключения.
    """
    return True
```

### Pythonic код

```python
# ✅ Хорошо: использовать enumerate
for i, item in enumerate(items):
    print(f"{i}: {item}")

# ❌ Плохо
for i in range(len(items)):
    print(f"{i}: {items[i]}")

# ✅ Хорошо: использовать zip
for name, age in zip(names, ages):
    print(f"{name} is {age}")

# ❌ Плохо
for i in range(len(names)):
    print(f"{names[i]} is {ages[i]}")

# ✅ Хорошо: list comprehension
squares = [x**2 for x in range(10)]

# ❌ Плохо
squares = []
for x in range(10):
    squares.append(x**2)

# ✅ Хорошо: использовать with для файлов
with open("file.txt") as f:
    content = f.read()

# ❌ Плохо
f = open("file.txt")
content = f.read()
f.close()

# ✅ Хорошо: использовать get для словарей
value = my_dict.get("key", default_value)

# ❌ Плохо
if "key" in my_dict:
    value = my_dict["key"]
else:
    value = default_value

# ✅ Хорошо: проверка на None
if x is None:
    pass

# ❌ Плохо
if x == None:
    pass

# ✅ Хорошо: проверка пустой последовательности
if not my_list:
    pass

# ❌ Плохо
if len(my_list) == 0:
    pass

# ✅ Хорошо: использовать any/all
if any(x > 10 for x in numbers):
    pass

# ❌ Плохо
found = False
for x in numbers:
    if x > 10:
        found = True
        break
```

### Обработка ошибок

```python
# ✅ Хорошо: конкретные исключения
try:
    value = int(user_input)
except ValueError:
    print("Некорректный ввод")

# ❌ Плохо: перехват всех исключений
try:
    value = int(user_input)
except:
    print("Ошибка")

# ✅ Хорошо: использовать EAFP (Easier to Ask for Forgiveness than Permission)
try:
    value = my_dict["key"]
except KeyError:
    value = default_value

# ❌ Плохо: LBYL (Look Before You Leap)
if "key" in my_dict:
    value = my_dict["key"]
else:
    value = default_value
```

### Производительность

```python
# ✅ Хорошо: генераторы для больших данных
sum_of_squares = sum(x**2 for x in range(1000000))

# ❌ Плохо: создание списка в памяти
sum_of_squares = sum([x**2 for x in range(1000000)])

# ✅ Хорошо: использовать set для проверки принадлежности
valid_items = {"apple", "banana", "cherry"}
if item in valid_items:  # O(1)
    pass

# ❌ Плохо: использовать список
valid_items = ["apple", "banana", "cherry"]
if item in valid_items:  # O(n)
    pass

# ✅ Хорошо: join для конкатенации строк
result = " ".join(words)

# ❌ Плохо: использование +
result = ""
for word in words:
    result += word + " "

# Профилирование кода
import cProfile
import pstats

def my_function():
    # код для профилирования
    pass

cProfile.run('my_function()', 'output.stats')
stats = pstats.Stats('output.stats')
stats.sort_stats('cumulative').print_stats(10)
```

### Типизация (Type Hints)

```python
from typing import List, Dict, Optional, Union, Tuple, Callable

# Базовые типы
def greet(name: str) -> str:
    return f"Hello, {name}!"

# Коллекции
def process_items(items: List[int]) -> Dict[str, int]:
    return {"total": sum(items)}

# Optional (может быть None)
def find_user(user_id: int) -> Optional[Dict[str, str]]:
    return None

# Union (несколько возможных типов)
def process_value(value: Union[int, str]) -> str:
    return str(value)

# Python 3.10+ синтаксис
def process_value(value: int | str) -> str:
    return str(value)

# Callable (функция как аргумент)
def apply_function(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# Generics
from typing import TypeVar, Generic

T = TypeVar('T')

class Stack(Generic[T]):
    def __init__(self) -> None:
        self.items: List[T] = []
    
    def push(self, item: T) -> None:
        self.items.append(item)
    
    def pop(self) -> T:
        return self.items.pop()

# Проверка типов с помощью mypy:
# pip install mypy
# mypy your_script.py
```

### Тестирование

```python
import unittest

# Базовый тест
class TestMathOperations(unittest.TestCase):
    def test_addition(self):
        self.assertEqual(2 + 2, 4)
    
    def test_division(self):
        self.assertEqual(10 / 2, 5)
    
    def test_division_by_zero(self):
        with self.assertRaises(ZeroDivisionError):
            result = 1 / 0

if __name__ == '__main__':
    unittest.main()

# pytest (более современный)
# pip install pytest

def test_addition():
    assert 2 + 2 == 4

def test_division():
    assert 10 / 2 == 5

def test_division_by_zero():
    with pytest.raises(ZeroDivisionError):
        result = 1 / 0

# Запуск: pytest test_file.py
```

---

## Полезные библиотеки

```python
# requests - HTTP запросы
import requests
response = requests.get("https://api.example.com/data")
data = response.json()

# pandas - анализ данных
import pandas as pd
df = pd.read_csv("data.csv")
print(df.head())

# numpy - численные вычисления
import numpy as np
arr = np.array([1, 2, 3, 4, 5])
print(arr.mean())

# matplotlib - визуализация
import matplotlib.pyplot as plt
plt.plot([1, 2, 3, 4], [1, 4, 9, 16])
plt.show()

# sqlalchemy - работа с БД
from sqlalchemy import create_engine
engine = create_engine("sqlite:///database.db")

# flask - веб-фреймворк
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"

# fastapi - современный веб-фреймворк
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}
```

---

## Быстрая справка

```python
# Вывод
print("Hello")
print(f"x = {x}")

# Ввод
name = input("Введите имя: ")
num = int(input("Введите число: "))

# Условия
if x > 0:
    print("Положительное")
elif x < 0:
    print("Отрицательное")
else:
    print("Ноль")

# Циклы
for i in range(5):
    print(i)

while x < 10:
    x += 1

# Функция
def greet(name="World"):
    return f"Hello, {name}!"

# Класс
class Person:
    def __init__(self, name):
        self.name = name
    
    def greet(self):
        return f"Hello, I'm {self.name}"

# Работа с файлами
with open("file.txt", "r") as f:
    content = f.read()

# List comprehension
squares = [x**2 for x in range(10)]

# Словарь
user = {"name": "Alice", "age": 30}
print(user["name"])
print(user.get("email", "не указан"))

# Try-except
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Ошибка деления на ноль")

# Lambda
square = lambda x: x**2

# Map, filter
numbers = [1, 2, 3, 4, 5]
squared = list(map(lambda x: x**2, numbers))
evens = list(filter(lambda x: x % 2 == 0, numbers))
``` строк
name = "Alice"
age = 30

# f-strings (рекомендуется, Python 3.6+)
msg = f"My name is {name} and I'm {age} years old"
msg = f"{name.upper()} is {age * 2} in dog years"
msg = f"Price: {price:.2f}"  # 2 знака после запятой

# format()
msg = "My name is {} and I'm {}".format(name, age)
msg = "My name is {0} and I'm {1}".format(name, age)
msg = "My name is {n} and I'm {a}".format(n=name, a=age)

# % форматирование (старый стиль)
msg = "My name is %s and I'm %d" % (name, age)

# Методы строк
text = "  Hello, World!  "

# Регистр
text.upper()                # "  HELLO, WORLD!  "
text.lower()                # "  hello, world!  "
text.capitalize()           # "  hello, world!  "
text.title()                # "  Hello, World!  "
text.swapcase()             # "  hELLO, wORLD!  "

# Очистка
text.strip()                # "Hello, World!" (убрать пробелы с краёв)
text.lstrip()               # "Hello, World!  " (слева)
text.rstrip()               # "  Hello, World!" (справа)
text.strip("! ")            # "Hello, World" (убрать указанные символы)

# Поиск и проверка
text.startswith("  Hello")  # True
text.endswith("!")          # True
text.find("World")          # 9 (индекс начала, -1 если не найдено)
text.index("World")         # 9 (индекс начала, ValueError если не найдено)
text.count("o")             # 2
"123".isdigit()             # True
"abc".isalpha()             # True
"abc123".isalnum()          # True
" ".isspace()               # True

# Замена
text.replace("World", "Python")  # "  Hello, Python!  "
text.replace("l", "L", 1)        # заменить только первое вхождение

# Разделение и объединение
words = "apple,banana,cherry".split(",")  # ['apple', 'banana', 'cherry']
words = "hello world".split()             # ['hello', 'world'] (по пробелам)
path = "/usr/local/bin".split("/")        # ['', 'usr', 'local', 'bin']

result = " ".join(["Hello", "World"])     # "Hello World"
result = "-".join(["2025", "11", "15"])   # "2025-11-15"

# Разделение на строки
lines = "Line 1\nLine 2\nLine 3".splitlines()  # ['Line 1', 'Line 2', 'Line 3']

# Выравнивание
"test".center(10)           # "   test   "
"test".ljust(10)            # "test      "
"test".rjust(10)            # "      test"
"test".center(10, "-")      # "---test---"

# Индексация и срезы
s = "Python"
s[0]                        # "P"
s[-1]                       # "n"
s[1:4]                      # "yth"
s[:3]                       # "Pyt"
s[3:]                       # "hon"
s[::2]                      # "Pto" (каждый второй символ)
s[::-1]                     # "nohtyP" (разворот строки)

# Проверка подстроки
"Py" in "Python"            # True
"Java" not in "Python"      # True

# Escape-последовательности
newline = "Line 1\nLine 2"  # перенос строки
tab = "Col1\tCol2"          # табуляция
quote = "He said \"Hi\""    # кавычки
backslash = "C:\\Users"     # обратный слэш
raw = r"C:\Users\name"      # сырая строка (игнорирует escape)