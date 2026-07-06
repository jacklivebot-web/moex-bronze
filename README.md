# MOEX Bronze Layer

Локальный bronze-слой исторических биржевых данных Московской биржи, загруженных исключительно через публичный MOEX ISS API (`iss.moex.com`).

Проект ориентирован на пользователя, который клонирует репозиторий с GitHub и запускает его с чистого окружения.

---

## Что делает проект

- Загружает исторические свечи (OHLCV) через публичный ISS API Московской биржи.
- Сохраняет сырьё в Parquet с hive-партиционированием по `instrument/interval/year`.
- Создаёт DuckDB-базу `data/market.duckdb` с вьюхой `bronze`, читающей Parquet напрямую.
- Обеспечивает идемпотентное инкрементальное обновление: повторный `update.py` не создаёт дублей и не перезаписывает данные без необходимости.
- Генерирует отчёт о пропусках (`gaps_report.csv` + `gaps_report.md`).

Слой содержит только сырьё + SQL-доступ. Никакой торговой логики, индикаторов, ресемплинга, очистки выбросов или самодельной склейки continuous-фьючерсов.

---

## Ограничения источника

Публичный MOEX ISS API, описанный в официальной документации:

- [Программный интерфейс АПИ (API) к ИСС](https://www.moex.com/a2193)
- [Программный интерфейс ISS (PDF)](https://www.moex.com/ru/documents/6523)

не предоставляет следующие вещи:

1. **Continuous-фьючерс `RI_CONT`**. В публичном ISS нет инструмента с таким кодом. Запросы к `securities/RI_CONT.json` и `engines/futures/markets/forts/boards/RFUD/securities/RI_CONT/candles.json` возвращают пустой список.
2. **Готовую склейку фьючерсных контрактов**. Документация ISS описывает получение данных по отдельным инструментам и разрезам, но не содержит механизма формирования непрерывной фьючерсной серии.

Что можно сделать самому:

- `RI_CONT` можно построить downstream-скриптом из загруженных отдельных контрактов (`RIU6`, `RIZ6`, `RIH7`, `RIM7` и др.). Для этого нужен календарь экспираций и выбранная стратегия rollover (по объёму, по открытому интересу или по дате экспирации). Этот скрипт выходит за рамки данного задания.
- Доступная глубина истории зависит от инструмента и интервала и ограничивается публичным ISS.

---

## Структура проекта

```
moex-bronze/
├── ingest.py              # Первичная загрузка
├── update.py              # Идемпотентное инкрементальное обновление
├── moex_iss.py            # Клиент MOEX ISS API
├── storage.py             # Работа с Parquet и DuckDB
├── gaps_report.py         # Отчёт о пропусках
├── acceptance_check.sql   # SQL-проверки качества
├── data_dictionary.md     # Описание колонок
├── requirements.txt       # Зависимости Python
├── README.md              # Этот файл
└── data/
    ├── bronze/            # Hive-партиционированные Parquet
    └── market.duckdb      # DuckDB с вьюхой bronze
```

---

## Инструменты и интервалы

Стартовый набор инструментов задаётся в `moex_iss.py`. Его можно расширить параметрами командной строки.

- **Индексы**: `IMOEX`, `RTSI`
- **Акции**: `SBER`, `GAZP`, `LKOH`, `GMKN`, `ROSN`
- **Валюта**: `USD000UTSTOM` (USD/RUB TOM)
- **Фьючерсы РТС**: `RIU6`, `RIZ6`, `RIH7`, `RIM7` и другие контракты родными кодами.
- **Continuous-фьючерс `RI_CONT`**: не доступен в публичном ISS, в bronze будет пустой результат.

Интервалы: `1d` (дневной), `1h` (60 минут), `10m` (10 минут) — максимум, что отдаёт бесплатный ISS.

---

## Схема bronze

| Колонка | Тип | Описание |
|---|---|---|
| `instrument` | VARCHAR | Код инструмента (`IMOEX`, `SBER`, `RIU6`, …) |
| `interval` | VARCHAR | `1d`, `1h`, `10m` |
| `ts` | TIMESTAMP | Время бара, tz-aware, Europe/Moscow (MSK) |
| `open` | DOUBLE | Цена открытия |
| `high` | DOUBLE | Максимум |
| `low` | DOUBLE | Минимум |
| `close` | DOUBLE | Цена закрытия |
| `volume` | DOUBLE | Объём в штуках/контрактах |
| `value` | DOUBLE | Оборот в рублях |

Уникальный ключ: `(instrument, interval, ts)`. Дублей нет.

---

## Установка и запуск (с чистого окружения)

### 1. Клонировать репозиторий

```bash
git clone https://github.com/<user>/moex-bronze.git
cd moex-bronze
```

### 2. Создать виртуальное окружение

```bash
python3 -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate  # Windows
```

### 3. Установить зависимости

```bash
pip install -r requirements.txt
```

Содержимое `requirements.txt`:

```
duckdb>=1.5.0
pandas>=2.2.0
pyarrow>=15.0.0
requests>=2.31.0
python-dateutil>=2.8.0
tabulate>=0.9.0
```

### 4. Первичная загрузка данных

```bash
python ingest.py
```

Это загрузит весь стартовый набор инструментов во всех трёх интервалах за максимально доступную глубину истории.

> **Время выполнения**: для полного стартового набора с десятиминутными барами может потребоваться от 30 минут до 2 часов, в зависимости от скорости ответа MOEX ISS. Для быстрой проверки ограничьте диапазон:
>
> ```bash
> python ingest.py --from-date 2024-01-01 --till-date 2024-06-30
> ```

### 5. Инкрементальное обновление

```bash
python update.py
```

Скрипт:

- находит последний бар для каждого instrument/interval в bronze;
- скачивает только недостающие данные;
- записывает новые строки в Parquet;
- обновляет вьюху `bronze` в DuckDB.

Повторный запуск идемпотентен:

```bash
python update.py
python update.py
```

Второй запуск не должен изменить `total_rows` и `content_hash`.

### 6. Сгенерировать отчёт о пропусках

```bash
python gaps_report.py
```

Результат:

- `gaps_report.csv` — подробный список крупных разрывов.
- `gaps_report.md` — сводка по инструментам и интервалам.

### 7. Проверить качество данных

```bash
# Если установлен DuckDB CLI
duckdb data/market.duckdb < acceptance_check.sql

# Через Python-обёртку
python - <<'PY'
import duckdb
from pathlib import Path

con = duckdb.connect("data/market.duckdb", read_only=True)
sql = Path("acceptance_check.sql").read_text()
for stmt in [s.strip() for s in sql.split(";") if s.strip()]:
    lines = [l for l in stmt.splitlines() if l.strip() and not l.strip().startswith("--")]
    if lines:
        print("BLOCK:", lines[0].lstrip("- "))
        for row in con.execute("\n".join(lines)).fetchall():
            print(" ", row)
con.close()
PY
```

---

## Параметры командной строки

### `ingest.py`

```bash
# Только выбранные инструменты и интервалы
python ingest.py --instruments IMOEX SBER --intervals 1d 1h

# Ограничить диапазон дат
python ingest.py --from-date 2024-01-01 --till-date 2026-07-06

# Свои пути для данных
python ingest.py --data-dir ./data --bronze-dir ./data/bronze

# Справка
python ingest.py --help
```

### `update.py`

```bash
# Обновить весь стартовый набор
python update.py

# Только выбранные
python update.py --instruments SBER USD000UTSTOM --intervals 1d

# Справка
python update.py --help
```

---

## acceptance_check.sql

Файл содержит 6 проверочных блоков:

1. **Полнота** — все инструменты присутствуют во всех трёх интервалах.
2. **Дубли** — отсутствие дубликатов по `(instrument, interval, ts)`.
3. **OHLC** — базовая целостность (с учётом допуска 0.5 для биржевого округления).
4. **NULL** — отсутствие пропусков в ключевых колонках.
5. **Gaps** — список торговых дней с аномально низким числом баров.
6. **Идемпотентность** — контрольный hash всех строк.

---

## Примеры SQL-запросов

### 1. Последние 10 дневных баров SBER

```sql
SELECT *
FROM bronze
WHERE instrument = 'SBER' AND interval = '1d'
ORDER BY ts DESC
LIMIT 10;
```

### 2. Средний дневной оборот по инструментам

```sql
SELECT
    instrument,
    AVG(value) AS avg_value,
    COUNT(*)   AS days
FROM bronze
WHERE interval = '1d'
GROUP BY instrument
ORDER BY avg_value DESC;
```

### 3. Часовые доходности USD/RUB TOM за последний торговый день

```sql
WITH last_day AS (
    SELECT MAX(CAST(ts AS DATE)) AS d
    FROM bronze
    WHERE instrument = 'USD000UTSTOM' AND interval = '1h'
)
SELECT
    b.ts,
    b.close,
    (b.close / LAG(b.close) OVER (ORDER BY b.ts) - 1) * 100 AS pct_return
FROM bronze b, last_day
WHERE b.instrument = 'USD000UTSTOM'
  AND b.interval = '1h'
  AND CAST(b.ts AS DATE) = last_day.d;
```

---

## Важные замечания

- Все timestamps хранятся в tz-aware виде с таймзоной `Europe/Moscow` (MSK).
- `bronze` view в DuckDB читает Parquet-файлы напрямую через hive-партиционирование (`instrument`, `interval`, `year`).
- `RI_CONT` отсутствует в публичном MOEX ISS. Построение continuous series предполагается отдельным downstream-скриптом на основе загруженных контрактов.
- Выполняется только типизация и дедупликация; никаких трансформаций, ресемплинга или фильтрации выбросов.
- `gaps_report` документирует крупные разрывы. Большинство из них объяснимы выходными, праздниками и остановками торгов.

---

## Зависимости

- Python 3.10+
- `duckdb`
- `pandas`
- `pyarrow`
- `requests`
- `python-dateutil`
- `tabulate`

Установка всех зависимостей одной командой:

```bash
pip install -r requirements.txt
```

---

## Лицензия

MIT
