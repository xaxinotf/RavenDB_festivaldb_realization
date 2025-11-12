

# Порівняльна таблиця

| Вимір                   | SQL (PostgreSQL)                            | RavenDB (NoSQL документи)                                                                     |                                   |
| ----------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------------- | --------------------------------- |
| Дані/модель             | Нормалізовані таблиці + PK/FK, M:N через зв’язки | Денормалізовані документи (JSON), вкладені під-об’єкти, масиви; «колекції» за типом документа |                                   |
| Схемність               | Жорстка схема (DDL), CHECK/UNIQUE                | Схема гнучка; валідація на рівні застосунку/потоків (patch/scripts) або Custom Conventions    |                                   |
| Ідентифікатори          | `bigint` + sequences                             | Строкові ключі (`artists/1-A`) чи GUID; авто-ген                                              |                                   |
| Зв’язки                 | FK + каскади, композитні ключі                   | Посилання за id, `Include`/`Load` для витягування, часто — **вбудовування** (embed)           |                                   |
| M:N                     | Проміжні таблиці (join)                          | Масиви id або піддокументи; інколи окрема «зв’язкова» колекція                                |                                   |
| Поліморфізм (solo/band) | ISA-підтипи `solo_artist`, `band`                | Один документ `Artist` з полем `type: "solo"                                                  | "band"`; різні поля в одному типі |
| Композитні PK (Stage)   | PK (venue_id, stage_name)                        | Вкладений масив `Venue.stages[]` або ключ виду `stages/venueId                                | stageName`                        |
| Цілісність              | Гарантується СУБД (FK, UNIQUE, CHECK)            | Додаток/індекси; унікальність через **Compare-Exchange** або унікальний індекс                |                                   |
| Транзакції/ACID         | Повні ACID, багатотабличні                       | ACID для документів; підтримка **cluster-wide transactions** за потреби                       |                                   |
| JOIN/запити             | SQL + JOIN, агрегати                             | RQL (подібний до LINQ/SQL), без класичних JOIN; `Include`, `Load`, **Map-Reduce**             |                                   |
| Індекси                 | B-tree/GIN, явні CREATE INDEX                    | **Автоіндекси** + статичні; індекси можуть проєктувати обчислювані поля                       |                                   |
| Агрегації               | `GROUP BY`                                       | **Map-Reduce індекси**, лічильники, time series                                               |                                   |
| Час/зони                | `timestamptz`, перевірки                         | `DateTime`/ISO-8601; індексація по діапазонах, time series як окремий тип                     |                                   |
| Повнотекст/пошук        | `GIN`/`tsvector`                                 | Вбудований full-text, аналізатори мов                                                         |                                   |
| Версіонування           | Тригери/історія вручну                           | **Revisions** (історія документів) «з коробки»                                                |                                   |
| Вкладення               | Окремі таблиці/LOB                               | **Attachments** до документа                                                                  |                                   |
| Події/стріми            | Тригери/NOTIFY                                   | **Subscriptions**, ETL, патчі, зміни через **Change Vector**                                  |                                   |
| Масштабування           | Вертикаль/шардінг (складно)                      | Реплікація/шардінг вбудовано; multi-master                                                    |                                   |
| Бекапи                  | pg_dump/реплікація                               | Snapshots/continual backups                                                                   |                                   |
| Міграції                | Міграції DDL                                     | Мінімальні; еволюція схеми у коді/скриптах                                                    |                                   |
| Типові патерни          | Нормалізація, JOIN                               | **Денормалізація для read-патернів**, вбудовування локальних даних                            |                                   |

---

# Як змоделювати  схему у RavenDB

| SQL-таблиця                                        | Колекція/структура в RavenDB                                                                                                                                     | Нотатки та варіанти                                                                                         |                                                                            |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `artist`, `solo_artist`, `band`                    | `Artists` (одна колекція) з полем `type: "solo"                                                                                                                  | "band"`; для band — `formedYear`, `genre`                                                                   | Поля соліста (`realName`, `birthDate`) з’являються лише коли `type="solo"` |
| `band_member` (M:N)                                | Варіант A: у документі гурту `members: [artistId]` • Варіант B: окрема колекція `BandMembers`                                                                    | A — простіше/швидше читати склад гурту; B — якщо потрібні часті перехресні запити                           |                                                                            |
| `artist_event` (лайн-ап M:N)                       | Варіант A: у `Event` — `lineup: [{artistId, slot, ...}]` • Варіант B: у `Artist` — `gigs: [{eventId, ...}]` • Варіант C: колекція `ArtistEvent`                  | Зазвичай беруть A (подія «володіє» лайнапом), а зворотні посилання тримають асинхронно (patch/subscription) |                                                                            |
| `artist_mentorship` (рекурсивне M:N)               | У `Artist`: `mentors: [artistId]` та/або `mentees: [artistId]`; або колекція `Mentorships`                                                                       | Якщо потрібні запити на «граф» — роби окрему колекцію                                                       |                                                                            |
| `person`                                           | `People`                                                                                                                                                         | Один документ на людину                                                                                     |                                                                            |
| `organizer`                                        | `Organizers`                                                                                                                                                     |                                                                                                             |                                                                            |
| `equipment_provider`                               | `Providers`                                                                                                                                                      | Поле `serviceType` лишається як є                                                                           |                                                                            |
| `venue`                                            | `Venues` з вкладеним масивом `stages: [{name, capacity, type}]`                                                                                                  | Це природна заміна слабкої сутності `stage(venue_id, stage_name)`                                           |                                                                            |
| `stage` (weak entity)                              | Вкладений у `Venue`                                                                                                                                              | Посилання з події — або `venueId` + `stageName`, або дублюємо атрибути сцени в події                        |                                                                            |
| `festival`                                         | `Festivals`                                                                                                                                                      | `directorPersonId`, `mainOrganizerId`; **унікальність `name`** — через Compare-Exchange                     |                                                                            |
| `volunteer_team`, `volunteer_team_person`          | Вкладено в `Festival`: `teams: [{name, coordinatorId, members:[{personId, role}]}]`                                                                              | Якщо членство сильно «живе» окремо — можна виділити окремі колекції                                         |                                                                            |
| `event`                                            | `Events`                                                                                                                                                         | Поля часу; посилання на `festivalId`, `venueId`, `stageName`; `estimatedBudget` як число                    |                                                                            |
| `event_equipment_provider` (M:N)                   | У `Event`: `equipmentProviders: [providerId]` (або об’єкти з деталями)                                                                                           | Простий read-патерн: «що потрібно події?»                                                                   |                                                                            |
| `contract` (тернарний: (artist,event) → organizer) | Варіант A (рекоменд.): у `Event.lineup[]` кожен елемент має `artistId` та `organizerId` • Варіант B: колекція `Contracts` з унікальністю за `(artistId,eventId)` | Для унікальності B — порівняння/блокування через Compare-Exchange                                           |                                                                            |

---



Артисти — компактна проєкція

from Artists as a
order by a.display_name
select {
  id: id(a),
  name: a.display_name,
  type: a.artist_type,
  country: a.country
}

<img width="913" height="916" alt="image" src="https://github.com/user-attachments/assets/6e845807-c8b2-4ecc-9b4b-b34ee02d56dd" />


Колекція + load
from Events as e
order by e.date
load e.venue as v
select {
  event: e.name,
  date:  e.date,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}


<img width="891" height="820" alt="image" src="https://github.com/user-attachments/assets/c3a3dec7-2b5e-47fe-bf0d-3cba73e6129e" />

Якщо поле дати інше (наприклад, startDate)

from Events as e
order by e.startDate
load e.venue as v
select {
  event: e.name,
  date:  e.startDate,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}

<img width="876" height="832" alt="image" src="https://github.com/user-attachments/assets/2f5b30b9-90a9-4169-ab42-5676ea2facfe" />



із індексом (якщо маєш власний Events/ByDate)

order by ставимо до load, а load — перед select.

from index 'Events/ByDate' as e
order by e.date
load e.venue as v
select {
  event: e.name,
  date:  e.date,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}


<img width="878" height="801" alt="image" src="https://github.com/user-attachments/assets/e188f981-4301-43eb-bf99-63e74c3ac3b2" />




поле дати існує й має саме таке ім’я;

from Events as e
order by e.date
select { event: e.name, venueId: e.venue }


<img width="877" height="711" alt="image" src="https://github.com/user-attachments/assets/badff99e-9648-4f79-8845-e5d2e99faa41" />



# Festival DB — RavenDB Demo

This repository contains a small **RavenDB** demo database for a music festival: artists, events, venues, providers, organizers, etc.  
It shows how a relational schema was mapped to a document model, how IDs/collections are unified, and includes a set of ready-to-run **RQL** queries.

> **Tested on** RavenDB **7.1.3** (build 71018).

---

## Contents

```
.
├── rql/                      # Saved example RQL queries
│   ├── artists_list.rql
│   ├── events_with_venue.rql
│   └── (optional) providers_grouped.rql
├── scripts/
│   └── powershell/           # Scripts actually used during migration
│       ├── normalize_events.ps1
│       └── fix_metadata_collections.ps1
├── indexes/                  # (optional) static indexes .rql, if any
├── docs/
│   ├── ERD.pdf               # draw.io export of ER diagram
│   ├── schema_readable.sql   # relational schema (for comparison)
│   └── migration_notes.md    # why this NoSQL model / denormalizations
├── data/                     # (ignored) local RavenData volume
├── docker-compose.yml
├── .gitignore
└── README.md
```

If the full database dump is large, it is **not committed** by default.  
Use **Git LFS** or publish the `.ravendbdump` in Releases/Drive and link it here.

---

## Quick start

### Option A — Docker (recommended)

1. Install Docker Desktop.
2. Start RavenDB:

```bash
docker compose up -d
```
This maps RavenDB to **http://127.0.0.1:8080** with TCP **38888**.

3. Open Studio in the browser → **http://127.0.0.1:8080**.
4. Create database **`festival_db`** (Databases → New Database).

### Option B — Windows zip

If you use the Windows zip (`run.ps1`):
```
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\run.ps1
```
Open **http://127.0.0.1:8080** and create database **`festival_db`**.

---

## Import the sample data

### Import via Studio

Studio → **Settings → Import Database** → *Full Database Export* → choose the dump `.ravendbdump` → **Import**.

- If the dump is not in the repo, get it from Releases (or link provided by the author).
- Alternative: use **Smuggler/rvn** CLI (optional).

---

## RQL — Saved Queries

All queries exist as files in `rql/` so you can open them in Studio and click **Run**.

### 1) Artists (projection)

File: `rql/artists_list.rql`

```rql
from Artists as a
order by a.display_name
select { id: id(a), name: a.display_name, type: a.artist_type, country: a.country }
```

### 2) Events with venue

File: `rql/events_with_venue.rql`

```rql
from Events as e
order by e.date
load e.venue as v
select {
  event: e.name,
  date:  e.date,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}
```

> If you see a parse error like “Expected end of query but got: order”, ensure **`order by` precedes `select`** (as above).

### 3) Events by date range (with parameters)

Create a new query in Studio or save as `rql/events_by_range.rql`:

```rql
from Events as e
where e.date >= $from and e.date <= $to
order by e.date
select { event: e.name, date: e.date }
```

Parameters example in Studio:
```json
{ "from": "2025-10-01T00:00:00Z", "to": "2025-10-31T23:59:59Z" }
```

### 4) Providers grouped (example; adjust names to your data)

If you have a link collection like `EventProviders` with a field `provider` that stores a document ID:

```rql
from EventProviders as ep
load ep.provider as p
group by ep.provider
select {
  providerId: key(),
  provider:   p.display_name || p.name,
  events:     count()
}
order by events desc
```

---

## Document model & IDs

- Collections used: **Artists**, **Events**, **Venues**, **Providers**, **Organizers**, **People**, **Stages**, etc.
- ID prefixes: `artists/*`, `events/*`, `venues/*`, `providers/*`, `organizers/*`, `people/*`, `stages/*`…
- Each document has `@metadata.@collection` set accordingly.
- Events reference venue via `e.venue` (document ID).

If you need to normalize old imports, see `scripts/powershell/*.ps1` which were used to:
- unify event fields (`name`, `date`, `venue`),
- set `@metadata.@collection` by ID prefix without changing actual IDs.

---

## Verification (health checks)

You can keep these as **Saved Queries** in Studio.

### All documents with their collection (for quick audit)

```rql
from @all_docs as d
order by id()
select { id: id(d), coll: metadata(d)['@collection'] }
```

### Events preview sorted by date

```rql
from Events as e
order by e.date
select { id: id(e), name: e.name, date: e.date, venue: e.venue }
```

---

## Troubleshooting

- **“Expected end of query but got: order”** — In RQL, the order is: `from …`, then optional `where`, then **`order by`**, then `select`.  
- **Can’t use `count()`** without `group by` — use a `group by` or count in the client.
- **Patch-by-query errors** — Studio has two tabs: **Query** and **Patch**. Use `from …` in *Query* and code in *Patch* (JS). For REST, pass `{ Query: "...", Patch: { Script: "..." } }`.
- **Collection mismatch on PUT** — include `@metadata.@collection` when replacing documents.
- **No results / nulls after migration** — ensure you’re reading the *inner* `Results` object when using raw REST; in Studio you see the document body directly.

---

## How this maps from SQL

See `docs/schema_readable.sql` and `docs/migration_notes.md`.  
Key design notes:
- Events denormalize venue name for fast display (optional).
- Artist vs. band captured by field `artist_type` and nested `solo`/`band` detail object (optional).
- ID design follows **semantic prefixes** to keep URLs human-readable and to leverage `startsWith()` queries.

---

## Development tips

- Keep **Saved Queries** for defense/demo (Artists, Events+Venue, Grouping).
- Prefer `load` over `include` when shaping projections.
- If you create static indexes, put their RQL into `indexes/`.

---

