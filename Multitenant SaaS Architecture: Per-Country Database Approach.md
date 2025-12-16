# Multitenant SaaS Architecture: Per-Country Database Approach

---

## 1. **Concept of Multitenancy**

**Multitenancy** is a software architecture paradigm allowing a single application instance to serve multiple customers (tenants), isolating each tenant’s environment while sharing resources. 

### **Types of Multitenancy by Data Storage:**
- **Shared Database, Shared Schema:** All tenants' data in the same tables; data separation via a column (e.g., `tenant_id`).
- **Shared Database, Separate Schemas:** Different schemas within the same DB for each tenant.
- **Separate Database Per Tenant:** Each tenant has an independent database (best isolation, higher ops overhead).

---

## 2. **Architecture for Per-Country Database in SaaS Application**

### **Tech Stack Example:**
- **Frontend:** React
- **Backend/API:** Laravel

### **Geographical Separation Requirement:**
- **Multi-country** support, but **one database per country**.

---

### **High-Level Architecture Diagram (Text Art)**

```
         +-----------------------------+
         |      React Frontend         |
         +-----------------------------+
                   |        |
     [user DE]     |        |   [user US]
                   V        V
             +---------------------+
             |   Load Balancer     |
             +---------------------+
                   |   
                   V
         +-----------------------+
         |   Laravel Backend     |
         | (Multi-DB Switching)  |
         +-----------+-----------+
                       |
         +-------------+-------------+
         |      |             |      |
      [DB-DE][DB-US]...[DB-IN][DB-JP]
         Separate database per country
```

---

### **Flow Explanation:**

1. **User initiates request via React Frontend** (country context established via selector, subdomain, or header).
2. **API request includes country code** (header, e.g., `X-Country: DE`).
3. **Laravel middleware reads country code, switches DB connection** (`config(['database.default' => ...]) + DB::purge()`).
4. **OLTP operations**: Insert/update/select in correct country’s database.
5. **Country isolation and compliance guaranteed** by separate DBs.

---

## 3. **How to Handle Initial Login and Authentication**

### **Problem:**
- Country (tenant) code is needed to select DB, but you may only know this after user logs in.
- Usernames might not be globally unique.

### **Best Practice: Always require or infer the country code BEFORE authentication.**
- **Login form includes country selector**; user chooses country on login screen.
- **API request includes country info** (as header or part of payload).
- **Laravel switches to correct DB before authentication.**

**Example Frontend (pseudo-code):**
```jsx
POST /api/login
Headers: X-Country: DE
Body: { "email": "user@example.com", "password": "secret" }
```

**Example Laravel Middleware:**
```php
public function handle($request, Closure $next)
{
    $country = $request->header('X-Country');
    $connections = ['DE' => 'mysql_de', ...];
    if (isset($connections[$country])) {
        config(['database.default' => $connections[$country]]);
        DB::purge($connections[$country]);
    }
    return $next($request);
}
```

---

## 4. **Code Explanation: DB::purge($connections[$country])**

`DB::purge($connection)` tells Laravel to forget any current database connection for the given connection name, so the next time a query runs, Laravel will establish a **new, fresh connection** based on the updated configuration.  
This avoids risks of using an old or incorrect connection when dynamically switching DBs per request.

---

## 5. **Cons of Per-Request DB Switching Approach**

| Cons                        | Impact on Scalability                       |
|-----------------------------|---------------------------------------------|
| **High connection overhead**| Each request may open a new DB connection, which is resource-intensive and slow. |
| **No connection pooling**   | Frequent purge/open cycles prevent Laravel/PHP from reusing DB connections, leading to DB connection exhaustion. |
| **Potential for data leaks**| Bugs or race conditions in switching may cause requests to access the wrong DB, risking tenant data exposure. |
| **Complex maintenance**     | More config/dbs to monitor, backup, alert.  |

---

## 6. **Risks Detailed**

### **High Connection Overhead**
- Frequent, short-lived connections (due to purging) => resource waste, slower requests, server overload at high concurrency.

### **No Connection Pooling**
- Losing persistent connections increases latency, exhausts connection limits on DB server under load.

### **Potential for Data Leaks**
- If DB switch logic is skipped, delayed, or buggy, a request may run queries against the wrong DB — leaking sensitive data between tenants/countries.

---

## 7. **Safer Multitenancy Approaches**

### **A. Shared DB, Row-Level Isolation**

- **One DB. All data gets a `country_code` column.**
- Use a global Laravel scope to always filter by current country.

**Example:**
```php
// In a model's boot() method
static::addGlobalScope('country', function ($query) {
    $code = app('country_code'); // set in middleware
    if ($code) {
        $query->where('country_code', $code);
    }
});
```

### **B. Shared DB, Separate Schema Per Tenant**
- One DB, multiple schemas. Use libraries like [stancl/tenancy].

### **C. If Per-Database is Required**

1. **Switch only in early middleware before any query.**
2. **Extensively test for race conditions** (especially in jobs/queues).
3. **Enforce tenant context in JWT/session, validate every request.**
4. **Add logging to audit all switches and queries by tenant.**

---

## 8. **For Massive Scale: Deploy Connection Pooling Proxies**

### **Why**
- Laravel/PHP apps open/close DB connections frequently (esp. with per-request switching, FPM).
- **Connection pooling proxies** (like PgBouncer for PostgreSQL, ProxySQL for MySQL) sit between your app and DB instances, maintaining a pool of reusable connections so that your DBs don't get overwhelmed by short-lived connects/disconnects.

### **How-To Example:**

**Install PgBouncer (Postgres):**
```sh
sudo apt-get install pgbouncer
```
**Sample config:**
```ini
[databases]
de_db = host=127.0.0.1 port=5432 dbname=de_db
us_db = host=127.0.0.1 port=5432 dbname=us_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
default_pool_size = 50
max_client_conn = 2000
pool_mode = transaction
```
**Point Laravel to PgBouncer:**
```php
// .env
DB_HOST=127.0.0.1
DB_PORT=6432 # PgBouncer port
DB_DATABASE=de_db
// etc
```

### **Benefits**
- Massive reduction in DB connection overhead.
- Protects DB from exhaustion.
- Allows per-request switching without penalty (DB pooler manages persistent backend connections).

---

## 9. **Summary Table**

| Approach                       | Data Isolation | Connection Scalability | Risk of Data Leak | Ops Overhead | Use Case             |
|---------------------------------|---------------|-----------------------|-------------------|--------------|----------------------|
| Shared DB, row isolation        | Logical       | High                  | Very low (if scoped correctly) | Low          | Most SaaS apps          |
| Shared DB, schema per tenant    | Logical+      | High                  | Very low          | Med          | Custom tenants        |
| Per-DB, with pooling proxy      | Physical      | Med-High (pooler)     | Medium            | High         | Absolute isolation    |
| Per-DB, direct (no pooling)     | Physical      | Low                   | High              | High         | Rare; legacy         |

---

## 10. **Additional Mitigations and Recommendations**

- Always switch DB connections **before** any queries are executed.
- Prefer shared DB + row isolation when legal/compliance allows.
- For per-DB setups, **use connection pooling proxies** in all environments (not just prod).
- Use thorough integration tests for cross-tenant/country data access.
- If using per-DB, encrypt backups and automate geo-compliance ops.

---

## 11. **References**
- [Laravel Docs: Database Connections](https://laravel.com/docs/database)
- [PgBouncer](https://www.pgbouncer.org/)
- [ProxySQL](https://proxysql.com/)
- [Stancl Tenancy for Laravel](https://tenancyforlaravel.com/)

---

**Prepared for: SaaS architects, Laravel engineers, DevOps practitioners.**