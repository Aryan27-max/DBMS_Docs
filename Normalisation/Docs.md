# Database Normalization: Guide By Aryan Gupta Emohis used(❌✅) (1NF → 5NF)

**Normalization** means organizing tables so that **each fact is stored in exactly one place**.
When a fact is stored in many places, those copies eventually disagree. That is the whole problem normalization solves.

Each normal form builds on the previous one. A table in 3NF is also in 1NF and 2NF.

> **Quick vocabulary**
> - **Primary key (PK)**: the column(s) that uniquely identify a row.
> - **Composite key**: a primary key made of 2+ columns.
> - **Non-key attribute**: any column that isn't part of the key.
> - **Dependency (A → B)**: "if I know A, I know B". Example: `student_id → student_name`.

---

## 1. First Normal Form (1NF)

### In simple words
Every cell holds **one value**, every row is **uniquely identifiable**, and the table doesn't **rely on hidden tricks**.

### The 4 rules (core learning)
| # | Rule | Why |
|---|------|-----|
| 1 | Don't use **row order** to convey information | SQL tables have no guaranteed order. "The first row is the winner" is meaningless. Store a `rank` column instead. |
| 2 | Don't **mix data types** in one column | A `height` column holding `180` and `'tall'` can't be compared or summed. |
| 3 | Every table needs a **primary key** | Without one, duplicate rows can exist and you can't target a single row. |
| 4 | No **repeating groups** | No `phone1, phone2, phone3` columns, and no `'98400, 98401'` lists in one cell. |

### Example

❌ **Not 1NF**: a list inside one cell, no primary key

| student_name | phones          |
|--------------|-----------------|
| Asha         | 98400, 98401    |
| Ravi         | 99500           |

✅ **1NF**: one value per cell, keys defined

`students`
| student_id (PK) | student_name |
|-----------------|--------------|
| 1               | Asha         |
| 2               | Ravi         |

`student_phones`
| student_id (PK, FK) | phone (PK) |
|---------------------|------------|
| 1                   | 98400      |
| 1                   | 98401      |
| 2                   | 99500      |

### PostgreSQL
```sql
CREATE TABLE students (
    student_id   SERIAL PRIMARY KEY,           -- Rule 3: has a key
    student_name TEXT NOT NULL                 -- Rule 2: one type per column
);

CREATE TABLE student_phones (
    student_id INT  REFERENCES students(student_id),
    phone      TEXT NOT NULL,                  -- Rule 4: one phone per row
    PRIMARY KEY (student_id, phone)
);

-- Rule 1: if order matters, store it explicitly
CREATE TABLE race_results (
    runner_id INT PRIMARY KEY,
    position  INT NOT NULL UNIQUE              -- don't rely on row order
);
```

---

## 2. Second Normal Form (2NF)

### In simple words
Be in 1NF, and every non-key column must depend on the **entire** primary key, **not just part of it**.

Only matters when the key is **composite** (2+ columns).

### Core learning
> Each non-key attribute must depend on the **entire** primary key.

### Example
Key = `(student_id, course_id)`

❌ **Not 2NF**

| student_id (PK) | course_id (PK) | student_name | grade |
|-----------------|----------------|--------------|-------|
| 1               | DB101          | Asha         | A     |
| 1               | OS102          | Asha         | B     |
| 2               | DB101          | Ravi         | A     |

- `grade` depends on **both** student and course ✅
- `student_name` depends on **only** `student_id` ❌ → "Asha" is repeated on every course she takes.

✅ **2NF**: move the partial dependency to its own table

`students`
| student_id (PK) | student_name |
|-----------------|--------------|
| 1               | Asha         |
| 2               | Ravi         |

`enrollments`
| student_id (PK) | course_id (PK) | grade |
|-----------------|----------------|-------|
| 1               | DB101          | A     |
| 1               | OS102          | B     |
| 2               | DB101          | A     |

### PostgreSQL
```sql
CREATE TABLE students (
    student_id   INT PRIMARY KEY,
    student_name TEXT NOT NULL
);

CREATE TABLE courses (
    course_id TEXT PRIMARY KEY
);

CREATE TABLE enrollments (
    student_id INT  REFERENCES students(student_id),
    course_id  TEXT REFERENCES courses(course_id),
    grade      CHAR(1),
    PRIMARY KEY (student_id, course_id)        -- grade needs BOTH
);
```

---

## 3. Third Normal Form (3NF)

### In simple words
Be in 2NF, and non-key columns must depend **only on the key**, **not on other non-key columns**.

### Core learning
> Each non-key attribute must depend on **the key, the whole key, and nothing but the key**.

### Example

❌ **Not 3NF**

| emp_id (PK) | emp_name | dept_id | dept_name   |
|-------------|----------|---------|-------------|
| 101         | Meena    | D1      | Sales       |
| 102         | Karthik  | D1      | Sales       |
| 103         | Divya    | D2      | Engineering |

`emp_id → dept_id → dept_name`. `dept_name` depends on `dept_id` (a non-key column), not directly on `emp_id`.
This is a **transitive dependency** (see the end).

✅ **3NF**

`departments`
| dept_id (PK) | dept_name   |
|--------------|-------------|
| D1           | Sales       |
| D2           | Engineering |

`employees`
| emp_id (PK) | emp_name | dept_id (FK) |
|-------------|----------|--------------|
| 101         | Meena    | D1           |
| 102         | Karthik  | D1           |
| 103         | Divya    | D2           |

### PostgreSQL
```sql
CREATE TABLE departments (
    dept_id   TEXT PRIMARY KEY,
    dept_name TEXT NOT NULL
);

CREATE TABLE employees (
    emp_id   INT  PRIMARY KEY,
    emp_name TEXT NOT NULL,
    dept_id  TEXT REFERENCES departments(dept_id)
);
```

---

## 4. Boyce-Codd Normal Form (BCNF)

### In simple words
A stricter 3NF. **Every** column, key columns included, must depend only on a key.
In other words: if `A → B`, then `A` must be a key (a candidate key).

### Core learning
> Each **attribute** (not just non-key ones) must depend on **the key, the whole key, and nothing but the key**.

3NF says "non-key attributes". BCNF says "**every** attribute". That one word is the difference.

### Example
Rule: each teacher teaches only one subject. Key = `(student_id, subject)`.

❌ **3NF but not BCNF**

| student_id (PK) | subject (PK) | teacher |
|-----------------|--------------|---------|
| 1               | Math         | Mr. Rao |
| 2               | Math         | Mr. Rao |
| 1               | Physics      | Ms. Iyer|

`teacher → subject`, but `teacher` is not a key. Fact "Mr. Rao teaches Math" is repeated, and nothing stops a row saying "Mr. Rao, Physics".

✅ **BCNF**

`teachers`
| teacher (PK) | subject |
|--------------|---------|
| Mr. Rao      | Math    |
| Ms. Iyer     | Physics |

`student_teachers`
| student_id (PK) | teacher (PK, FK) |
|-----------------|------------------|
| 1               | Mr. Rao          |
| 2               | Mr. Rao          |
| 1               | Ms. Iyer         |

### PostgreSQL
```sql
CREATE TABLE teachers (
    teacher TEXT PRIMARY KEY,
    subject TEXT NOT NULL                      -- teacher → subject, teacher is the key ✅
);

CREATE TABLE student_teachers (
    student_id INT,
    teacher    TEXT REFERENCES teachers(teacher),
    PRIMARY KEY (student_id, teacher)
);
```

---

## 5. Fourth Normal Form (4NF)

### In simple words
Don't put **two independent lists** about the same thing in one table.

A **multivalued dependency** (`A →→ B`) means "A has a *set* of B values". Example: an employee has a set of skills.

### Core learning
> The only multivalued dependencies allowed in a table are **multivalued dependencies on the key**.

### Example
Skills and languages are **unrelated** to each other.

❌ **Not 4NF**: forced to list every combination

| emp_id (PK) | skill (PK) | language (PK) |
|-------------|------------|---------------|
| 101         | SQL        | Tamil         |
| 101         | SQL        | English       |
| 101         | Python     | Tamil         |
| 101         | Python     | English       |

Add one new skill → must add a row **for each** language. Mess.

✅ **4NF**: one table per independent list

`employee_skills`
| emp_id (PK) | skill (PK) |
|-------------|------------|
| 101         | SQL        |
| 101         | Python     |

`employee_languages`
| emp_id (PK) | language (PK) |
|-------------|---------------|
| 101         | Tamil         |
| 101         | English       |

### PostgreSQL
```sql
CREATE TABLE employee_skills (
    emp_id INT,
    skill  TEXT,
    PRIMARY KEY (emp_id, skill)
);

CREATE TABLE employee_languages (
    emp_id   INT,
    language TEXT,
    PRIMARY KEY (emp_id, language)
);
```

---

## 6. Fifth Normal Form (5NF)

### In simple words
If a table can be **rebuilt exactly by joining smaller tables**, split it into those smaller tables.

### Core learning
> It must not be possible to describe the table as the **logical result of joining** some other tables together.

### Example
Business rule: *If a supplier sells a part, and the supplier works on a project, and the project uses that part, then the supplier supplies that part to that project.*

❌ **Not 5NF**

| supplier (PK) | part (PK) | project (PK) |
|---------------|-----------|--------------|
| S1            | Bolt      | Bridge       |
| S1            | Nut       | Bridge       |
| S1            | Bolt      | Tower        |

Because of the rule, this table is just the join of three pairwise facts. Storing it as one table repeats those facts.

✅ **5NF**: store the three pairwise relationships

`supplier_parts`
| supplier | part |
|----------|------|
| S1       | Bolt |
| S1       | Nut  |

`supplier_projects`
| supplier | project |
|----------|---------|
| S1       | Bridge  |
| S1       | Tower   |

`project_parts`
| project | part |
|---------|------|
| Bridge  | Bolt |
| Bridge  | Nut  |
| Tower   | Bolt |

Joining all three gives back the original table exactly.

### PostgreSQL
```sql
CREATE TABLE supplier_parts (
    supplier TEXT, part TEXT,
    PRIMARY KEY (supplier, part)
);

CREATE TABLE supplier_projects (
    supplier TEXT, project TEXT,
    PRIMARY KEY (supplier, project)
);

CREATE TABLE project_parts (
    project TEXT, part TEXT,
    PRIMARY KEY (project, part)
);

-- Rebuild the original table when you need it
CREATE VIEW supplier_part_project AS
SELECT sp.supplier, sp.part, pj.project
FROM supplier_parts sp
JOIN supplier_projects pj ON pj.supplier = sp.supplier
JOIN project_parts pp     ON pp.project  = pj.project AND pp.part = sp.part;
```

> ⚠️ 5NF only applies when such a business rule really exists. If a supplier can sell a part and work on a project *without* supplying that part to it, the 3-column table is correct as-is.

---

## Cheat Sheet

| NF   | One-line test |
|------|---------------|
| 1NF  | One value per cell, has a key, no repeating groups, no meaning in row order |
| 2NF  | No column depends on **part** of a composite key |
| 3NF  | No non-key column depends on **another non-key** column |
| BCNF | For every `A → B`, A is a key |
| 4NF  | No two **independent lists** in one table |
| 5NF  | Table can't be rebuilt by joining smaller tables |

---

## The Anomalies (why we normalize)

Using the un-normalized table from 3NF:

| emp_id | emp_name | dept_id | dept_name |
|--------|----------|---------|-----------|
| 101    | Meena    | D1      | Sales     |
| 102    | Karthik  | D1      | Sales     |
| 103    | Divya    | D2      | Engineering |

###  Update anomaly
Same fact stored in many rows → **you must update all of them**.
Rename "Sales" to "Marketing" and miss one row → now D1 has two names. Data contradicts itself.

```sql
UPDATE emp_dept SET dept_name = 'Marketing' WHERE emp_id = 101;
-- Row 102 still says 'Sales'. Inconsistent!
```

###  Insertion anomaly
**Can't add one fact without another.**
New department D3 "HR" has no employees yet. You can't insert it, because `emp_id` (the key) would be empty.

###  Deletion anomaly
**Deleting one fact accidentally deletes another.**
Divya leaves → delete row 103 → the fact "D2 is Engineering" is gone too.

**After normalization** (separate `departments` table): rename in one row, add departments freely, delete employees safely.

---

## Transitive Dependencies

A **transitive dependency** is an indirect dependency: `A → B → C`, so A determines C only *through* B.

```
emp_id  →  dept_id  →  dept_name
   (key)    (non-key)    (depends on a non-key!)
```

`dept_name` isn't really a fact about the employee. It's a fact about the **department**. Keeping it in the employee table causes all three anomalies above.

**Fix:** move `B → C` into its own table (`departments`), and keep only `B` as a foreign key. Removing transitive dependencies is exactly what **3NF** does.

---

### Remember it in one line
> **Every fact in one place, and every column describes the key, the whole key, and nothing but the key.**