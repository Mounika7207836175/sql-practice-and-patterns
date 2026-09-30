# Employees Whose Name Starts and Ends With the Same Letter

## Problem
Table `employee_names(emp_id, emp_name)`. Return employees whose name's first letter matches its last letter (case-insensitive).

| emp_id | emp_name |
|--------|----------|
| 1 | Anna |
| 2 | Bob |
| 3 | Charlie |
| 4 | David |
| 5 | Amanda |

Expected result: **Anna, Bob, David, Amanda** (Charlie starts with C, ends with E — excluded).

---

## Approach 1: SUBSTR + LENGTH

```sql
SELECT emp_name
FROM employee_names
WHERE UPPER(SUBSTR(emp_name, 1, 1)) = UPPER(SUBSTR(emp_name, LENGTH(emp_name), 1));
```

**How it works**
- `SUBSTR(emp_name, 1, 1)` gets the first character.
- `SUBSTR(emp_name, LENGTH(emp_name), 1)` gets the last character, by starting exactly at the position equal to the string's length.
- `UPPER()` on both sides makes the comparison case-insensitive.

---

## Approach 2: LEFT / RIGHT

```sql
SELECT emp_name
FROM employee_names
WHERE UPPER(LEFT(emp_name, 1)) = UPPER(RIGHT(emp_name, 1));
```

**How it works**
- `LEFT(string, n)` and `RIGHT(string, n)` grab the first/last `n` characters directly, no need to calculate a position with `LENGTH`.
- Cleaner and more readable than Approach 1 for this case.

---

## Approach 3: LIKE with a dynamic pattern

```sql
SELECT emp_name
FROM employee_names
WHERE UPPER(emp_name) LIKE CONCAT(UPPER(LEFT(emp_name, 1)), '%', UPPER(LEFT(emp_name, 1)));
```

**How it works**
- For each row, a custom pattern is built using that row's own first letter twice: e.g. for "Charlie" the pattern becomes `'C%C'`.
- `%` in `LIKE` means "any number of characters in between."
- The row's own (uppercased) name is then checked against its own custom pattern — "CHARLIE" doesn't match `'C%C'` since it ends in E, so it's excluded.

**Point of confusion clarified in session:** the pattern isn't a generic "any letter" check — both `%`-flanking characters come from that same row's first letter, so the pattern only matches if the name's last letter happens to equal its first letter.

---

## Approach 4: Regex — attempted, not achievable in Snowflake as originally planned

**The idea:** use a capture group and a backreference to express "same character at start and end" in one pattern:
```
^(.).*\1$
```
- `^` / `$` — start / end of string
- `(.)` — capture the first character
- `.*` — any characters in between
- `\1` — a backreference, meaning "the same text captured by group 1"

**Why it doesn't work here:** Snowflake's `RLIKE` / `REGEXP_LIKE` uses the RE2 regex engine, which **does not support backreferences** (`\1`) at all — this is a deliberate limitation of RE2, not a mistake in the pattern. So this exact approach cannot be used in Snowflake, even though it works in engines like PostgreSQL or Python's `re` module.

**Errors hit along the way (session debugging):**
- Wrapping the whole pattern in square brackets, `'[^(.).*\1$]'`, is invalid for a different reason: square brackets define a "character class" (match any one of the listed characters), not a full pattern — so this returned 0 rows regardless of the backreference issue.

**Workaround (not yet built):** extract the first and last letters separately using `REGEXP_SUBSTR`, then compare them with a plain `=`, avoiding backreferences entirely. Noted as a possible follow-up, not completed in this session.

---

## Comparison

| Approach | Technique | Notes |
|----------|-----------|-------|
| 1. SUBSTR + LENGTH | String functions | Needs LENGTH to find the last character |
| 2. LEFT / RIGHT | String functions | Cleanest, most direct |
| 3. LIKE dynamic pattern | Pattern matching | Builds a per-row pattern using CONCAT |
| 4. Regex with backreference | Not usable in Snowflake | RE2 engine has no backreference support |

## Key Takeaways
- `LEFT`/`RIGHT` are more direct than `SUBSTR` + `LENGTH` for grabbing string ends.
- A `LIKE` pattern can be built dynamically per row with `CONCAT`, using that row's own data inside the pattern.
- Square brackets `[...]` in regex define a character class, not a whole pattern — don't wrap an entire pattern in them.
- Snowflake's regex engine (RE2) does not support backreferences (`\1`); patterns relying on "match the same captured text again" need a different technique, such as extracting pieces with `REGEXP_SUBSTR` and comparing them directly.
