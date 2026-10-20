# Шаблон скилла naming-check

Часть главы [14-стартовый набор](../guide/14-%D1%81%D1%82%D0%B0%D1%80%D1%82%D0%BE%D0%B2%D1%8B%D0%B9%20%D0%BD%D0%B0%D0%B1%D0%BE%D1%80.md). Скилл состоит из двух файлов: SKILL.md и скрипта проверки всего хранилища. Создайте папку `00-atlas/agents/skills/naming-check/` и два файла по блокам ниже. Замените `<ВАШЕ ИМЯ>` и `ГГГГ-ММ-ДД`.

Файл `00-atlas/agents/skills/naming-check/SKILL.md`:

````markdown
---
name: naming-check
description: "Проверка имен файлов и папок на соответствие правилам CLAUDE.md. Применяй перед созданием любого нового файла и при заданиях на переименование."
metadata:
  author: <ВАШЕ ИМЯ>
  created: ГГГГ-ММ-ДД
  version: 1.0.0
  origin: derived
  skill_status: действует
allowed-tools:
  - Read
  - Glob
  - Grep
---

# Naming-check: проверка имен файлов и папок

Проверь имена на соответствие правилам именования из CLAUDE.md.

Нарушения (в порядке частоты):
1. Заглавные буквы (кроме CLAUDE.md, SKILL.md, README.md).
2. Буква «е» с двумя точками вместо обычной «е».
3. Тире (длинное или среднее) вместо дефиса.
4. Типографские кавычки.
5. Символы : * ? " < > | \ /.
6. Точка или пробел в конце имени.
7. Зарезервированные имена Windows: con, prn, aux, nul, com1-com9, lpt1-lpt9.
8. В папке скиллов 00-atlas/agents/skills/ имя папки не в kebab-case латиницей.

Для каждого нарушения выведи: путь, что нарушено, предложенное имя.

Файлы не переименовывай, только отчет. Переименование ломает ссылки и делается по явному заданию.

Проверка всего хранилища скриптом (запускает пользователь на сервере, скрипт ничего не меняет):

```bash
sudo -u brain python3 /opt/second-brain/vault/00-atlas/agents/skills/naming-check/scripts/check-names.py
```

## История изменений

| Версия | Дата | Что изменилось |
|---|---|---|
| 1.0.0 | ГГГГ-ММ-ДД | Первая версия из стартового набора. |
````

Файл `00-atlas/agents/skills/naming-check/scripts/check-names.py`. Если синхронизация показывает файл .py в Obsidian как скрытый, создайте его на сервере от пользователя brain командой `sudo -u brain nano`, а не через Obsidian:

````python
#!/usr/bin/env python3
"""Проверка имен файлов и папок хранилища на соответствие CLAUDE.md.
Только отчет. Ничего не переименовывает."""
import os
import re
import sys
import unicodedata

VAULT = "/opt/second-brain/vault"
SKIP_DIRS = {".obsidian", ".git", ".trash", "node_modules"}
SKIP_PREFIX = "."
# Файлы, для которых регистр обязателен
CASE_EXCEPTIONS = {"CLAUDE.md", "SKILL.md", "README.md"}
# Папка, внутри которой действует kebab-case латиницей
KEBAB_ZONE = os.path.join(VAULT, "00-atlas", "agents", "skills")
FORBIDDEN_CHARS = set(':*?"<>|\\/' + "\u2014\u2013" + "«»"
                      + "“”‘’")
WIN_RESERVED = ({"con", "prn", "aux", "nul"}
                | {f"com{i}" for i in range(1, 10)}
                | {f"lpt{i}" for i in range(1, 10)})
KEBAB_RE = re.compile(r"^[a-z0-9]+(-[a-z0-9]+)*$")


def check(name, in_kebab_zone):
    problems = []
    if name in CASE_EXCEPTIONS:
        return problems
    stem = os.path.splitext(name)[0]
    if in_kebab_zone:
        if not KEBAB_RE.match(stem):
            problems.append("не kebab-case латиницей (зона скиллов)")
        return problems
    if name != name.lower():
        problems.append("заглавные буквы")
    if "\u0451" in name or "\u0401" in name:
        problems.append("буква U+0451 (е с двумя точками)")
    bad = sorted(set(name) & FORBIDDEN_CHARS)
    if bad:
        problems.append("запрещенные символы: " + " ".join(bad))
    if name.endswith(".") or name.endswith(" "):
        problems.append("точка или пробел в конце")
    if stem.lower() in WIN_RESERVED:
        problems.append("зарезервированное имя Windows")
    if unicodedata.normalize("NFC", name) != name:
        problems.append("ненормализованный юникод (NFD)")
    return problems


def suggest(name):
    if name in CASE_EXCEPTIONS:
        return name
    s = name.lower()
    s = s.replace("\u0451", "е")
    s = s.replace("\u2014", "-").replace("\u2013", "-")
    for ch in "«»“”‘’":
        s = s.replace(ch, "")
    for ch in ':*?"<>|\\/':
        s = s.replace(ch, "")
    s = unicodedata.normalize("NFC", s)
    return s.rstrip(". ")


def main():
    found = 0
    dup_case = []
    for root, dirs, files in os.walk(VAULT):
        dirs[:] = [d for d in dirs
                   if d not in SKIP_DIRS and not d.startswith(SKIP_PREFIX)]
        in_kebab = root.startswith(KEBAB_ZONE)
        # имена, различающиеся только регистром
        seen = {}
        for n in dirs + files:
            key = n.lower()
            if key in seen:
                dup_case.append(os.path.join(root, n))
            seen[key] = n
        for name in dirs + files:
            problems = check(name, in_kebab)
            if problems:
                found += 1
                rel = os.path.relpath(os.path.join(root, name), VAULT)
                print(rel)
                print("    нарушения: " + "; ".join(problems))
                print("    предлагается: " + suggest(name))
                print()
    if dup_case:
        print("ВНИМАНИЕ: имена, различающиеся только регистром")
        print("(возможные дубликаты от синхронизации между системами;")
        print("не переименовывать без разбора):")
        for p in dup_case:
            print("  " + os.path.relpath(p, VAULT))
        print()
    print("Всего нарушений: " + str(found))
    return 0


if __name__ == "__main__":
    sys.exit(main())
````

Проверка скрипта: команда из SKILL.md выводит строку «Всего нарушений: N» и не падает с ошибкой.
