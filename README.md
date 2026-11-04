# Week 3 Assignment

This is my Module 3 assignment for MSAI-5523.

## How to run ingest.py and open the report

These steps are for macOS. Type each command and press Return.

### 1. Open Terminal

Press **Command (⌘) + Space** to open Spotlight, type `Terminal`, and press **Return**.

### 2. Go to the assignment folder

```bash
cd ~/workspace/language-of-computation/week3-assignment
```

### 3. Make sure you are on the assignment branch

```bash
git switch assignment
```

### 4. Check that Python 3 is available

```bash
python3 --version
```

This should print a version number, such as `Python 3.15.2`.

### 5. Load the first CSV file

```bash
python3 ingest.py books.csv
```

When it finishes, it prints one summary line: how many rows were read, loaded, updated, skipped and rejected, plus the number of ISBN repairs and warnings. It also prints where `books.db` and `ingestion-log.json` are.

### 6. Load the second CSV file

```bash
python3 ingest.py books-batch2.csv
```

Each run adds one new entry to `ingestion-log.json`. Earlier runs are kept. Running a file again checks every row again; books that are already in the database are skipped, not added twice.

### 7. Open the report

```bash
open report.html
```

This opens the report in your default web browser. No server is needed.

1. Click **Open ingestion-log.json**.
2. Choose `ingestion-log.json` in the `week3-assignment` folder.
3. The runs appear newest first. Click a section, such as **Rejected rows**, to see its rows.

You can also open the report from Finder: go to `week3-assignment` and double-click `report.html`.
