---
layout: post
title:  "Unix Epoch: Why we do What we do"
date:   2024-11-30 23:54:36 +0000
categories: ["Overflow Issues"]
tags: ["unix epoch", "integer overflow", "python", "c", "Y2K"]
---

### **TL;DR**

> Have you ever wondered how computer calculates the current timestamp across different devices with consistency? This blog explores why a specific date was chosen, its technical implications, and the challenges it has faced over the years.
{: .prompt-info }

### Pre-Context

I was working on a software development project in python, where I needed to grab the current timestamp and find the difference from a given timestamp. Simple, right?
Like many others, including ChatGPT, I imported the `time` module and used `time()` function to retrieve the current time in UTC.

And what did I get? Not the human readable format: `2024-12-01 00:14:35 UTC`, oh no. I got a random number that looked like this:

```python
import time
print(time.time()) # returns float with [seconds].[subseconds]
print(int(time.time())) # returns int with [seconds]

# output
1733012075.388034
1733012075
```

A quick fix using `datetime` module with `utcfromtimestamp` function worked like a charm.

```python
import time
from datetime import datetime
datetime.utcfromtimestamp(int(time.time())).strftime('%Y-%m-%d %H:%M:%S UTC')

# output
2024-12-01 00:14:35 UTC
```

Alright! That issue is resolved, and I can now focus on other development tasks. Yet, deep down, I can't help but wonder — what's really happening under the hood?

you know how it is when curiosity strikes. Somewhere in the back of my mind, it yelled -- <mark>Why we do What we do?</mark>

- Why did I get a random number as If I ran `random.random()` in python.
- Why is it just an integer counting seconds?
- And what's the story behind this epoch time concept anyway?

### Why Numbers?

Here is a visualization of how my mind tries to stack up questions one by one.

<img alt="Mind Visualization Stack" src="https://lh3.googleusercontent.com/pw/AP1GczMLhNJxOyiIAEsD9LPik5BODkMmj4m7O8s-5U_2z8rKTNZrY8GCQA1GiANUO-xoerE-Zz1bfQd1aIwrAHYqcFEqtG282pitU92y7Vrdh_MrU_Z_vUeLEV9hoKe6n0OHDzNHlvC7oYVa_Una6WKn_imA=w548-h386-s-no" />

In this blog, Let's dig deeper.

- The `2024-12-01 00:14:35 UTC` date format is a human readable format but computers don't understand that. All they know is binaries (0s and 1s).
- Every thing related to computers like Large Language Models (LLMs), relies on numbers, math, and transformers that assign numerical weights to decide the next word.
- Similarly, whenever you deal with computers, you need to tone down and narrow things down to numbers specifically 0s and 1s.
- For instance, dates are often represented as strings (`utcfromtimestamp`) which are stored as integers (`time`), which are ultimately sequences of binary digits.

> These date numbers are language-neutral, machine-readable and can be used for mathematical operations like additions or subtractions to calculate different dates.
{: .prompt-tip }

To convince myself, I said:

> *"Those random numbers are necessary to represent time data using integers for simplicity, precision, and accuracy."*

A moment after this, I thought: What should be the reference point to all these numbers? and I get an answer that it's January 1, 1970. Oh boy!

<img alt="Mind Visualization Stack" src="https://lh3.googleusercontent.com/pw/AP1GczMGtUuS3XsMJgo8r_boFO-cUfw_Hy3_tXawNOoqlexjY_ax5kD9JiN2DY-VlUjv3c_kSpZzRzV7bZqBFG_DZufrMJtg15DuravBepfIDOPwz6M-Ix2NOVeRuwQxrS3a_8qIzvXfMQHWRgV_hqWl651u=w548-h386-s-no" />

### Why January 1, 1970?

A reference date is needed to provide a common starting point for calculating time intervals and comparing timestamps across different systems. For Instance, to represent December 1, 2024, the system counts seconds from January 1, 1970.

This is a LLM generated script that calculates seconds between two dates without any modules:

```python
# sample LLM generated script that manually evaluates the seconds between two given date
def is_leap_year(year):
    """Returns True if the year is a leap year, False otherwise."""
    if (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0):
        return True
    return False

def days_in_month(year, month):
    """Returns the number of days in a month, accounting for leap years."""
    days_in_months = {
        1: 31, 2: 28, 3: 31, 4: 30, 5: 31, 6: 30,
        7: 31, 8: 31, 9: 30, 10: 31, 11: 30, 12: 31
    }
    if month == 2 and is_leap_year(year):
        return 29
    return days_in_months[month]

def calculate_seconds_since_epoch(year, month, day):
    """Calculates the number of seconds since January 1, 1970 to a given date."""
    # Start from 1970
    total_seconds = 0

    # Add full years in seconds
    for y in range(1970, year):
        total_seconds += 365 * 24 * 60 * 60  # Regular year
        if is_leap_year(y):
            total_seconds += 24 * 60 * 60  # Add one extra day for leap year

    # Add full months of the current year in seconds
    for m in range(1, month):
        total_seconds += days_in_month(year, m) * 24 * 60 * 60  # Days in each month

    # Add days of the current month in seconds
    total_seconds += (day - 1) * 24 * 60 * 60  # Days before the given day

    return total_seconds

# Current time: December 1, 2024
current_year = 2024
current_month = 12
current_day = 1

# Time from the epoch to the current time
seconds_since_epoch = calculate_seconds_since_epoch(current_year, current_month, current_day)
print("Seconds since January 1, 1970 to December 1, 2024:", seconds_since_epoch)

```

Similarly, you get the same mysterious timestamp when you execute `time.time()`

> *But, Why did Engineers chose January 1, 1970?*

It wasn't arbitrary. When unix was being developed in the late 1960s, the engineers needed a standardized way to handle the time across systems. Also, it was the beginning of the decade, so the first date of the decade was chosen. Everything went well until a significant issue came up.

### Integer Overflow

<img alt="Breaking Bad - Say My Name" src="https://steamuserimages-a.akamaihd.net/ugc/854975340501406948/419F40761A85D6277540D8D7292E12D339400684/?imw=5000&imh=5000&ima=fit&impolicy=Letterbox&imcolor=%23000000&letterbox=false" />

In Unix systems, time is stored as the number of seconds elapsed since the Epoch. This is implemented using a `signed 32-bit integer`, which can represent numbers from `-2,147,483,648` to `2,147,483,647`.

This means that time can be accurately represented only until January 19, 2038. Beyond this, an integer overflow occurs, commonly referred to as the `Y2K38` issue.

```c
signed int max_time = 2147483647;
signed int overflow_time = max_time + 1;
printf("%d", overflow_time);
```

> -2147483648
{: .prompt-danger }

When this C program runs on 32-bit systems, it overflows with the signed integer dealing with boundary values and prints the smallest value of the 32 bit integer which may be misinterpreted as December 13, 1901.

### Potential Impact

- After January 2038, legacy systems in banking, aviation, healthcare, and embedded systems that still use 32-bit time representations will face issues, potentially crashing or misbehaving.
- Unix Epoch will no longer be valid implementation and developer will need to come up with new approaches
- However it is very unlikely to impact the 64 bit systems which has larger range of values (over billions).
- firmware and hardware level changes need to be done via add on hardware or extending the 32 bit limits

### Mitigations

- To completely avoid the issue, migrate the 32-bit systems to 64-bit systems
- Use an alternative time format that doesn't rely on the 32 bit integer tracking

> - This brings to an end. I hope this help developers grasp the importance of understanding what a function is truly intended to do and how it can affect systems.
> - asking yourself - what, why and understanding the fundamentals, their functionality, limitations, and potential impacts will serve you well in the long run.
{: .prompt-tip }
