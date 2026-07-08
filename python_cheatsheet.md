# Python Cheatsheet

> Targets **CPython 3.9+**; newer-version features are gated inline (e.g. `3.10+`). Current stable is 3.13/3.14 — everything here is forward-compatible.

---

## Numeric Types & Operators

```python
# Arithmetic
3 + 2   # 5       addition
3 - 2   # 1       subtraction
3 * 2   # 6       multiplication
3 / 2   # 1.5     division (always float)
3 // 2  # 1       floor division
3 % 2   # 1       modulo (remainder)
3 ** 2  # 9       exponentiation

# Augmented assignment
a += 5    # a = a + 5
a -= 1    # a = a - 1
a *= 2    # a = a * 2
a /= 2    # a = a / 2
a **= 2   # a = a ** 2
a %= 3    # a = a % 3

# Built-ins
abs(-3.5)              # 3.5
round(3.14159, 2)      # 3.14
min(3, 1, 4)           # 1
max(3, 1, 4)           # 4
sum([1, 2, 3])         # 6
type(42)               # <class 'int'>

# Type conversion
int('42')              # 42
float('3.14')          # 3.14
str(100)               # '100'
bool(0)                # False
```

---

## Strings

```python
s = 'hello world'

# Indexing & slicing
s[0]       # 'h'
s[-1]      # 'd'
s[1:5]     # 'ello'
s[::2]     # 'hlowrd'   (step)
s[::-1]    # reverse

# Common methods
s.upper()               # 'HELLO WORLD'
s.lower()               # 'hello world'
s.capitalize()          # 'Hello world'
s.strip()               # remove leading/trailing whitespace
s.lstrip()              # remove leading only
s.rstrip()              # remove trailing only
s.replace('l', 'r')     # 'herro worrd'
s.split()               # ['hello', 'world']
s.split(',')            # split by delimiter
s.find('world')         # 6  (index of first match, -1 if not found)
s.count('l')            # 3  (count non-overlapping occurrences)

from collections import Counter
Counter(s)              # Counter({'l': 3, 'o': 2, ' ': 1, 'h': 1, 'e': 1, 'w': 1, 'r': 1, 'd': 1})
Counter(s).most_common(3)  # [('l', 3), ('o', 2), (' ', 1)]  — top 3 most frequent chars
s.startswith('hello')   # True
s.endswith('world')     # True
s.isdigit()             # False
s.isalpha()             # False  (has space)
' '.join(['a', 'b'])    # 'a b'  (join list into string)
len(s)                  # 11

# Formatting
name = 'Ada'
f'Hello {name}'                     # f-string (preferred)
'Hello {}'.format(name)             # .format()
'Hello %s' % name                   # % formatting

# Alignment & padding (width-based)
'hi'.ljust(10, '-')     # 'hi--------'   left-justify in width 10, pad with '-'
'hi'.rjust(10, '-')     # '--------hi'   right-justify
'hi'.center(10, '-')    # '----hi----'   center
'hi'.center(11, '*')    # '****hi*****'  extra pad goes to the right
'42'.zfill(5)           # '00042'        zero-pad numbers (keeps leading sign)
```

---

## Conditionals

```python
# Comparison operators
==  !=  <  >  <=  >=

# Logical operators
and  or  not

# Membership operators
x in [1, 2, 3]        # True
x not in [1, 2, 3]    # False

# If-elif-else
if x < 0:
    print('negative')
elif x > 0:
    print('positive')
else:
    print('zero')

# Ternary
label = 'even' if x % 2 == 0 else 'odd'
```

---

## Loops

```python
# While
while x <= 10:
    x += 1

# For with range
for i in range(10):        # 0–9
for i in range(1, 10):     # 1–9
for i in range(0, 10, 2):  # 0,2,4,6,8 (step)

# For over iterable
for char in 'hello':
for item in my_list:
for k, v in my_dict.items():

# Enumerate
for idx, val in enumerate(my_list):
    print(idx, val)

# Zip — iterate two iterables in parallel
for a, b in zip(list1, list2):
    print(a, b)

# Control flow
break       # exit loop
continue    # skip to next iteration
pass        # do nothing (placeholder)
```

---

## Lists

```python
lst = [1, 'hello', 3.0, True]

# Indexing & slicing
lst[0]      lst[-1]     lst[1:3]    lst[::2]

# Mutation methods
lst.append(x)           # add to end
lst.extend([4, 5])      # add multiple
lst.insert(0, 'x')      # insert at index
lst.remove(x)           # remove first occurrence
lst.pop()               # remove & return last element
lst.pop(i)              # remove & return element at index i
lst.sort()              # sort in-place (mutates)
lst.sort(reverse=True)  # descending in-place
sorted(lst)             # returns NEW sorted list (non-mutating)
sorted(lst, key=len, reverse=True)
lst.reverse()           # reverse in-place
lst.copy()              # shallow copy

# Info methods
lst.count(x)            # count occurrences
lst.index(x)            # index of first occurrence
len(lst)                # length

# Constructors
list(range(5))          # [0, 1, 2, 3, 4]
list('abc')             # ['a', 'b', 'c']
```

---

## Tuples

```python
t = (1, 2, 3)
t = 1, 2, 3             # implicit tuple
t = (1,)                # single-element tuple — trailing comma required
t = ()                  # empty tuple

# Access (no mutation)
t[0]        t[-1]       t[1:3]
t.count(1)  t.index(2)

# Unpacking
a, b, c = t
a, b = b, a             # swap values
```

> Tuples are **immutable** — no append, remove, or reassignment.

---

## Sets

```python
s = {1, 2, 3}
s = set([1, 2, 2, 3])   # deduplicates → {1, 2, 3}
s = set()               # empty set (NOT {})

# Methods
s.add(4)
s.update({5, 6})
s.remove(x)             # KeyError if missing
s.discard(x)            # silent if missing
s.pop()                 # remove arbitrary element
s.clear()

# Set operations
a.union(b)              # a | b
a.intersection(b)       # a & b
a.difference(b)         # a - b
a.symmetric_difference(b)  # a ^ b
a.issubset(b)
a.issuperset(b)
a.isdisjoint(b)

# Membership — O(1)
x in s
```

---

## Dictionaries

```python
d = {'name': 'Ada', 'age': 30}

# Access
d['name']                           # KeyError if missing
d.get('name', 'default')           # safe access

# Modification
d['age'] = 31                       # update
d['email'] = 'a@b.com'             # add new key
d.pop('age')                        # remove & return
d.update({'city': 'Berlin'})        # merge / add key-value pairs
d.setdefault('score', 0)           # set key only if it doesn't exist

# Merge dicts — two ways:
merged = {**d1, **d2}              # Python 3.5+  — d2 values win on duplicate keys
merged = d1 | d2                   # Python 3.9+  — cleaner syntax, same behaviour

# Iteration
for k in d:                         # keys
for v in d.values():                # values
for k, v in d.items():              # key-value pairs

# Methods
d.keys()   d.values()   d.items()

# Build from two lists
dict(zip(['a','b'], [1, 2]))        # {'a': 1, 'b': 2}
```

---

## Comprehensions

```python
# List
[x**2 for x in range(5)]
[x for x in range(10) if x % 2 == 0]

# Dict
{x: x**2 for x in range(5)}
{k: v for k, v in d.items() if v > 0}

# Set
{x**2 for x in [1, 1, 2, 3]}

# Generator (lazy, memory-efficient)
gen = (x**2 for x in range(1000))
next(gen)
```

---

## Functions

```python
def greet(name, greeting='Hello'):
    """Docstring."""
    return f'{greeting}, {name}!'

# Calls
greet('Ada')                      # positional + default
greet('Ada', 'Hi')                # positional
greet(name='Ada', greeting='Hi')  # keyword

# Rules: defaults must come AFTER non-defaults
def f(a, b=10): ...     # valid
def f(a=10, b): ...     # SyntaxError

# *args — variable positional arguments (tuple)
def total(*args):
    return sum(args)

total(1, 2, 3)          # 6

# **kwargs — variable keyword arguments (dict)
def show(**kwargs):
    for k, v in kwargs.items():
        print(k, v)

show(name='Ada', age=30)

# Combined
def f(a, b=10, *args, **kwargs):
    pass
```

---

## Lambda

```python
square   = lambda x: x ** 2
add      = lambda x, y: x + y
classify = lambda x: 'even' if x % 2 == 0 else 'odd'
```

---

## map / filter / reduce

```python
from functools import reduce

nums = [1, 2, 3, 4, 5]

list(map(lambda x: x * 2, nums))         # [2, 4, 6, 8, 10]
list(filter(lambda x: x % 2 == 0, nums)) # [2, 4]
reduce(lambda x, y: x + y, nums)         # 15

# Chaining
result = list(map(lambda x: x**2, filter(lambda x: x % 2 == 0, nums)))
```

---

## Iterators & Generators

```python
# Iterator protocol
it = iter([1, 2, 3])
next(it)    # 1
next(it)    # 2

# Generator function
def squares(n):
    for i in range(1, n + 1):
        yield i * i         # pauses here, resumes on next()

gen = squares(5)
next(gen)   # 1
next(gen)   # 4

# Generator expression
gen = (x**2 for x in range(10))
```

> `yield` saves function state; values computed on-demand (lazy).

---

## Error Handling

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print('division by zero')
except (TypeError, ValueError) as e:
    print(f'error: {e}')
else:
    print('no exception')           # runs only if no exception raised
finally:
    print('always runs')            # cleanup

# Common built-in exceptions
# ZeroDivisionError  — division by zero
# ValueError         — right type, wrong value  e.g. int('abc')
# TypeError          — wrong type        e.g. '5' + 3
# IndexError         — index out of range
# KeyError           — dict key missing
# FileNotFoundError  — file doesn't exist
# AttributeError     — object has no attribute
# StopIteration      — iterator exhausted

# Raise custom exception
class InvalidAgeError(Exception):
    pass

raise InvalidAgeError('Age must be positive')

# Assert (dev-time checks)
assert x > 0, 'x must be positive'
```

### Logging

```python
import logging
logging.basicConfig(level=logging.DEBUG, filename='app.log',
                    format='%(asctime)s - %(levelname)s - %(message)s')

logging.debug('detailed info')
logging.info('normal operation')
logging.warning('unexpected but ok')
logging.error('something failed')
logging.critical('app may crash')
```

---

## Object-Oriented Programming

### Class Basics

```python
class Car:
    wheels = 4                          # class variable (shared)

    def __init__(self, make, speed):
        self.make = make                # instance variable
        self.speed = speed

    def accelerate(self):
        self.speed += 10

    def __str__(self):
        return f'{self.make} @ {self.speed}km/h'

    def __repr__(self):
        return f'Car({self.make!r}, {self.speed})'

    @classmethod
    def info(cls):                      # receives class, not instance
        return f'All cars have {cls.wheels} wheels'

    @staticmethod
    def general():                      # no self or cls
        return 'Cars are transportation'

my_car = Car('Toyota', 60)
my_car.accelerate()
print(my_car)
```

---

### Inheritance

```python
class Vehicle:
    def __init__(self, make):
        self.make = make

    def start(self):
        return f'{self.make} started'

class Car(Vehicle):
    def __init__(self, make, doors):
        super().__init__(make)      # call parent __init__
        self.doors = doors

    def drive(self):
        return f'{self.make} is driving'

c = Car('Tesla', 4)
c.start()   # inherited
c.drive()   # own method
```

---

### Encapsulation

```python
class BankAccount:
    def __init__(self):
        self.owner = 'Ada'          # public
        self._balance = 1000        # protected (convention)
        self.__pin = 1234           # private (name-mangled → _BankAccount__pin)

    def get_balance(self):
        return self._balance
```

---

### Polymorphism

```python
# Method overriding
class Animal:
    def speak(self): return 'generic sound'

class Dog(Animal):
    def speak(self): return 'woof'

# Duck typing
class Bird:
    def fly(self): print('bird flies')

class Plane:
    def fly(self): print('plane soars')

def launch(obj):
    obj.fly()       # works for any object with fly()

# isinstance / issubclass
isinstance(my_car, Car)         # True
isinstance(my_car, Vehicle)     # True  (checks up the chain)
issubclass(Car, Vehicle)        # True
```

---

### Abstraction (ABC)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self): pass

    @abstractmethod
    def perimeter(self): pass

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self):        return 3.14 * self.r ** 2
    def perimeter(self):   return 2 * 3.14 * self.r
```

---

### Decorators

```python
# Authorization
def admin_only(f):
    def wrap(user):
        if user != 'admin':
            return 'Access Denied'
        return f(user)
    return wrap

@admin_only
def view_data(user):
    return 'Secret data'

# Caching
def cache(f):
    store = {}
    def wrap(n):
        if n not in store:
            store[n] = f(n)
        return store[n]
    return wrap

@cache
def expensive(n):
    return n ** n

# Logging
def log(f):
    def wrap(*args, **kwargs):
        print(f'calling {f.__name__}')
        result = f(*args, **kwargs)
        print(f'returned {result}')
        return result
    return wrap
```

---

## Functional Programming

| Concept | Description | Example |
|---------|-------------|---------|
| Pure function | Same input → same output, no side effects | `def add(a,b): return a+b` |
| Immutability | Return new objects, don't modify existing | `return lst + [item]` |
| Higher-order fn | Takes/returns functions | `map`, `filter`, custom |
| Referential transparency | Expression can be replaced by its value | `x = 5; y = x + 1` |

---

## Concurrency & Parallelism

| | Threading | Multiprocessing |
|-|-----------|-----------------|
| Best for | I/O-bound tasks | CPU-bound tasks |
| Memory | Shared | Separate per process |
| GIL | Limited by GIL | Not limited |

```python
import threading
t = threading.Thread(target=func, args=(arg,))
t.start()
t.join()

from multiprocessing import Pool
with Pool(4) as p:
    results = p.map(func, iterable)
```

---

## Time Complexity (Big O)

| Notation | Name | Example |
|----------|------|---------|
| O(1) | Constant | dict/set lookup, list index |
| O(log n) | Logarithmic | binary search |
| O(n) | Linear | list search, single loop |
| O(n log n) | Linearithmic | `sorted()`, merge sort |
| O(n²) | Quadratic | nested loops |
| O(2ⁿ) | Exponential | brute-force subsets |

```python
# O(1) — dict/set access
d[key]
x in my_set

# O(n) — list search
x in my_list

# O(n²) — nested loop
for i in range(n):
    for j in range(n):
        ...
```

---

## Coding Best Practices (PEP 8)

```python
# snake_case for variables and functions
my_variable = 10
def my_function(): ...

# PascalCase for classes
class MyClass: ...

# UPPER_CASE for constants
MAX_SIZE = 100

# 4 spaces for indentation (no tabs)
# Max 79 characters per line
# Spaces around binary operators
x = i * 2 - 1
# Avoid shadowing builtins
my_list = [...]   # NOT: list = [...]
```

---

## `__name__ == "__main__"` Pattern

```python
def main():
    print('Running directly')

if __name__ == '__main__':
    main()
# Runs only when executed directly, not when imported
```

---

## File I/O

```python
# --- Reading ---
# Use one approach per open — after read() the file pointer is at the end;
# calling readlines() after it would return []
with open('file.txt', 'r') as f:
    content = f.read()              # entire file as one string

with open('file.txt', 'r') as f:
    lines = f.readlines()           # list of lines, each ending with '\n'

# Iterate line by line (memory-efficient for large files)
with open('file.txt', 'r') as f:
    for line in f:
        print(line.strip())        # .strip() removes the trailing '\n'

# --- Writing ---
with open('out.txt', 'w') as f:    # 'w' = write (overwrites existing file)
    f.write('hello\n')

with open('out.txt', 'a') as f:    # 'a' = append (keeps existing content)
    f.write('world\n')

# --- Common modes ---
# 'r'  — read text (default)
# 'w'  — write text, truncates first
# 'a'  — append text
# 'rb' / 'wb' — read/write binary (images, PDFs, etc.)

# --- CSV via csv module ---
import csv

with open('data.csv', 'r') as f:
    reader = csv.DictReader(f)      # each row becomes a dict keyed by header
    for row in reader:
        print(row['name'])

with open('out.csv', 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age'])
    writer.writeheader()
    writer.writerow({'name': 'Ada', 'age': 30})
```

---

## `collections` Module

```python
from collections import Counter, defaultdict, deque, namedtuple

# --- Counter: count occurrences ---
words = ['apple', 'banana', 'apple', 'cherry', 'banana', 'apple']
c = Counter(words)
# Counter({'apple': 3, 'banana': 2, 'cherry': 1})

c.most_common(2)        # [('apple', 3), ('banana', 2)]
c['apple']              # 3
c['mango']              # 0  (no KeyError for missing keys)

# --- defaultdict: dict that auto-creates missing keys ---
# Avoids the KeyError you'd get with a plain dict
dd = defaultdict(list)
dd['fruits'].append('apple')   # no need to check if 'fruits' key exists
dd['fruits'].append('banana')
# defaultdict(<class 'list'>, {'fruits': ['apple', 'banana']})

dd2 = defaultdict(int)
for word in words:
    dd2[word] += 1             # starts from 0 automatically

# --- deque: double-ended queue, O(1) on both ends ---
# list.insert(0, x) is O(n); deque.appendleft(x) is O(1)
dq = deque([1, 2, 3])
dq.appendleft(0)        # [0, 1, 2, 3]
dq.append(4)            # [0, 1, 2, 3, 4]
dq.popleft()            # removes 0 from the left
dq.pop()                # removes 4 from the right

dq = deque(maxlen=3)    # fixed-size; oldest item drops off when full
dq.extend([1, 2, 3, 4]) # deque([2, 3, 4])

# --- namedtuple: tuple with named fields, no class boilerplate ---
Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
p.x         # 3  — access by name
p[0]        # 3  — still accessible by index
p._asdict() # {'x': 3, 'y': 4}
```

---

## `json` Module

```python
import json

# --- Python dict → JSON string ---
data = {'name': 'Ada', 'age': 30, 'active': True}
json_str = json.dumps(data)                      # '{"name": "Ada", "age": 30, "active": true}'
json_str = json.dumps(data, indent=2)            # pretty-printed with 2-space indent

# --- JSON string → Python dict ---
parsed = json.loads('{"name": "Ada", "age": 30}')
parsed['name']   # 'Ada'

# --- Write JSON to file ---
with open('data.json', 'w') as f:
    json.dump(data, f, indent=2)   # json.dump writes to file; json.dumps returns string

# --- Read JSON from file ---
with open('data.json', 'r') as f:
    loaded = json.load(f)          # json.load reads from file; json.loads reads from string

# --- Type mapping ---
# Python  →  JSON
# dict    →  object  {}
# list    →  array   []
# str     →  string  ""
# int/float → number
# True/False → true/false
# None    →  null
```

---

## Scope — LEGB Rule

Python resolves names in this order: **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in.

```python
x = 'global'                    # G — global scope

def outer():
    x = 'enclosing'             # E — enclosing scope (for inner())

    def inner():
        x = 'local'             # L — local scope (shadows the others)
        print(x)                # 'local'

    inner()
    print(x)                    # 'enclosing'

outer()
print(x)                        # 'global'

# --- global keyword: modify a global variable from inside a function ---
count = 0

def increment():
    global count                # without this, assigning count creates a NEW local variable
    count += 1

# --- nonlocal keyword: modify an enclosing (not global) variable ---
def make_counter():
    n = 0
    def tick():
        nonlocal n              # without this, n += 1 would raise UnboundLocalError
        n += 1
        return n
    return tick

counter = make_counter()
counter()   # 1
counter()   # 2
```

---

## Type Hints

Type hints don't enforce types at runtime — they're for readability and static analysis tools like `mypy`.

```python
# --- Basic types ---
def greet(name: str) -> str:
    return f'Hello, {name}'

def add(a: int, b: int) -> int:
    return a + b

def nothing() -> None:          # function returns nothing
    print('hi')

# --- Collections (Python 3.9+ — no need to import from typing) ---
def process(items: list[int]) -> dict[str, int]:
    return {'total': sum(items)}

# --- Optional: value can be the type OR None ---
def find(name: str) -> str | None:   # Python 3.10+
    ...

from typing import Optional
def find_old(name: str) -> Optional[str]:   # pre-3.10 equivalent
    ...

# --- Union: multiple allowed types ---
def double(x: int | float) -> int | float:
    return x * 2

# --- Variables ---
age: int = 30
names: list[str] = ['Ada', 'Alan']
scores: dict[str, float] = {'Alice': 9.5}

# --- Callable: for functions passed as arguments ---
from typing import Callable
def apply(func: Callable[[int], int], value: int) -> int:
    return func(value)
```

---

## `@property` Decorator

`@property` lets you access a method like an attribute — useful for controlled read/write access.

```python
class Temperature:
    def __init__(self, celsius: float):
        self._celsius = celsius         # _prefix signals "don't touch directly"

    @property
    def celsius(self) -> float:
        return self._celsius            # called as temp.celsius, not temp.celsius()

    @celsius.setter
    def celsius(self, value: float):
        if value < -273.15:             # validate before setting
            raise ValueError('Below absolute zero')
        self._celsius = value

    @property
    def fahrenheit(self) -> float:
        return self._celsius * 9/5 + 32  # computed on the fly, looks like an attribute

temp = Temperature(25)
print(temp.celsius)       # 25      — calls the getter
print(temp.fahrenheit)    # 77.0    — computed property, no setter needed
temp.celsius = 100        # calls the setter with validation
temp.celsius = -300       # raises ValueError
```

---

## Extended Unpacking

```python
# Basic unpacking (already works with tuples/lists)
a, b, c = [1, 2, 3]

# --- Starred unpacking: capture "the rest" into a list ---
first, *rest = [1, 2, 3, 4, 5]
# first = 1, rest = [2, 3, 4, 5]

*init, last = [1, 2, 3, 4, 5]
# init = [1, 2, 3, 4], last = 5

first, *middle, last = [1, 2, 3, 4, 5]
# first = 1, middle = [2, 3, 4], last = 5

# --- Unpacking in function calls ---
nums = [1, 2, 3]
print(*nums)            # same as print(1, 2, 3)

def add(a, b, c): return a + b + c
add(*nums)              # unpacks list into positional args

params = {'a': 1, 'b': 2, 'c': 3}
add(**params)           # unpacks dict into keyword args

# --- Merging lists and dicts ---
combined = [*list1, *list2]             # concatenate without + operator
merged   = {**dict1, **dict2}          # merge; dict2 wins on duplicate keys
```

---

## `any()` and `all()`

```python
nums = [2, 4, 6, 7, 8]

# any() — True if AT LEAST ONE element is truthy
any(x % 2 != 0 for x in nums)   # True  (7 is odd)
any(x > 100 for x in nums)      # False (none exceed 100)

# all() — True if EVERY element is truthy
all(x > 0 for x in nums)        # True  (all positive)
all(x % 2 == 0 for x in nums)   # False (7 is odd)

# Works on any iterable, not just generators
any([False, False, True])        # True
all([True, True, True])          # True
all([])                          # True  (vacuously true — empty has no counter-example)
any([])                          # False (no truthy element found)

# Practical use: check if any/all values in a dict meet a condition
scores = {'Alice': 85, 'Bob': 42, 'Charlie': 91}
all(v >= 50 for v in scores.values())   # False (Bob failed)
any(v >= 90 for v in scores.values())   # True  (Charlie)
```

---

## `datetime` Module

```python
from datetime import datetime, date, timedelta

# --- Current date and time ---
now   = datetime.now()          # local date + time
today = date.today()            # local date only

# --- Create specific date/datetime ---
dt = datetime(2024, 6, 15, 9, 30, 0)   # year, month, day, hour, min, sec
d  = date(2024, 6, 15)

# --- Access components ---
dt.year     # 2024
dt.month    # 6
dt.day      # 15
dt.hour     # 9

# --- Formatting: datetime → string ---
dt.strftime('%Y-%m-%d')          # '2024-06-15'
dt.strftime('%d/%m/%Y %H:%M')   # '15/06/2024 09:30'
# %Y=4-digit year  %m=month  %d=day  %H=hour(24h)  %M=minute  %S=second

# --- Parsing: string → datetime ---
datetime.strptime('2024-06-15', '%Y-%m-%d')
datetime.strptime('15/06/2024 09:30', '%d/%m/%Y %H:%M')

# --- Arithmetic with timedelta ---
tomorrow   = today + timedelta(days=1)
last_week  = today - timedelta(weeks=1)
in_90_days = today + timedelta(days=90)

deadline = date(2024, 12, 31)
days_left = (deadline - today).days    # number of days between two dates
```

---

## `copy` Module — Shallow vs Deep Copy

```python
import copy

# Assignment — NOT a copy; both variables point to the same object
a = [1, [2, 3]]
b = a
b[0] = 99
print(a)    # [99, [2, 3]]  — a is affected because b IS a

# --- Shallow copy: new outer object, but nested objects are still shared ---
a = [1, [2, 3]]
b = copy.copy(a)       # also: b = a.copy()  or  b = a[:]
b[0] = 99              # only changes b's first element
print(a)               # [1, [2, 3]]  — outer is independent

b[1].append(4)         # modifies the INNER list, which is still shared
print(a)               # [1, [2, 3, 4]]  — a is affected!

# --- Deep copy: fully independent copy at every level ---
a = [1, [2, 3]]
b = copy.deepcopy(a)
b[1].append(4)         # inner list is now a separate object
print(a)               # [1, [2, 3]]  — a is NOT affected

# Rule of thumb:
# Flat structures (no nesting)  → shallow copy is fine
# Nested structures             → use deepcopy to be safe
```

---

## `itertools` Module

```python
import itertools

# --- chain: flatten multiple iterables into one ---
list(itertools.chain([1, 2], [3, 4], [5]))      # [1, 2, 3, 4, 5]
list(itertools.chain.from_iterable([[1,2],[3]])) # [1, 2, 3]  (from a list of lists)

# --- combinations: choose r items, ORDER does NOT matter ---
list(itertools.combinations([1, 2, 3], 2))
# [(1, 2), (1, 3), (2, 3)]

# --- permutations: choose r items, ORDER matters ---
list(itertools.permutations([1, 2, 3], 2))
# [(1,2),(1,3),(2,1),(2,3),(3,1),(3,2)]

# --- product: cartesian product (nested for-loops in one line) ---
list(itertools.product([1, 2], ['a', 'b']))
# [(1,'a'),(1,'b'),(2,'a'),(2,'b')]

# --- islice: lazy slice of an iterable (no indexing needed) ---
gen = (x**2 for x in range(1000))
list(itertools.islice(gen, 5))      # [0, 1, 4, 9, 16]  — only computes first 5

# --- groupby: group consecutive elements by a key ---
# NOTE: input must be sorted by the key first
data = [('A', 1), ('A', 2), ('B', 3), ('B', 4)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# A [('A', 1), ('A', 2)]
# B [('B', 3), ('B', 4)]
```

---

## `@dataclass`

`@dataclass` auto-generates `__init__`, `__repr__`, and `__eq__` from field annotations — no boilerplate.

```python
from dataclasses import dataclass, field

@dataclass
class Student:
    name: str
    age:  int
    grades: list[float] = field(default_factory=list)  # mutable default must use field()
    active: bool = True                                  # immutable default is fine directly

s = Student('Ada', 20)
s.grades.append(9.5)
print(s)   # Student(name='Ada', age=20, grades=[9.5], active=True)

# __eq__ is auto-generated — compares all fields
s1 = Student('Ada', 20)
s2 = Student('Ada', 20)
s1 == s2   # True

# --- frozen=True: makes instances immutable (like a namedtuple but with type hints) ---
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x = 3.0   # raises FrozenInstanceError

# --- order=True: auto-generates __lt__, __le__, __gt__, __ge__ for sorting ---
@dataclass(order=True)
class Item:
    priority: int
    name: str

items = [Item(3, 'low'), Item(1, 'high'), Item(2, 'mid')]
sorted(items)   # sorted by priority first
```

---

## `textwrap` Module

```python
import textwrap

text = 'The quick brown fox jumps over the lazy dog'

# fill: wrap text to a max width, return one string with newlines
textwrap.fill(text, width=15)
# 'The quick brown\nfox jumps over\nthe lazy dog'

# wrap: same, but returns a list of lines instead
textwrap.wrap(text, width=15)
# ['The quick brown', 'fox jumps over', 'the lazy dog']

# shorten: truncate to width, adding a placeholder
textwrap.shorten(text, width=20)          # 'The quick brown [...]'

# dedent: strip common leading whitespace (great for multi-line strings)
textwrap.dedent('''
    line 1
    line 2
''')                                      # removes the shared indent

# indent: add a prefix to every line
textwrap.indent('a\nb', '> ')             # '> a\n> b'

# Classic one-liner (HackerRank "Text Wrap")
def wrap(string, max_width):
    return textwrap.fill(string, max_width)
```

---

## Format Spec Mini-Language

The `:` inside an f-string / `format()` controls width, alignment, sign, and precision — `{value:[fill][align][sign][width][,][.precision][type]}`.

```python
n = 42
pi = 3.14159

# Alignment & width  (< left, > right, ^ center)
f'{n:>6}'      # '    42'   right-align in width 6
f'{n:<6}'      # '42    '   left-align
f'{n:^6}'      # '  42  '   center
f'{n:*^6}'     # '**42**'   center, fill with '*'
f'{n:06}'      # '000042'   zero-pad

# Numbers
f'{pi:.2f}'    # '3.14'     2 decimal places
f'{pi:8.2f}'   # '    3.14' width 8, 2 decimals
f'{1234567:,}' # '1,234,567' thousands separator
f'{0.256:.1%}' # '25.6%'    percentage
f'{255:x}'     # 'ff'       hex   (b=binary, o=octal, X=upper hex)
f'{n:+}'       # '+42'      always show sign

# Debugging (3.8+): show the expression AND its value
f'{n=}'        # 'n=42'
```

---

## Walrus Operator `:=` (3.8+)

Assign a value *and* use it in the same expression — avoids computing/reading something twice.

```python
# Assign inside a condition
if (n := len(data)) > 10:
    print(f'{n} items — too many')

# Read until a sentinel, no duplicate call
while (line := input()) != 'quit':
    print(line)

# Filter + reuse the computed value in a comprehension
results = [y for x in data if (y := f(x)) is not None]
```

---

## Structural Pattern Matching — `match` (3.10+)

```python
def handle(command):
    match command.split():
        case ['go', direction]:                 # bind the second word
            return f'moving {direction}'
        case ['drop', *items]:                   # capture the rest into a list
            return f'dropping {items}'
        case ['quit' | 'exit']:                  # OR pattern
            return 'bye'
        case _:                                  # wildcard (default)
            return 'unknown'

# Match on structure / types
match point:
    case (0, 0):                 print('origin')
    case (x, 0):                 print(f'on x-axis at {x}')
    case (x, y) if x == y:       print('diagonal')          # guard clause
    case {'name': n, 'age': a}:  print(f'{n} is {a}')       # dict pattern
    case Point(x=px, y=py):      print(f'Point {px},{py}')  # class pattern
```

---

## Context Managers (`with`)

```python
# The `with` statement guarantees setup/teardown even if an error occurs.
# Any object implementing __enter__ / __exit__ works.
import time

class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    def __exit__(self, exc_type, exc_val, tb):
        print(f'took {time.perf_counter() - self.start:.3f}s')
        # return True to suppress the exception; falsy re-raises it

with Timer():
    do_work()

# contextlib.contextmanager — build one from a generator (less boilerplate)
from contextlib import contextmanager

@contextmanager
def open_db():
    conn = connect()
    try:
        yield conn          # everything before yield = setup, after = teardown
    finally:
        conn.close()

with open_db() as conn:
    conn.query(...)

# Manage multiple resources in one line
with open('a.txt') as fa, open('b.txt', 'w') as fb:
    fb.write(fa.read())

# suppress: ignore specific exceptions without try/except
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove('maybe.txt')
```

---

## `functools` Module

```python
from functools import lru_cache, cache, partial, reduce, wraps

# --- lru_cache / cache: memoize a function's results ---
@lru_cache(maxsize=None)          # @cache is the 3.9+ shorthand for maxsize=None
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
fib(100)                          # fast — subresults cached
fib.cache_info()                  # hits/misses stats

# --- partial: pre-fill some arguments, get a new callable ---
def power(base, exp): return base ** exp
square = partial(power, exp=2)
square(5)                         # 25

# --- reduce: fold an iterable to a single value ---
reduce(lambda a, b: a * b, [1, 2, 3, 4])   # 24

# --- wraps: preserve name/docstring when writing decorators ---
def log(f):
    @wraps(f)                     # without this, wrapper.__name__ becomes 'wrapper'
    def wrapper(*a, **k):
        return f(*a, **k)
    return wrapper
```

---

## `re` — Regular Expressions

```python
import re

text = 'Contact: ada@mail.com, alan@dev.io'

re.search(r'\d+', 'abc123')          # first match object (.group() → '123'), else None
re.match(r'\d+', '123abc')           # match only at the START of the string
re.findall(r'\w+@\w+\.\w+', text)    # ['ada@mail.com', 'alan@dev.io']  (list of all)
re.sub(r'\s+', '_', 'a  b   c')      # 'a_b_c'  (replace)
re.split(r'[,;]\s*', 'a, b; c')      # ['a', 'b', 'c']

# Groups & named groups
m = re.search(r'(\w+)@(\w+)', 'ada@mail')
m.group(0), m.group(1), m.group(2)   # ('ada@mail', 'ada', 'mail')

m = re.search(r'(?P<user>\w+)@(?P<host>\w+)', 'ada@mail')
m.group('user')                      # 'ada'

# Compile once, reuse many times (faster in loops)
pattern = re.compile(r'\d{4}-\d{2}-\d{2}')
pattern.findall(log_text)

# Common tokens: \d digit  \w word char  \s whitespace  .  any
#   + 1-or-more  * 0-or-more  ? optional  {n,m} range  ^ start  $ end
```

---

## `pathlib` — Modern File Paths

```python
from pathlib import Path

p = Path('data') / 'raw' / 'file.csv'   # '/' joins paths cross-platform

p.name          # 'file.csv'
p.stem          # 'file'
p.suffix        # '.csv'
p.parent        # Path('data/raw')
p.exists()      # bool
p.is_file()     # bool
p.is_dir()      # bool

Path('out').mkdir(parents=True, exist_ok=True)   # create dirs, no error if present
p.read_text()                                     # read whole file as str
p.write_text('hello')                             # write str to file

# Globbing
list(Path('.').glob('*.py'))            # files in this dir
list(Path('.').rglob('*.csv'))          # recursive search
```

---

## `enum` — Enumerations

```python
from enum import Enum, auto

class Color(Enum):
    RED   = 1
    GREEN = 2
    BLUE  = auto()          # auto-assigns 3

Color.RED           # <Color.RED: 1>
Color.RED.name      # 'RED'
Color.RED.value     # 1
Color(1)            # <Color.RED: 1>  — look up by value
Color['RED']        # <Color.RED: 1>  — look up by name
list(Color)         # [<Color.RED: 1>, <Color.GREEN: 2>, <Color.BLUE: 3>]

# IntEnum compares to ints; StrEnum (3.11+) compares to strings
```

---

## `heapq` & `bisect`

```python
import heapq, bisect

# --- heapq: min-heap (smallest pops first) ---
h = [3, 1, 4, 1, 5]
heapq.heapify(h)              # rearrange list into a heap, O(n)
heapq.heappush(h, 2)
heapq.heappop(h)             # 1  (always the smallest)
heapq.nsmallest(3, h)        # 3 smallest
heapq.nlargest(3, h)         # 3 largest
# Max-heap trick: push negated values

# --- bisect: binary search on a SORTED list ---
arr = [1, 3, 5, 7, 9]
bisect.bisect_left(arr, 5)   # 2  (leftmost insert position)
bisect.insort(arr, 4)        # insert keeping sorted → [1, 3, 4, 5, 7, 9]
```

---

## Sorting with Keys & `operator`

```python
from operator import itemgetter, attrgetter

people = [('Ada', 30), ('Bob', 25), ('Cara', 30)]

# key= controls what to sort by
sorted(people, key=lambda p: p[1])              # by age
sorted(people, key=itemgetter(1))               # same, faster/clearer
sorted(people, key=itemgetter(1, 0))            # by age, then name (tie-break)
sorted(people, key=lambda p: (-p[1], p[0]))     # age DESC, name ASC

# Sort objects by attribute
sorted(cars, key=attrgetter('speed'), reverse=True)

# max/min accept key= too
max(people, key=itemgetter(1))                  # ('Ada', 30) — first max

# Transpose rows <-> columns with zip(*)
rows = [(1, 2, 3), (4, 5, 6)]
list(zip(*rows))                                # [(1, 4), (2, 5), (3, 6)]
```
