
# 📂 Secure File Storage in Laravel

This guide explains the **recommended way to store and serve sensitive files** (such as bills, invoices, legal documents, or confidential reports) using Laravel’s filesystem.  

Laravel provides a powerful abstraction layer for file storage (`Storage` facade), but the **choice of directory and access control is critical** when handling sensitive data.

---

## 🚨 Why Not Store in `public/`?

- Files in `public/` or `storage/app/public` (linked via `php artisan storage:link`) are **publicly accessible via URL**.
- Anyone who knows the file path can download it — ❌ **not secure for sensitive data**.
- These directories should only be used for **non-sensitive, user-facing assets** (e.g., product images, profile pictures, banners).

---

## ✅ Recommended Approach

### 1. Store in `storage/app/private`

Create a private folder for sensitive files:

```bash
storage/app/private/bills
storage/app/private/contracts
storage/app/private/reports
```

Example of storing a file:

```php
Storage::put('private/bills/invoice_123.pdf', $fileContent);
```

This ensures the file **cannot be directly accessed via a browser URL**.

---

### 2. Serve Files via a Controller

Since files in `storage/app/private` are not public, you need a controller to handle **authentication + authorization + download**.

```php
use Illuminate\Support\Facades\Storage;

class BillController extends Controller
{
    public function download($filename)
    {
        // ✅ Check permission
        if (!auth()->user()->can('view-bills')) {
            abort(403, 'Unauthorized');
        }

        // ✅ Build path
        $path = 'private/bills/' . $filename;

        // ✅ Check existence
        if (!Storage::exists($path)) {
            abort(404, 'File not found');
        }

        // ✅ Download securely
        return Storage::download($path);
    }
}
```

#### Example Route
```php
Route::get('/bills/download/{filename}', [BillController::class, 'download'])
     ->middleware('auth');
```

---

### 3. (Optional) Encrypt Before Storing

If files are **highly confidential** (e.g., medical records, financial statements), you may encrypt them before storage:

```php
use Illuminate\Support\Facades\Crypt;

// Encrypt before saving
$content = Crypt::encrypt(file_get_contents($uploadedFile));
Storage::put('private/bills/invoice_123.enc', $content);

// Decrypt when retrieving
$decrypted = Crypt::decrypt(Storage::get('private/bills/invoice_123.enc'));
```

---

### 4. Cloud Storage Use Case (S3, etc.)

If you use Amazon S3, DigitalOcean Spaces, or another cloud provider:

- Configure a **private bucket** (not public).  
- Generate **temporary signed URLs** only for authorized users.  

Example (S3):
```php
$url = Storage::disk('s3')->temporaryUrl(
    'bills/invoice_123.pdf',
    now()->addMinutes(5)
);
```
The file will be available only for **5 minutes**, then the URL expires.

---

## 💡 Use Cases

### 🔹 Use Case 1: Billing System
- Store customer invoices in `storage/app/private/bills`.
- Allow only logged-in users to download their own invoices.
- Admins can download all bills.

### 🔹 Use Case 2: HR/Payroll System
- Employee contracts, salary slips, and ID proofs stored in `storage/app/private/hr`.
- Access restricted by roles (employee, HR admin, finance team).

### 🔹 Use Case 3: Legal Documents
- Contracts stored encrypted.
- Only the legal department has access.
- Access logs maintained for compliance.

---

## ⚡ Best Practices

1. **Never** store sensitive files in `public/` or `storage/app/public`.
2. Use **authorization checks** (`Gate`, `Policy`, or roles) before serving files.
3. Enable **encryption** for highly sensitive data.
4. For cloud storage, use **private buckets** + **signed URLs**.
5. Consider **audit logging** (who downloaded which file, when).
6. Set strict **file naming conventions** (e.g., `invoice_{userId}_{date}.pdf`).

---

## 🛡️ Summary

- Store sensitive files in **non-public storage** (`storage/app/private`).  
- Serve them **through Laravel controllers**, not direct links.  
- Add **authorization checks** and optionally **encryption**.  
- For cloud storage, use **private disks with signed URLs**.  

This approach ensures that **bills, contracts, and sensitive documents remain secure** and are only accessible to authorized users.  