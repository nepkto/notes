# ACID-Compliant Ticket Booking in MySQL: Stored Procedure Example

This guide explains how to implement **ACID compliance** and safe concurrent ticket booking using a **plain MySQL stored procedure**, through a conversational approach. It covers real-world doubts, use cases, and how row-level locking works in MySQL.

---

### 👤 **Anil:**  
I want to make sure that if two users try to book the last ticket at the same time in my MySQL-backed ticket booking system, only one succeeds. How can I achieve this with a stored procedure?

---

### 👨‍💻 **Manoj:**  
Great question! You can use **transactions** and **row-level locking** in your stored procedure to guarantee ACID compliance and prevent double-booking.

---

## Example: Ticket Booking Stored Procedure (MySQL)

Suppose you have a `tickets` table:

| ticket_id | status    | user_id | ... |
|-----------|-----------|---------|-----|
| 1         | available | NULL    | ... |
| 2         | booked    | 5       | ... |

Here's a sample stored procedure:

```sql
DELIMITER //

CREATE PROCEDURE BookTicket(
    IN in_ticket_id INT,
    IN in_user_id INT
)
BEGIN
    DECLARE current_status VARCHAR(20);

    -- Start a transaction
    START TRANSACTION;

    -- Lock the ticket row for update (pessimistic locking)
    SELECT status INTO current_status
    FROM tickets
    WHERE ticket_id = in_ticket_id
    FOR UPDATE;

    IF current_status = 'available' THEN
        UPDATE tickets
        SET status = 'booked', user_id = in_user_id
        WHERE ticket_id = in_ticket_id;
        COMMIT;
        SELECT 'Success' AS result;
    ELSE
        ROLLBACK;
        SELECT 'Ticket not available' AS result;
    END IF;
END;
//

DELIMITER ;
```

---

### How Does This Ensure ACID Compliance?

- **Atomicity:** The booking is all-or-nothing, wrapped in a transaction.
- **Consistency:** The ticket is only booked if available.
- **Isolation:** `FOR UPDATE` locks the selected row until transaction completes; other transactions must wait.
- **Durability:** Once committed, the booking persists even after a crash.

---

### 👤 **Anil:**  
How does this work if two users try to book at exactly the same time?

---

### 👨‍💻 **Manoj:**  
When two users call the procedure for the same ticket:

- **User A's** transaction gets the lock first.
- **User B's** transaction waits for User A to finish.
- After User A books the ticket, User B's transaction resumes, finds the ticket already booked, and fails gracefully.

**Result:** Only one user books the ticket. No double-booking!

---

### Key Points

- **FOR UPDATE:** Locks the selected row, blocking other transactions from modifying it until the first transaction ends.
- **Transactions:** Wrap all logic to guarantee atomicity and isolation.
- **Error Handling:** Use conditionals to rollback if booking is not possible.

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

- Use transactions and row-level locking in MySQL for ACID-compliant, concurrency-safe booking.
- This prevents double-booking and ensures data reliability even under simultaneous requests.

---

> Still have questions?  
> Try simulating concurrent bookings or check out [MySQL locking documentation](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html) for more detail!
