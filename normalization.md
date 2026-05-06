# Лабораторна робота 5: Нормалізація бази даних

> **Мета:** Проаналізувати існуючу схему бази даних ігрової системи, виявити функціональні залежності, перевірити відповідність нормальним формам та виконати нормалізацію до 3NF для усунення надлишковості та аномалій.

---

## Зміст

1. [Аналіз початкової схеми та функціональні залежності](#1-аналіз-початкової-схеми-та-функціональні-залежності)
2. [Покрокова нормалізація](#2-покрокова-нормалізація)
3. [Фінальна схема у 3NF (SQL DDL)](#3-фінальна-схема-у-3nf-sql-ddl)
4. [Висновки](#4-висновки)

---

## 1. Аналіз початкової схеми та функціональні залежності

Дослідивши початкову структуру таблиць, можна виділити наступні ключові функціональні залежності (ФЗ):

### Таблиця `Account`

```
account_id → email, account_password, username, registration_date, online, status, in_game_time
```

### Таблиця `game_server`

```
server_id → server_name1, ip_address, status, current_characters
```

### Таблиця `Region`

```
region_id → server_id, region_name, region_biome, region_min_level
```

### Таблиця `Game_Character` *(початкова — до нормалізації)*

```
character_id → account_id, server_id, region_id, respawn_structure_id, nickname,
               character_level, character_health, character_stamina,
               character_class, character_position
```

> ⚠️ **Виявлена транзитивна залежність:**
> `character_id → region_id → server_id`
> Атрибут `server_id` у `Game_Character` визначається через `region_id`, а не безпосередньо через `character_id`. Крім того, атрибути `character_health` та `character_stamina` залежать від `character_class`, що також є транзитивною залежністю.

### Таблиця `ItemTemplate`

```
item_id → item_name, item_type, item_description, item_weight, item_max_stack, rarity, item_cost
```

### Таблиця `Game_Structure`

```
structure_id → region_id, structure_name, structure_position, structure_type, is_buyable, structure_cost
```

### Таблиця `Character_Quests`

```
{character_id, quest_id} → quest_status
```

### Таблиця `Structure_ownership`

```
{character_id, structure_id} → structure_role
```

> ⚠️ **Виявлена проблема:** У початковій схемі таблиця `Structure_ownership` не мала визначеного первинного ключа, що допускало повне дублювання записів.

---

## 2. Покрокова нормалізація

### Перехід до Першої нормальної форми (1NF)

**Вимога 1NF:** Усі атрибути є атомарними, рядки унікальні (немає повторюваних груп).

**Проблема:**

Таблиця `Structure_ownership` не мала визначеного первинного ключа. Відсутність унікального ідентифікатора рядків допускала повне дублювання записів, що порушує визначення відношення та 1NF.

**Оцінка решти таблиць:**

Усі інші таблиці містять лише атомарні значення (жодного стовпця зі списками або множинними значеннями), тому вони відповідають 1NF.

**Рішення:** Додати композитний первинний ключ до `Structure_ownership`:

```sql
ALTER TABLE Structure_ownership
ADD PRIMARY KEY (character_id, structure_id);
```

**Результат:** Схема відповідає **1NF**.

---

### Перехід до Другої нормальної форми (2NF)

**Вимога 2NF:** Таблиця у 1NF + жоден неключовий атрибут не залежить лише від *частини* складеного ключа.

**Оцінка таблиць зі складеними ключами:**

| Таблиця | Складений ключ | Неключові атрибути | Часткова залежність? |
|---|---|---|---|
| `Character_Quests` | `{character_id, quest_id}` | `quest_status` | ❌ Ні — залежить від обох |
| `Structure_ownership` | `{character_id, structure_id}` | `structure_role` | ❌ Ні — залежить від обох |

`quest_status` характеризує конкретне проходження квесту конкретним персонажем — залежить від усього ключа. `structure_role` характеризує роль конкретного персонажа у конкретній структурі — також залежить від усього ключа.

**Рішення:** Часткових залежностей не виявлено. Схема автоматично відповідає **2NF** після виконання вимог 1NF.

---

### Перехід до Третьої нормальної форми (3NF)

**Вимога 3NF:** Таблиця у 2NF + жоден неключовий атрибут не залежить від іншого неключового атрибута (немає транзитивних залежностей).

#### Проблема 1: Транзитивна залежність через `server_id` у `Game_Character`

У початковій таблиці `Game_Character` існував стовпець `server_id`. Оскільки `region_id` вже є зовнішнім ключем до таблиці `Region`, яка сама містить `server_id`, виникає транзитивний ланцюжок:

```
character_id → region_id → server_id
```

Збереження `server_id` у `Game_Character` створює **аномалію оновлення**: якщо регіон мігрує на інший сервер, доведеться оновлювати записи кожного персонажа в цьому регіоні.

**Рішення:** Видалити стовпець `server_id` з таблиці `Game_Character`. Інформацію про сервер персонажа завжди можна отримати динамічно через `JOIN Region`:

```sql
ALTER TABLE Game_Character
DROP COLUMN server_id;
```

#### Проблема 2: Транзитивна залежність через `character_health` та `character_stamina` у `Game_Character`

Атрибути `character_health` та `character_stamina` у початковій схемі залежали від `character_class`, а не безпосередньо від `character_id`:

```
character_id → character_class → character_health, character_stamina
```

Це класична транзитивна залежність: базові характеристики визначаються класом персонажа, а не самим персонажем. При зміні базових параметрів класу довелось би оновлювати кожен запис персонажа цього класу.

**Рішення:** Винести `character_health` та `character_stamina` в окрему таблицю `Character_Class_Info` з первинним ключем `class_name`, а з `Game_Character` ці стовпці видалити:

```sql
CREATE TABLE Character_Class_Info (
    class_name   class_type PRIMARY KEY,
    base_health  INT NOT NULL,
    base_stamina INT NOT NULL
);

ALTER TABLE Game_Character
DROP COLUMN character_health,
DROP COLUMN character_stamina;
```

Тепер `character_class` у `Game_Character` є зовнішнім ключем до `Character_Class_Info(class_name)`, а базові характеристики зберігаються лише один раз.

**Результат:** Схема відповідає **3NF**.

---

## 3. Фінальна схема у 3NF (SQL DDL)

### Типи даних

```sql
CREATE TYPE online_status       AS ENUM ('Online', 'Offline', 'Sleeping');
CREATE TYPE account_status      AS ENUM ('Banned', 'OK', 'Muted', 'Penalized');
CREATE TYPE server_status       AS ENUM ('On', 'Off');
CREATE TYPE item_category       AS ENUM ('weapon', 'resource', 'trash');
CREATE TYPE rarity_type         AS ENUM ('Common', 'Uncommon', 'Rare', 'Legendary');
CREATE TYPE bonus_type          AS ENUM ('Strength', 'Accuracy');
CREATE TYPE structure_type1     AS ENUM ('ruin', 'circus', 'church', 'mine');
CREATE TYPE class_type          AS ENUM ('Mage', 'Warrior', 'Archer', 'Berserker');
CREATE TYPE quest_status_type   AS ENUM ('Active', 'Completed', 'Failed');
CREATE TYPE structure_role_type AS ENUM ('Owner', 'Co_owner', 'Guest');
```

### Таблиці

```sql
CREATE TABLE Account (
    account_id        SERIAL PRIMARY KEY,
    email             VARCHAR(255) NOT NULL UNIQUE,
    account_password  VARCHAR(24)  NOT NULL CHECK (LENGTH(account_password) >= 8),
    username          VARCHAR(24)  NOT NULL UNIQUE,
    registration_date TIMESTAMP,
    online            online_status  NOT NULL,
    status            account_status NOT NULL,
    in_game_time      INTERVAL
);

CREATE TABLE game_server (
    server_id          SERIAL PRIMARY KEY,
    server_name1       VARCHAR(64) NOT NULL,
    ip_address         VARCHAR(45) NOT NULL,
    status             server_status NOT NULL,
    current_characters INT DEFAULT 0 CHECK (current_characters >= 0)
);

CREATE TABLE Region (
    region_id        SERIAL PRIMARY KEY,
    server_id        INT NOT NULL REFERENCES game_server(server_id),
    region_name      VARCHAR(100),
    region_biome     VARCHAR(100),
    region_min_level INT NOT NULL
);

CREATE TABLE Game_Structure (
    structure_id       SERIAL PRIMARY KEY,
    region_id          INT REFERENCES Region(region_id),
    structure_name     VARCHAR(50),
    structure_position DECIMAL(12, 4) NOT NULL DEFAULT 0.0000,
    structure_type     structure_type1 NOT NULL,
    is_buyable         BOOLEAN,
    structure_cost     DECIMAL(10, 2) DEFAULT 0
);

-- Нова таблиця: усуває транзитивну залежність character_class → health/stamina
CREATE TABLE Character_Class_Info (
    class_name   class_type PRIMARY KEY,
    base_health  INT NOT NULL,
    base_stamina INT NOT NULL
);

CREATE TABLE Game_Character (
    character_id         SERIAL PRIMARY KEY,
    account_id           INT REFERENCES Account(account_id),
    region_id            INT REFERENCES Region(region_id),
    respawn_structure_id INT REFERENCES Game_Structure(structure_id),
    nickname             VARCHAR(50),
    character_level      INT DEFAULT 0 NOT NULL
                             CHECK (character_level >= 0 AND character_level <= 100),
    character_class      class_type REFERENCES Character_Class_Info(class_name),
    character_position   DECIMAL(12, 4)
    -- server_id ВИДАЛЕНО (транзитивна залежність через region_id)
    -- character_health, character_stamina ВИДАЛЕНО (винесено до Character_Class_Info)
);

CREATE TABLE ItemTemplate (
    item_id          SERIAL PRIMARY KEY,
    item_name        VARCHAR(50) NOT NULL,
    item_type        item_category NOT NULL,
    item_description TEXT,
    item_weight      DECIMAL(5, 2) DEFAULT 0.0,
    item_max_stack   INT CHECK (item_max_stack > 0 AND item_max_stack <= 64),
    rarity           rarity_type NOT NULL,
    item_cost        DECIMAL(10, 2)
);

CREATE TABLE Inventory_Slot (
    slot_id      SERIAL PRIMARY KEY,
    character_id INT REFERENCES Game_Character(character_id),
    item_id      INT REFERENCES ItemTemplate(item_id),
    item_amount  INT DEFAULT 1 CHECK (item_amount > 0),
    is_equipped  BOOLEAN DEFAULT FALSE
);

CREATE TABLE WeaponStas (
    item_id      INT PRIMARY KEY REFERENCES ItemTemplate(item_id),
    damage       DECIMAL(10, 1),
    weapon_bonus bonus_type
);

CREATE TABLE Quest_System (
    quest_id           SERIAL PRIMARY KEY,
    item_id            INT NOT NULL REFERENCES ItemTemplate(item_id),
    quest_name         VARCHAR(50),
    quest_desciription TEXT,
    quest_xp           DECIMAL(10, 2)
);

CREATE TABLE Character_Quests (
    character_id INT REFERENCES Game_Character(character_id),
    quest_id     INT REFERENCES Quest_System(quest_id),
    quest_status quest_status_type NOT NULL,
    PRIMARY KEY (character_id, quest_id)
);

CREATE TABLE Structure_ownership (
    character_id   INT REFERENCES Game_Character(character_id),
    structure_id   INT REFERENCES Game_Structure(structure_id),
    structure_role structure_role_type,
    PRIMARY KEY (character_id, structure_id)  -- виправлено: додано PK для 1NF
);
```

---

## 4. Висновки

| Крок | Нормальна форма | Виявлена проблема | Рішення |
|---|---|---|---|
| **1NF** | Перша | `Structure_ownership` не мала первинного ключа → можливе дублювання рядків | Додано `PRIMARY KEY (character_id, structure_id)` |
| **2NF** | Друга | Часткових залежностей не виявлено | Змін не потрібно |
| **3NF** | Третя | Транзитивна залежність `character_id → region_id → server_id` | Видалено `server_id` з `Game_Character` |
| **3NF** | Третя | Транзитивна залежність `character_id → character_class → health/stamina` | Створено `Character_Class_Info`, видалено надлишкові стовпці |

**Підсумок:** У результаті нормалізації схема приведена до **3NF**. Кожен факт зберігається рівно один раз, усунено аномалії вставки, оновлення та видалення. Цілісність даних забезпечується зовнішніми ключами, а надлишкові стовпці (`server_id`, `character_health`, `character_stamina` у `Game_Character`) замінено на динамічне отримання через `JOIN`.
