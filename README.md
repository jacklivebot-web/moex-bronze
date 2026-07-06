# MOEX Bronze Layer

Локальный bronze-слой исторических биржевых данных Московской биржи, загруженных исключительно через публичный MOEX ISS API (`iss.moex.com`).

Этот репозиторий содержит уже сформированные данные: Parquet-файлы с hive-партиционированием и DuckDB-базу `data/market.duckdb` с вьюхой `bronze`. Исходный код загрузчика не публикуется до финального расчёта.

---

## Что уже в репозитории

```
moex-bronze/
├── README.md              # Этот файл
├── data/
│   ├── bronze/            # Hive-партиционированные Parquet (instrument/interval/year)
│   └── market.duckdb      # DuckDB с вьюхой bronze над Parquet
├── gaps_report.md         # Сводный отчёт о пропусках
└── gaps_report.csv        # Детальный отчёт о пропусках
```

Данные можно сразу открывать и запрашивать через DuckDB без дополнительной загрузки.

---

## Схема bronze

| Колонка | Тип | Описание |
|---|---|---|
| `instrument` | VARCHAR | Код инструмента (`IMOEX`, `SBER`, `RIU6`, …) |
| `interval` | VARCHAR | `1d`, `1h`, `10m` |
| `ts` | TIMESTAMP WITH TIME ZONE | Время бара, tz-aware, Europe/Moscow (MSK) |
| `open` | DOUBLE | Цена открытия |
| `high` | DOUBLE | Максимум |
| `low` | DOUBLE | Минимум |
| `close` | DOUBLE | Цена закрытия |
| `volume` | DOUBLE | Объём в штуках/контрактах |
| `value` | DOUBLE | Оборот в рублях |

Уникальный ключ: `(instrument, interval, ts)`. Дублей нет.

---

## Инструменты и интервалы

- **Индексы**: `IMOEX`, `RTSI`
- **Акции**: `SBER`, `GAZP`, `LKOH`, `GMKN`, `ROSN`
- **Валюта**: `USD000UTSTOM` (USD/RUB TOM)
- **Фьючерсы РТС**: отдельные контракты (`RIU6`, `RIZ6`, `RIH7`, `RIM7`)
- **Continuous-фьючерс `RI_CONT`**: не доступен в публичном MOEX ISS (см. раздел ниже)

Интервалы: `1d` (дневной), `1h` (60 минут), `10m` (10 минут).

---

## Быстрый старт: открыть данные

### 1. Клонировать репозиторий

```bash
git clone https://github.com/jacklivebot-web/moex-bronze.git
cd moex-bronze
```

### 2. Установить DuckDB CLI или Python

**CLI (рекомендуется):**

```bash
# Linux/macOS
wget https://github.com/duckdb/duckdb/releases/download/v1.1.3/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip
./duckdb data/market.duckdb
```

**Python:**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install duckdb pandas
python - <<'PY'
import duckdb
con = duckdb.connect("data/market.duckdb", read_only=True)
print(con.execute("SELECT * FROM bronze WHERE instrument='SBER' AND interval='1d' ORDER BY ts DESC LIMIT 5").fetchall())
con.close()
PY
```

### 3. Примеры SQL-запросов

#### Последние 10 дневных баров SBER

```sql
SELECT *
FROM bronze
WHERE instrument = 'SBER' AND interval = '1d'
ORDER BY ts DESC
LIMIT 10;
```

#### Средний дневной оборот по инструментам

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

#### Часовые доходности USD/RUB TOM за последний торговый день

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

## Ограничения источника

Публичный MOEX ISS API, описанный в официальной документации:

- [Программный интерфейс АПИ (API) к ИСС](https://www.moex.com/a2193)
- [Программный интерфейс ISS (PDF)](https://www.moex.com/ru/documents/6523)

не предоставляет:

1. **Continuous-фьючерс `RI_CONT`**. В публичном ISS нет инструмента с таким кодом. Запросы к нему возвращают пустой список.
2. **Готовую склейку фьючерсных контрактов**. Документация описывает получение данных по отдельным инструментам, но не содержит механизма формирования непрерывной фьючерсной серии.

Что можно сделать самому:

- `RI_CONT` можно построить downstream-скриптом из загруженных отдельных контрактов (`RIU6`, `RIZ6`, `RIH7`, `RIM7` и др.). Для этого нужен календарь экспираций и выбранная стратегия rollover (по объёму, по открытому интересу или по дате экспирации).
- Доступная глубина истории зависит от инструмента и интервала и ограничивается публичным ISS.

---

## Проверка качества

Репозиторий включает `gaps_report.md` и `gaps_report.csv` — сводку и детальный список крупных разрывов между барами. Большинство разрывов объяснимы выходными, праздниками и остановками торгов.

Если у вас есть доступ к исходному коду загрузчика, проверки качества выполняются SQL-скриптом `acceptance_check.sql` через DuckDB:

```bash
duckdb data/market.duckdb < acceptance_check.sql
```

Проверки охватывают:
- полноту данных по инструментам и интервалам;
- отсутствие дубликатов по `(instrument, interval, ts)`;
- базовую целостность OHLC;
- отсутствие NULL в ключевых колонках;
- идемпотентность инкрементального обновления.

---

## Воспроизведение с нуля

> **Примечание:** исходный код загрузчика (`ingest.py`, `update.py`, `moex_iss.py`, `storage.py`, `gaps_report.py`) в данный репозиторий не включён. Если потребуется воспроизвести загрузку, код будет предоставлен отдельно после согласования.

Для работы с уже загруженными данными достаточно DuckDB CLI или Python-библиотеки `duckdb`.

---

## Лицензия

MIT
