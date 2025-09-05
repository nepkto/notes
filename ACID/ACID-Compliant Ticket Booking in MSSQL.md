# ACID-Compliant Ticket Booking in MSSQL: Stored Procedure Example

This guide explains how to implement **ACID compliance** and safe concurrent ticket booking using a **plain MSSQL stored procedure**, through a conversational approach. It covers real-world doubts, use cases, and the key role of locking hints.

---

### 👤 **Anil:**  
I want to make sure that if two users try to book the last ticket at the same time in my MSSQL-backed ticket booking system, only one succeeds. How can I achieve this with a stored procedure?

---

### 👨‍💻 **Manoj:**  
Great question! You can use **transactions** and **locking hints** in your stored procedure to guarantee ACID compliance and prevent double-booking.

---

## Example: Ticket Booking Stored Procedure (MSSQL)

Suppose you have a `Tickets` table:

| TicketId | Status    | UserId | ... |
|----------|-----------|--------|-----|
| 1        | available | NULL   | ... |
| 2        | booked    | 5      | ... |

Here's a sample stored procedure:

```sql
CREATE PROCEDURE BookTicket
    @TicketId INT,
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        -- Lock the specific ticket row for update (pessimistic locking)
        -- This ensures only one transaction can update it at a time
        DECLARE @CurrentStatus NVARCHAR(20);

        SELECT @CurrentStatus = Status
        FROM Tickets WITH (UPDLOCK, ROWLOCK)
        WHERE TicketId = @TicketId;

        IF @CurrentStatus = 'available'
        BEGIN
            UPDATE Tickets
            SET Status = 'booked', UserId = @UserId
            WHERE TicketId = @TicketId;
            COMMIT TRANSACTION;
            SELECT 'Success' AS Result;
        END
        ELSE
        BEGIN
            ROLLBACK TRANSACTION;
            SELECT 'Ticket not available' AS Result;
        END
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        SELECT ERROR_MESSAGE() AS ErrorMessage;
    END CATCH
END
```

---

### How Does This Ensure ACID Compliance?

- **Atomicity:** The booking is all-or-nothing, wrapped in a transaction.
- **Consistency:** The ticket is only booked if available.
- **Isolation:** `WITH (UPDLOCK, ROWLOCK)` ensures only one transaction can update the row at a time; others wait.
- **Durability:** Committed bookings persist even after a crash.

---

### 👤 **Anil:**  
How does this work if two users try to book at exactly the same time?

---

### 👨‍💻 **Manoj:**  
When two users run the procedure for the same ticket:

- **User A**'s transaction acquires the lock first.
- **User B**'s transaction waits until User A completes.
- After User A books the ticket, User B's transaction resumes, finds the ticket already booked, and fails gracefully.

**Result:** Only one user books the ticket. No double-booking!

---

### Key Points

- **UPDLOCK**: Acquires an update lock, blocking others from reading/writing until done.
- **ROWLOCK**: Ensures row-level, not table-level, locking.
- **Transactions**: Wrap all logic to guarantee atomicity and isolation.
- **Error Handling**: Use `TRY...CATCH` blocks for safe rollbacks.

---

### 👤 **Anil:**  
Where else can I use this pattern?

---

### 👨‍💻 **Manoj:**  
This approach is perfect for:

- Hotel and seat reservations
- Inventory management
- Financial transactions (bank transfers)
- E-commerce stock control

---

## Summary

- Use transactions and locking hints in MSSQL for ACID-compliant, concurrency-safe booking.
- This prevents double-booking and ensures data reliability even under simultaneous requests.

---

> Still have questions?  
> Try simulating concurrent bookings or check out [SQL Server locking documentation](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver16) for more detail!
