# 🛒 Python Web Scraping — Amazon Price Tracker

## Preface

This project builds a fully automated **Amazon price tracker** in Python. It scrapes a product's title and price, logs each reading with a timestamp into a CSV file, and sends an email alert when the price drops below a target threshold.

The steps are laid out in build order — each one adds a single piece until the whole system is assembled into one reusable function that runs on a schedule.

---

## 📋 Table of Contents

1. [Import Libraries](#1-import-libraries)
2. [Connect to Amazon & Pull Data](#2-connect-to-amazon--pull-data)
3. [Clean the Data](#3-clean-the-data)
4. [Create a Timestamp](#4-create-a-timestamp)
5. [Create CSV & Write First Row](#5-create-csv--write-first-row)
6. [Read CSV with Pandas](#6-read-csv-with-pandas)
7. [Append New Data to CSV](#7-append-new-data-to-csv)
8. [Combine Everything into One Function](#8-combine-everything-into-one-function)
9. [Run the Tracker on a Schedule](#9-run-the-tracker-on-a-schedule)
10. [Preview the Final CSV](#10-preview-the-final-csv)
11. [Email Alert When Price Drops](#11-email-alert-when-price-drops)

---

## 1. Import Libraries

> Pull in every library the project needs upfront. Each one handles a specific job — scraping, timing, file writing, data reading, and emailing.

```python
from bs4 import BeautifulSoup
import requests
import time
import datetime
import csv
import pandas as pd
import smtplib
```

| Library | Purpose |
|---|---|
| `requests` | Sends HTTP requests to Amazon |
| `BeautifulSoup` | Parses the raw HTML response |
| `time` | Adds delays between requests |
| `datetime` | Generates timestamps for each price check |
| `csv` | Writes and appends rows to the CSV file |
| `pandas` | Reads and previews the saved CSV |
| `smtplib` | Sends the price alert email |

---

## 2. Connect to Amazon & Pull Data

> Use a `Session` with realistic browser headers so Amazon doesn't immediately block the request. Hit the Amazon homepage first to collect cookies, then request the product page.

```python
# Connect to website and pull data
session = requests.Session()
session.headers.update({
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36 Edg/145.0.0.0",
    "Accept-Language": "en-US,en;q=0.9",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Referer": "https://www.google.com/",
})

# Hit the homepage first to get cookies, then hit the product page (so amazon doesn't think ur a bot)
session.get("https://www.amazon.com")
time.sleep(2)

URL = 'https://www.amazon.com/Funny-Data-Analyst-Definition-Scientist/dp/B07NLP2PKY'
page = session.get(URL)
soup = BeautifulSoup(page.content, "html.parser")

title = soup.find(id="productTitle").get_text()
price = soup.find(id="apex-pricetopay-accessibility-label").get_text()

print(title)
print(price)
```

> 💡 `session.get("https://www.amazon.com")` before the product URL is important — it mimics how a real browser first lands on the homepage and picks up cookies, making the request look more legitimate.

---

## 3. Clean the Data

> The raw scraped text contains extra whitespace and currency formatting. Strip both so the values are clean before saving.

```python
# Clean up data a little bit
price = price.strip()[3:]   # removes leading whitespace and currency prefix (e.g. "Rp.")
title = title.strip()       # removes leading/trailing whitespace

print(title)
print(price)
```

> ⚠️ `[3:]` slices off the first 3 characters — this assumes the currency prefix is exactly 3 characters (e.g. `Rp.` or `US$`). Adjust the slice number if your currency prefix is a different length.

---

## 4. Create a Timestamp

> Record the exact date each price check runs. This gets saved alongside the title and price so you can track price changes over time.

```python
# Create a time stamp for ur output to track when data is collected
today = datetime.date.today()
print(today)
```

**Example output:**
```
2025-03-10
```

---

## 5. Create CSV & Write First Row

> Create the CSV file, write the column headers, then write the first data row. Use `'w'` mode — this creates a fresh file (overwrites if it already exists).

```python
# Create CSV and write headers and data into the file
header = ['Title', 'Price', 'Date']
data = (title, price, today)

with open('AmazonWebScraperDataset.csv', 'w', newline='', encoding='UTF8') as f:
    writer = csv.writer(f)
    writer.writerow(header)
    writer.writerow(data)
```

> 💡 Run this **once** to set up the file with headers. After that, use Step 7 (`'a+'` mode) to append new rows without overwriting.

---

## 6. Read CSV with Pandas

> After writing, confirm the file looks correct by reading it back with Pandas.

```python
df = pd.read_csv(r"C:\Users\ACER\AmazonWebScraperDataset.csv")
print(df)
```

**Expected output:**
```
                          Title   Price        Date
0  Funny Data Analyst T-Shirt   15.99  2025-03-10
```

---

## 7. Append New Data to CSV

> Instead of overwriting, use `'a+'` mode to add new rows to the bottom of the existing file each time the tracker runs.

```python
# Appending data into csv
with open('AmazonWebScraperDataset.csv', 'a+', newline='', encoding='UTF8') as f:
    writer = csv.writer(f)
    writer.writerow(data)
```

| Mode | Behaviour |
|---|---|
| `'w'` | Creates file fresh — overwrites everything |
| `'a+'` | Opens existing file and appends to the end |

---

## 8. Combine Everything into One Function

> Wrap the entire pipeline — connect, scrape, clean, timestamp, save, and alert — into a single `check_price()` function so it can be called repeatedly on a schedule.

```python
# Combine all code above into one function

def check_price():
    session = requests.Session()
    session.headers.update({
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36 Edg/145.0.0.0",
        "Accept-Language": "en-US,en;q=0.9",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Referer": "https://www.google.com/",
    })

    # Hit the homepage first to get cookies, then hit the product page (so amazon doesn't think ur a bot)
    session.get("https://www.amazon.com")
    time.sleep(2)

    URL = 'https://www.amazon.com/Funny-Data-Analyst-Definition-Scientist/dp/B07NLP2PKY'
    page = session.get(URL)
    soup = BeautifulSoup(page.content, "html.parser")

    title = soup.find(id="productTitle").get_text()
    price = soup.find(id="apex-pricetopay-accessibility-label").get_text()

    # Clean up data a little bit
    price = price.strip()[3:]
    title = title.strip()

    today = datetime.date.today()

    header = ['Title', 'Price', 'Date']
    data = (title, price, today)

    with open('AmazonWebScraperDataset.csv', 'a+', newline='', encoding='UTF8') as f:
        writer = csv.writer(f)
        writer.writerow(data)

    if(price < 300000):
        send.mail()
```

> ⚠️ `send.mail()` on the last line should be `send_mail()` — note the underscore. This is a bug in the original code that will raise a `NameError` at runtime.

---

## 9. Run the Tracker on a Schedule

> Call `check_price()` repeatedly for a set number of runs, with a delay between each one. Each run appends a new timestamped row to the CSV.

```python
# Runs check_price after a set time and inputs data into your CSV
max_runs = 2

for i in range(max_runs):
    check_price()
    print(f"Run {i+1}/{max_runs} complete")
    time.sleep(5)

print("Done — all runs complete.")
```

> 💡 Increase `max_runs` and `time.sleep()` for real tracking. For example: `max_runs = 48` with `time.sleep(1800)` checks the price every 30 minutes for 24 hours.

---

## 10. Preview the Final CSV

> After the runs complete, read the CSV with Pandas to verify all rows were recorded correctly.

```python
df = pd.read_csv(r"C:\Users\ACER\AmazonWebScraperDataset.csv")
df
```

**Expected output (after 2 runs):**
```
                          Title   Price        Date
0  Funny Data Analyst T-Shirt   15.99  2025-03-10
1  Funny Data Analyst T-Shirt   15.99  2025-03-10
```

---

## 11. Email Alert When Price Drops

> If the price falls below a target, automatically send an email notification using Gmail's SMTP server.

```python
# If you want to try sending yourself an email (just for fun) when a price hits below a certain level you can try it
# out with this script

def send_mail():
    server = smtplib.SMTP_SSL('smtp.gmail.com', 465)
    server.ehlo()
    #server.starttls()
    server.ehlo()
    server.login('naufalizzudin36@gmail.com', 'xxxxxxxxxxxxxx')

    subject = "The Shirt you want is below Rp. 300,000! Now is your chance to buy!"
    body = "Naufal, This is the moment we have been waiting for. Now is your chance to pick up the shirt of your dreams. Don't mess it up! Link here: https://www.amazon.com/Funny-Data-Analyst-Definition-Scientist/dp/B07NLP2PKY"

    msg = f"Subject: {subject}\n\n{body}"

    server.sendmail(
        'naufalizzudin36@gmail.com',
        msg
    )
```

> ⚠️ **Before this works:**
> - Replace `'xxxxxxxxxxxxxx'` with a real **Gmail App Password** (not your Gmail login password). Generate one at: Google Account → Security → 2-Step Verification → App Passwords.
> - `smtplib.SMTP_SSL` on port `465` uses SSL from the start — `server.starttls()` is commented out correctly since it's not needed with SSL.

---

## 💡 Full Pipeline at a Glance

```
Import libraries
    ↓
Open a session + set browser headers
    ↓
Hit Amazon homepage (get cookies)
    ↓
Request product page → parse HTML
    ↓
Extract title + price → clean text
    ↓
Add today's date as timestamp
    ↓
Append row to CSV
    ↓
Price below target? → send_mail()
    ↓
Wait → repeat (loop)
```

---

## 🐛 Bugs Noted in Original Code

| Step | Issue | Fix |
|---|---|---|
| Step 7 | Missing closing `)` on `open(...)` — `'UTF8' as f:` | Add `)` before `as f` |
| Step 8 | `send.mail()` uses dot instead of underscore | Change to `send_mail()` |
| Step 11 | `server.sendmail()` is missing the recipient email argument | Add recipient as second argument: `server.sendmail('from@gmail.com', 'to@gmail.com', msg)` |

---

> 💬 *Always store credentials like email passwords in environment variables or a `.env` file — never hardcode them directly in your script.*
