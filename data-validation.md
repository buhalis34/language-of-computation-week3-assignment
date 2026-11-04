# Data Validation: ISBN columns in `books.csv` and `books-batch2.csv`

Diagnosis only. Neither CSV file was changed and nothing has been fixed yet.

- **Files checked:** `books.csv` (10,000 data rows, downloaded 2026-11-04 from
  `https://raw.githubusercontent.com/zygmuntz/goodbooks-10k/master/books.csv`) and
  `books-batch2.csv` (31 data rows).
- **Columns checked:** `isbn` (meant to hold an ISBN-10) and `isbn13` (meant to hold an ISBN-13).
- **Line numbers** below are line numbers in the file. Line 1 is the header, so the first book is on line 2.

## How the values were checked

Every value was read as plain text, exactly as stored, with no conversion to numbers. A value counts as **valid** only if it is valid exactly as stored:

| Type | Rule |
|---|---|
| ISBN-10 | Exactly 10 characters: 9 digits, then a digit or an uppercase `X`. Weighted sum (10×d1 + 9×d2 + … + 1×d10, with X = 10) must be divisible by 11. |
| ISBN-13 | Exactly 13 digits, starting with `978` or `979`. Weighted sum (alternating weights 1 and 3) must be divisible by 10. |

Values that fail were then tested to find out *why* they fail. For example: would they pass if missing leading zeros were put back, if spaces and hyphens were removed, or if they were moved to the other column?

The rows were also checked against each other. Do the ISBN-10 and ISBN-13 on a row describe the same book? Is any ISBN used by two different books? Are any rows duplicated?

---

## Summary

| | `books.csv` (10,000 rows) | `books-batch2.csv` (31 rows) |
|---|---|---|
| `isbn` valid as stored | **2,690** (26.9%) | **12** (38.7%) |
| `isbn` invalid as stored | 6,610 | 14 |
| `isbn` empty | 700 | 5 |
| `isbn13` valid as stored | **0** (0%) | **9** (29.0%) |
| `isbn13` invalid as stored | 9,415 | 14 |
| `isbn13` empty | 585 | 8 |

Most of the damage comes from one cause: **the ISBN columns were saved as numbers instead of text** at some point, probably by a spreadsheet or by pandas. Storing them as numbers did two things:

1. It removed leading zeros from ISBN-10s (6,587 rows in `books.csv`).
2. It turned every ISBN-13 into a floating-point number written in scientific notation, such as `9.78043902348e+12`. That rounds away the last digit, which is the check digit (9,410 rows in `books.csv`).

`books-batch2.csv` has the same two problems in the rows copied from `books.csv`. Its new rows add several typical manual-entry mistakes: hyphens and spaces, a lowercase `x`, a letter typed inside the number, an extra digit, a value in the wrong column, a wrong check digit, a copied ISBN, and a duplicated row.

---

## 1. `books.csv`

### 1.1 `isbn` column (ISBN-10)

| Status | Rows | Example (line: value → title) |
|---|---:|---|
| Valid as stored | 2,690 | line 10: `1416524797`, *Angels & Demons* |
| A. Leading zeros missing (valid once the zeros are put back) | 6,587 | line 2: `439023483`, *The Hunger Games* |
| B. Empty | 700 | line 107: empty, *Bossypants* |
| C. Short and wrong check digit, even with the zeros put back | 14 | line 917: `812971060`, *Reading Lolita in Tehran* |
| D. 10 digits but wrong check digit | 9 | line 1444: `9380658797`, *The Secret of the Nagas* |
| **Total** | **10,000** | |

**A. Leading zeros missing (6,587 rows)**
- *What's wrong:* An ISBN-10 always has 10 characters, but these have 7, 8 or 9 digits: 5,563 rows have 9 digits, 913 have 8 and 111 have 7. In every one of these rows, putting the zeros back in front gives a valid ISBN-10. For example, `439023483` becomes `0439023483`, *The Hunger Games*, and `61120081` becomes `0061120081`, *To Kill a Mockingbird*.
- *Probable cause:* The column was read or saved as an integer, which drops leading zeros. Values ending in `X` (814 rows, such as `043965548X`) kept their zeros only because the `X` stopped them from being read as numbers.

**B. Empty (700 rows)**
- *What's wrong:* There is no ISBN-10 at all. In 565 of these rows `isbn13` is empty as well. In the other 135 there is an ISBN-13 but no ISBN-10.
- *Probable cause:* The source records (Goodreads) had no ISBN-10 for the chosen edition. This often happens with e-book or Kindle editions, which use other identifiers, and with ISBN-13s starting with `979`, which have no ISBN-10 form at all. Most of the empty rows are well-known books (*Bossypants*, *Lolita*, *The Stranger*, *A Dance with Dragons*), so the problem is missing data, not books that lack an ISBN.

**C. Short and wrong check digit, even with the zeros put back (14 rows)**
- *What's wrong:* After padding with zeros, the last digit (the check digit) is wrong. For example, line 917 `812971060` becomes `0812971060`, but the correct check digit for `081297106` is `X`. Three of the 14 should end in `X`: lines 917, 2375 and 5027. The full list of lines is 917, 1096, 1544, 1628, 2375, 2600, 3301, 3395, 5027, 6274, 6402, 8423, 8553 and 9733.
- *Probable cause:* The ISBN was already wrong in the source data, or its last digit was mistyped. Where an ISBN-13 exists on the same row, its first 12 digits usually match these first 9 digits, which suggests only the check digit is damaged. Losing the leading zeros (problem A) happened on top of that.

**D. 10 digits but wrong check digit (9 rows)**
- *What's wrong:* The value is the right length but fails the check. Example: line 1444 `9380658797`, which should end in `6`.
- *Probable cause:* In 8 of the 9 rows, putting `978` in front gives a **valid ISBN-13**. For example, `978` + `9380658797` = `9789380658797`. This means someone made the "ISBN-10" by cutting `978` off the front of the ISBN-13 without recalculating the check digit, which uses a different formula in ISBN-10. The 9th row (line 4323, `1400139027`) is probably a plain typo.
- Lines: 1444, 2779, 3474, 3666, 4323, 4810, 6734, 7479, 9188.

### 1.2 `isbn13` column (ISBN-13)

| Status | Rows | Example (line: value → title) |
|---|---:|---|
| Valid as stored | **0** | |
| E. Scientific notation (check digit rounded away) | 9,410 | line 2: `9.78043902348e+12`, *The Hunger Games* |
| F. Empty | 585 | line 107: empty, *Bossypants* |
| G. Plain number ending in `.0`, not an ISBN-13 | 5 | line 959: `76783609419.0`, *The Complete Anne of Green Gables Boxed Set* |
| **Total** | **10,000** | |

**E. Scientific notation (9,410 rows)**
- *What's wrong:* The value is stored as a floating-point number with only 12 significant digits, for example `9.78043902348e+12`. An ISBN-13 has 13 digits, so the 13th digit (the check digit) is rounded away. When that digit was 5 or more, the rounding also changed the 12th digit. Some values show even fewer digits because trailing zeros were dropped: line 12 `9.78159448e+12` is really `9781594480003`, *The Kite Runner*.
- *Can the original be recovered?*
  - **9,239 rows** also have a valid ISBN-10, and the ISBN-13 calculated from it rounds to exactly the stored value. The correct ISBN-13 can be rebuilt from the ISBN-10.
  - **15 rows** have a valid ISBN-10 that **does not match** the stored value. In 3 of them (lines 2895, 3690, 5269) the "ISBN-13" is really a 12-digit UPC product barcode, such as `6.45573003012e+11` = `645573003012`, which is a valid UPC. The other 12 hold a different ISBN-13 than the ISBN-10 on the same row, probably from another edition of the same book. Example: line 623, *Nights in Rodanthe*, has `isbn` `446612707` but `isbn13` `9.78055381395e+12`. One of them (line 1107) starts with `979-0`, a prefix used for printed music (ISMN), not books.
  - **156 rows** have no usable ISBN-10 to check against. For 51 of them only one valid ISBN-13 is consistent with the rounded number. The other 105 have two possible answers, so the true value cannot be known without looking it up.
- *Probable cause:* The column was read as a floating-point number, and the file was later written out with the default float formatting. This happened when the dataset was created, not when it was downloaded.

**F. Empty (585 rows)**
- *What's wrong:* There is no ISBN-13. 565 of these rows have no ISBN-10 either. The other 20 have an ISBN-10, so their ISBN-13 could be calculated.
- *Probable cause:* Same as problem B: the edition record had no ISBN.

**G. Plain number ending in `.0` (5 rows)**
- *What's wrong:* These are short enough to print without scientific notation, but none of them is an ISBN-13:
  - Lines 959, 6143 and 6438 (`76783609419.0`, `25986247522.0`, `31809014745.0`) are **12-digit UPC barcodes with the leading 0 lost**: `076783609419`, `025986247522` and `031809014745` are all valid UPCs. Line 959 is a boxed set, which usually carries a UPC instead of an ISBN.
  - Line 1444 `97893806587.0` is the first 11 digits of the ISBN-13 `9789380658797` (see problem D).
  - Line 8175 `195170342.0` is an ISBN-10 (`0195170342`) in the wrong column. It is the same value as the row's `isbn`.
- *Probable cause:* The same number conversion as E. A UPC or ISBN-10 was entered in the ISBN-13 field in the source data.

### 1.3 Checks across rows in `books.csv`
- No duplicate `book_id` values.
- No ISBN-10 is shared by two different books.

---

## 2. `books-batch2.csv`

This file mixes **16 rows that already exist in `books.csv`** (same `book_id`) with **15 new rows** (`book_id` 10001 to 10014, where 10001 appears twice).

### 2.1 `isbn` column (ISBN-10)

| Status | Rows | Example (line: value → title) |
|---|---:|---|
| Valid as stored | 12 | line 4: `0439139600`, *Harry Potter and the Goblet of Fire* |
| A. Leading zeros missing | 7 | line 6: `441773028`, *The Long Walk Home* |
| B. Empty | 5 | line 2: empty, *Bossypants* |
| H. Hyphens or spaces | 2 | line 30: `0-450-19827-8`, *Salt and Ember* |
| I. Lowercase `x` | 1 | line 27: `031200009x`, *Winter Orchard* |
| J. Letter inside the number | 1 | line 3: `04390A3483`, *The Glass Aviary* |
| K. Wrong check digit | 1 | line 11: `1854433017`, *Small Hours* |
| L. ISBN-13 in the ISBN-10 column | 1 | line 18: `9781982147600`, *An Atlas of Small Kindnesses* |
| M. Too long (11 digits) | 1 | line 20: `15287463097`, *Borrowed Light* |
| **Total** | **31** | |

**A. Leading zeros missing (7 rows):** lines 6, 8, 12, 13, 17, 29 and 32. Six of them are copied from `books.csv` and have the same problem there. Line 6 (*The Long Walk Home*, book 10012) is a new row, which shows the new rows also went through the same number conversion. *Cause:* the column was treated as a number.

**B. Empty (5 rows):**
- Line 2, *Bossypants*: empty in `books.csv` too.
- Line 26, *Harry Potter and the Deathly Hallows*: **data was lost**. `books.csv` has `545010225` / `9.78054501022e+12` for this book, but batch 2 has both columns empty.
- Line 10, *Notes from the Night Ferry*: has a valid ISBN-13 starting with `979` (`9798351220413`). An ISBN-13 starting with `979` has no ISBN-10 form, so this empty value is **correct, not an error**.
- Line 25, *The Cartographer's Error*: has a valid ISBN-13 starting with `978` (`9780062398123`), so its ISBN-10 is missing but could be calculated (`0062398121`).
- Line 22, *Untitled Manuscript* by "Unknown": both columns empty. It looks like a placeholder for an unpublished work.

**H. Hyphens or spaces (2 rows):** line 30 `0-450-19827-8` and line 31 `" 162 001 4750 "` (with spaces at both ends). Both are valid once the hyphens and spaces are removed, and both match the row's ISBN-13. *Cause:* the ISBN was typed or pasted the way it is printed on a book, which a strict check counts as invalid.

**I. Lowercase `x` (1 row):** line 27 `031200009x`. It is valid once the `x` is made uppercase, and it matches its ISBN-13 `9780312000097`. *Cause:* manual entry. The standard requires an uppercase `X`.

**J. Letter inside the number (1 row):** line 3 `04390A3483`. A letter is only allowed as the last character, and only an `X`. *Cause:* probably a typo or a misread character. Replacing the `A` with `2` gives `0439023483`, which is *The Hunger Games*, so this may also be a mixed-up copy of another book's ISBN. The row has no ISBN-13 to check against.

**K. Wrong check digit (1 row):** line 11 `1854433017`. The right length, but the last digit should be `6`. *Cause:* a typo in one digit, most likely the last one. There is no ISBN-13 on the row to confirm which digit is wrong.

**L. ISBN-13 in the ISBN-10 column (1 row):** line 18 `9781982147600` is a valid ISBN-13 entered in the `isbn` column, and the `isbn13` column is empty. Its ISBN-10 would be `1982147601`. *Cause:* the value was entered in the wrong column.

**M. Too long (1 row):** line 20 `15287463097`, *Borrowed Light*, has 11 digits. The first 10, `1528746309`, are a valid ISBN-10, but that number belongs to **another book in the same file**, *The Lantern Keeper's Daughter* (lines 14 and 15). *Cause:* probably a copy-and-paste from the wrong row plus an extra keystroke. Trimming it would *not* give the right ISBN.

### 2.2 `isbn13` column (ISBN-13)

| Status | Rows | Example (line: value → title) |
|---|---:|---|
| Valid as stored | 9 | line 4: `9780439139601`, *Harry Potter and the Goblet of Fire* |
| E. Scientific notation | 14 | line 5: `9.78140130858e+12`, *The Five People You Meet in Heaven* |
| F. Empty | 8 | line 3: empty, *The Glass Aviary* |
| **Total** | **31** | |

- **E. Scientific notation (14 rows):** 13 rows are copied unchanged from `books.csv`. The 14th is new (line 6, *The Long Walk Home*). All 14 match their row's ISBN-10 once rounded, so all can be rebuilt. *Cause:* the same float conversion as in `books.csv`.
- **F. Empty (8 rows):** lines 2, 3, 11, 18, 20, 22, 24 and 26. On line 18 the ISBN-13 is in the wrong column (problem L). On line 26 the value was lost (see 2.1 B).
- **Valid but doesn't match its ISBN-10 (1 of the 9 valid rows):** line 16, *Tides of Marrow Bay*, has `isbn` `1555550126` (valid) and `isbn13` `9781555550134` (also valid). They are two different books: the ISBN-13 for `1555550126` would be `9781555550127`, and the ISBN-10 for `9781555550134` would be `1555550134`. *Cause:* probably a typing slip that produced a neighbouring number. At least one of the two values is wrong, and the file alone can't tell which.

### 2.3 Checks across rows in `books-batch2.csv`

| Problem | Rows | Detail | Probable cause |
|---|---:|---|---|
| Duplicate row | 2 | Lines 14 and 15 are identical: book 10001, *The Lantern Keeper's Daughter*. | The same record was pasted or imported twice. |
| ISBN shared by two different books | 2 | Line 24, *Paper Moons* (book 10013, new), has `isbn` `1416524797`. That is the ISBN of *Angels & Demons* (book 9), which appears on line 21 of this file and in `books.csv`. | Copy-and-paste from another row. Both values pass the checksum, so a per-value check alone would miss this. |
| ISBN-10 contained in another book's value | 1 | Line 20, *Borrowed Light* (see M). | As above. |

---

## 3. Comparing the two files

Of the 16 batch-2 rows that also appear in `books.csv`:

- **14 have identical `isbn` and `isbn13` values**, with the same problems as in `books.csv`.
- **Line 4, book 24, *Goblet of Fire*:** batch 2 has the *correct, complete* values `0439139600` / `9780439139601`. `books.csv` has the damaged `439139600` / `9.7804391396e+12`.
- **Line 26, book 25, *Deathly Hallows*:** batch 2 has both values **empty**, while `books.csv` has them. Data was lost between the files.

---

## 4. Causes, from most to least common

| Probable cause | Problems | Rows affected (`books.csv` + batch 2) |
|---|---|---:|
| ISBN-13 stored as a float (scientific notation, check digit rounded away) | E, G | 9,415 + 14 |
| ISBN-10 stored as an integer (leading zeros dropped) | A | 6,587 + 7 |
| Missing in the source data (no ISBN for that edition) | B, F | 700 / 585 + several |
| Wrong check digit in the source or from a typo | C, D, K | 23 + 1 |
| Non-ISBN codes (UPC barcodes, a music ISMN) in the ISBN-13 field | E (some), G | 7 |
| Manual-entry formatting (hyphens, spaces, lowercase `x`) | H, I | 0 + 3 |
| Manual-entry mistakes (wrong column, extra digit, letter inside, copied from another row, duplicate row, mismatched pair) | J, L, M, 2.2, 2.3 | 0 + 8 |

## 5. Notes for the fixing step (nothing was fixed yet)

- Always read these columns **as text** (for example, `dtype=str` in pandas, or text-formatted columns in a spreadsheet). Otherwise any cleanup will bring the same damage back.
- The scientific-notation ISBN-13s **cannot be repaired from the value alone**. They should be rebuilt from a valid ISBN-10 where one exists (9,239 + 14 rows) and looked up elsewhere for the rest.
- Problems A, H and I can be fixed automatically and safely. Problems C, D, J, K, L, M, the mismatched pair, the shared ISBN and the UPC values need a decision or a lookup, so they should be flagged rather than guessed.
