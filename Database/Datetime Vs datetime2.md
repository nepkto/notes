

**`datetime2`** and **`datetime`** are both SQL Server data types for storing date and time values, but they have important differences:

### 1. Precision
- **`datetime`**: Fixed precision (accurate to 1/300th of a second, or ~3.33 milliseconds).
- **`datetime2`**: Variable precision (from 0 to 7 decimal places for seconds, accurate up to 100 nanoseconds).

### 2. Range
- **`datetime`**: 1753-01-01 to 9999-12-31.
- **`datetime2`**: 0001-01-01 to 9999-12-31 (wider range).

### 3. Storage Size
- **`datetime`**: 8 bytes.
- **`datetime2`**: 6 to 8 bytes (depends on precision).

### 4. Recommendation
- **`datetime2`** is recommended for new development because it is more precise, flexible, and uses less storage for lower precisions.

#### Example

```sql
-- datetime
DECLARE @dt datetime = '2024-06-01 12:34:56.123'
-- datetime2
DECLARE @dt2 datetime2(7) = '2024-06-01 12:34:56.1234567'
```

**Summary:**  
Use `datetime2` for better precision, range, and efficiency. Use `datetime` only for legacy compatibility.