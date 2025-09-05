# Online Ticket Booking & ACID Compliance in Laravel: A Conversational Guide

This README walks you through **ACID compliance** and **locking** in Laravel using an online ticket booking scenario. Presented as a conversation between two developers, it covers real-world doubts, use cases, types of locking, and key points for designing robust, concurrency-safe booking systems.

---

### 👤 **Anil:**  
I'm building an **online ticket booking system** in Laravel. I want to make sure that if two users try to book the last ticket at the same time, only one succeeds. I keep hearing about "ACID compliance" and "locking"—how does this actually work in Laravel?

---

### 👨‍💻 **Sam:**  
Great question, Anil! Let's break it down step by step.

## What is ACID Compliance?

**ACID** stands for:

- **Atomicity:** The booking happens all or nothing.
- **Consistency:** The database always stays valid—no double bookings!
- **Isolation:** Simultaneous bookings don't interfere.
- **Durability:** Once booked, the ticket stays booked.

---

### 👤 **Anil:**  
Okay, but how do I *implement* this in Laravel? What does the code look like?

---

### 👨‍💻 **Sam:**  
You need to use **database transactions** and **locking** to enforce ACID properties. Here’s a typical booking function:

```php
use Illuminate\Support\Facades\DB;

function bookTicket($ticketId, $userId) {
    return DB::transaction(function () use ($ticketId, $userId) {
        // Lock the ticket for update so no one else can book at the same time
        $ticket = Ticket::where('id', $ticketId)
            ->where('status', 'available')
            ->lockForUpdate()
            ->first();

        if (!$ticket) {
            throw new Exception('Ticket not available');
        }

        $ticket->status = 'booked';
        $ticket->user_id = $userId;
        $ticket->save();

        return $ticket;
    });
}
```

---

### 👤 **Anil:**  
But what if **User A** and **User B** both try to book the last ticket *at the same time*? Won't they both lock the row?

---

### 👨‍💻 **Sam:**  
Not quite! Here’s how the database handles it:

#### **Concurrency Scenario:**

- **User A** and **User B** send booking requests for Ticket #1 simultaneously.
- The database allows only **one transaction** to lock the row first (say, User A).
- **User B**'s transaction waits until User A finishes.
- Once User A books the ticket, User B resumes and sees the ticket is already booked. User B’s transaction fails gracefully.

#### **Timeline Table:**

| Time | User A                           | User B                               |
|------|----------------------------------|--------------------------------------|
| t1   | Requests lock on Ticket #1       | Requests lock on Ticket #1           |
| t2   | Lock acquired, books ticket      | Waiting for lock                     |
| t3   | Commits, releases lock           | Lock acquired, sees ticket booked    |
| t4   | Success response                 | Error response                       |

---

### 👤 **Anil:**  
Can you explain the types of locking and how they are used in Laravel?

---

### 👨‍💻 **Sam:**  
Absolutely! There are two main types of locking you might use:

## Types of Locking

### 1. **Pessimistic Locking**
- **Definition:** Assumes conflicts will occur, so it locks resources (rows) as soon as they're accessed.
- **Usage in Laravel:**  
  - **`lockForUpdate()`**: Locks the row for writing. Other transactions must wait until the current transaction is complete.
  - **`sharedLock()`**: Locks the row for reading. Others can read, but can't update until the lock is released.

**Example: Pessimistic Locking**
```php
DB::transaction(function () {
    $ticket = Ticket::where('id', 1)->lockForUpdate()->first();
    // Perform update...
});
```
Here, `lockForUpdate()` ensures only one transaction can update the ticket at a time.

### 2. **Optimistic Locking**
- **Definition:** Assumes conflicts are rare. No locks are placed during read. Before updating, it checks if someone else changed the row.
- **Usage in Laravel:**  
  - Laravel does not provide built-in optimistic locking, but you can implement it using a `version` or `updated_at` column.
  - Before saving, check that the version or timestamp matches. If not, abort the update.

**Example: Optimistic Locking**
```php
// Not built-in; custom implementation
// Add a 'version' column to tickets table
if ($ticket->version === $posted_version) {
    $ticket->version++;
    $ticket->save();
} else {
    throw new Exception('Ticket has been modified, please reload.');
}
```

---

### 👤 **Anil:**  
When should I use pessimistic vs. optimistic locking?

---

### 👨‍💻 **Sam:**  
- **Pessimistic locking:** Use when conflicts are likely and you need strict data integrity (e.g., booking systems, banking).
- **Optimistic locking:** Use when conflicts are rare and you want maximum performance (e.g., low-traffic web apps).

---

### 👤 **Anil:**  
Are there other use cases for this approach?

---

### 👨‍💻 **Sam:**  
Absolutely! ACID-compliant transactions and locking are vital for:

- **Bank money transfers** (preventing double-spending)
- **Hotel room reservations**
- **Inventory management**
- **Seat selection**
- **E-commerce stock control**

---

## 📝 **Summary**

- **ACID compliance** ensures reliability and consistency in bookings.
- **Row-level locking** prevents double-booking—even in simultaneous requests.
- Laravel's `DB::transaction()` and `lockForUpdate()` make it easy to implement pessimistic locking.
- Optimistic locking can be custom-implemented for cases where performance is more important than strict conflict prevention.

---

> Still have questions?  
> Try simulating concurrent bookings or contact your database admin to see locking in action!
