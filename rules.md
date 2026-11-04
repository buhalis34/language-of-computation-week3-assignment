# Module 3 – Book Ingestion Rules

## 1. Handling ISBN Problems

My goal is to make sure the database contains accurate book information without guessing missing data. I want the program to fix simple problems but reject anything that cannot be verified.

- **Missing leading zeros:** Add zeros to the beginning of an ISBN-10 if it is too short, but only if the result passes the ISBN-10 check digit test.
- **Spaces and hyphens:** Remove spaces and hyphens before checking the ISBN.
- **Lowercase x:** Change a lowercase x at the end of an ISBN-10 to uppercase X.
- **Scientific notation:** Only repair a rounded ISBN-13 if it can be rebuilt from a valid ISBN-10 on the same row and the rebuilt number matches the stored rounded value. Otherwise, do not guess.
- **Missing ISBN:** If one ISBN is missing but the other is valid, use the valid one. Only calculate the missing ISBN-13 from an ISBN-10, or the missing ISBN-10 from an ISBN-13 starting with 978, when the conversion is valid.
- **Wrong check digit:** Reject the invalid ISBN. If the other ISBN is valid and there is no conflict, use that one.
- **Conflicting ISBNs:** Reject the row if the ISBN-10 and ISBN-13 identify different books.
- **Other barcodes:** Do not treat UPC codes or music codes as ISBNs.
- **Letters or extra digits:** Reject an ISBN that has unexpected letters or extra digits instead of guessing what was intended.
- **ISBN in the wrong column:** Move a valid ISBN-13 found in the ISBN-10 column to the ISBN-13 field only when there is no conflicting value.
- **Copied ISBN from another book:** Reject a row if its ISBN is already associated with a different book.
- **Both ISBNs missing or unusable:** Reject the row and include it in the report.

Repairs should only happen when the correct result can be determined using clear rules. I do not want the program making guesses based on what seems likely.

The original CSV files should never be changed.

## 2. Identifying Duplicate Books

Two rows count as the same book when they have the same valid ISBN-10 or ISBN-13.

I also want the program to compare the title and author information. If the same ISBN is attached to a different title or author, the row should be rejected for review rather than automatically treated as the same book.

If a book has no usable ISBN, the row should be rejected instead of using the title alone to identify it.

## 3. Books Already in the Database

If a book arrives again with the same ISBN and the same information, the program should skip it so there are no duplicates.

If a book has the same ISBN and the same title and author but other information has changed, the program should update the existing record.

If the ISBN belongs to a different title or author, the program should reject the row instead of overwriting the existing book.

Each run should check every row, even if the file has already been processed.

## 4. Ingestion Report

After every run, I want the program to create a report that tells me:

- When the program ran and which CSV file was processed.
- How many rows were checked.
- How many books were loaded for the first time.
- How many existing books were updated.
- How many rows were skipped because the books already existed.
- How many rows were rejected.
- Which rows were rejected and the reason for each rejection.
- Which ISBNs were repaired and what was changed.
- Which books were loaded with a warning, if any.

The report should make it easy for someone to understand what happened without looking through the database.

The program should keep the results from previous runs so I can compare them later.

## 5. ISBN Examples and Expected Results

| # | Original ISBN | Expected Action | Reason |
|---|---|---|---|
| 1 | 0439023483 | Load | Valid ISBN-10 |
| 2 | 9780439023481 | Load | Valid ISBN-13 |
| 3 | 439023483 | Repair | Add missing leading zero |
| 4 | 0-439-02348-3 | Repair | Remove hyphens |
| 5 | 0 439 02348 3 | Repair | Remove spaces |
| 6 | 031200009x | Repair | Change lowercase x to X |
| 7 | 9.78043902348e+12 | Repair if verified | Rebuild from matching valid ISBN-10 |
| 8 | 812971060 | Reject if still invalid | Fails ISBN-10 check after padding |
| 9 | 1854433017 | Reject if no valid alternative | Invalid check digit |
| 10 | 04390A3483 | Reject if no valid alternative | Contains an unexpected letter |
| 11 | 15287463097 | Reject if no valid alternative | Too many digits for ISBN-10 |
| 12 | 76783609419.0 | Reject if no valid alternative | Not a valid ISBN |
| 13 | Empty | Reject if both ISBNs unusable | Cannot reliably identify the book |
| 14 | 9781982147600 | Load as ISBN-13 | Valid ISBN-13 in the wrong column, if no conflict |
| 15 | 9780439023481 (already loaded) | Skip | Same book already exists with unchanged information |

These examples will be used to test whether the program follows my rules correctly.
