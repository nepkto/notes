# Timezone-aware Post Application

This application allows users to create and view posts, displaying the creation time according to each user's local timezone. Below is a guide on how to design your database and application logic to support timezone-aware timestamps.

---

## Table of Contents

- [Overview](#overview)
- [Database Design](#database-design)
- [Storing Timestamps in MSSQL](#storing-timestamps-in-mssql)
- [What is UTC?](#what-is-utc)
- [Handling Timestamps in the Application](#handling-timestamps-in-the-application)
- [Best Practices](#best-practices)

---

## Overview

- All timestamps (`created_at`, etc.) are stored in **UTC** in the database.
- The user's timezone is taken into account only when displaying times in the UI.
- This approach ensures consistency and avoids issues related to daylight saving time or server location.

---

## Database Design

**Posts Table Example (MSSQL):**
```sql
CREATE TABLE posts (
    id INT PRIMARY KEY,
    title NVARCHAR(200),
    content NVARCHAR(MAX),
    created_at DATETIMEOFFSET NOT NULL
);
```

**Users Table Example (optional):**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    username NVARCHAR(100),
    timezone NVARCHAR(50) -- e.g., 'Asia/Kathmandu'
);
```

- Use `DATETIMEOFFSET` for timestamps to explicitly store UTC (`+00:00` offset).

---

## Storing Timestamps in MSSQL

- Use UTC functions to ensure database server location does not affect stored time.
- Example insert using UTC:
    ```sql
    INSERT INTO posts (title, content, created_at)
    VALUES ('Hello world', 'This is a post.', SYSDATETIMEOFFSET());
    ```
- For UTC, ensure the offset is `+00:00` (or use `GETUTCDATE()` converted to `DATETIMEOFFSET`).

---

## What is UTC?

- **UTC (Coordinated Universal Time)** is the global time standard.
- It does not change with location or daylight saving time.
- Example: `2025-09-05 17:18:05 +00:00` (UTC)
- All timestamps should be stored in UTC, regardless of server location.

---

## Handling Timestamps in the Application

1. **On Post Creation:**  
   - Save the timestamp as UTC in the database.

2. **On Post Listing:**  
   - Retrieve the UTC timestamp from the database.
   - Convert the UTC time to the user's local timezone before displaying.
   - Conversion can be done in frontend (recommended with JavaScript) or backend (using libraries like `pytz` for Python).

**JavaScript Example:**
```javascript
const utcDate = new Date("2025-09-05T17:18:05Z");
console.log(utcDate.toLocaleString()); // Displays in user's local timezone
```

**Python Example:**
```python
import pytz
from datetime import datetime

def convert_utc_to_local(utc_dt, user_timezone):
    local_tz = pytz.timezone(user_timezone)
    return utc_dt.replace(tzinfo=pytz.utc).astimezone(local_tz)
```

---

## Best Practices

- **Always store timestamps in UTC** in your database.
- **Never store local times**; convert only at display time.
- **Store user timezone** preferences if you need to display times accurately outside of browser-based apps.
- **Use `DATETIMEOFFSET`** type in MSSQL for explicit UTC storage.
- **Convert UTC to local timezone** in your application layer when displaying timestamps.

---

## FAQ

**Q: Will the timestamp in the database depend on the server's location?**  
A: No. If you use UTC functions (`GETUTCDATE()`, `SYSDATETIMEOFFSET()` with `+00:00`), the timestamp will always be in UTC, regardless of the physical location of the server.

---

## References

- [Microsoft Docs: datetimeoffset (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetimeoffset-transact-sql)
- [Wikipedia: Coordinated Universal Time](https://en.wikipedia.org/wiki/Coordinated_Universal_Time)
- [MDN: Date.prototype.toLocaleString()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleString)