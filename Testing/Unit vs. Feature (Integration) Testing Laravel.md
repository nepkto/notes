# Unit vs. Feature (Integration) Testing in Laravel

Testing is a cornerstone of robust software development. In Laravel, two primary types of automated tests help ensure application quality: **unit tests** and **feature (integration) tests**. This guide explores their differences, use cases, and best practices, with enhanced examples and actionable tips for modern Laravel projects.

---

## Comparison Table: Unit Testing vs. Feature Testing

| Aspect       | Unit Testing                                          | Feature Testing                                      |
|--------------|------------------------------------------------------|------------------------------------------------------|
| **Focus**    | Individual methods or classes                        | Entire features or user flows                        |
| **Isolation**| Isolates code using mocks/stubs                      | Minimal mocking; interacts with the actual framework |
| **Speed**    | Very fast (no external dependencies)                 | Slower (integrates with DB, HTTP, etc.)              |
| **Complexity**| Simpler, focused tests                              | More complex, covers app integration                 |
| **Purpose**  | Verify logic and edge cases                          | Verify end-to-end behavior and integration           |
| **Best For** | Services, models, helpers                            | HTTP endpoints, user scenarios                       |

---

## Why Use Both?

A balanced Laravel test suite leverages both unit and feature tests:

- **Unit tests** validate the core logic in services, models, or utilities.
- **Feature tests** ensure all parts of your application work together as expected from the end user's perspective.

**Tip:** Use unit tests for business logic and feature tests for user-facing flows and API endpoints.

---

## Unit Testing

### What is Unit Testing?

- **Scope**: Tests a single "unit" (method or class) in isolation.
- **Isolation**: Uses mocking or stubbing for dependencies.
- **Speed**: Runs quickly (no database, file system, or HTTP).

### When to Use Unit Tests

- Testing business logic in services, models, or helpers.
- Verifying method outputs for various inputs.
- Handling edge cases, exceptions, and error conditions.

### Laravel Example: Discount Service

**Service Class: `app/Services/DiscountService.php`**
```php
<?php

namespace App\Services;

class DiscountService
{
    public function calculateDiscount(float $amount, float $percentage): float
    {
        return round($amount * ($percentage / 100), 2);
    }
}
```

**Unit Test: `tests/Unit/DiscountServiceTest.php`**
```php
<?php

namespace Tests\Unit;

use Tests\TestCase;
use App\Services\DiscountService;

class DiscountServiceTest extends TestCase
{
    /** @test */
    public function it_calculates_the_discount_correctly()
    {
        $service = new DiscountService();
        $discount = $service->calculateDiscount(200, 15);
        $this->assertEquals(30.0, $discount);
    }
}
```

**Key Points:**
- Direct instantiation of the service (no external dependencies).
- Focus on one method and its expected behavior.

---

## Feature (Integration) Testing

### What is Feature Testing?

- **Scope**: Validates interactions between multiple components.
- **Environment**: Uses HTTP requests, database, routes, middleware, etc.
- **Scenario**: Simulates real user actions (e.g., submitting a form, API requests).

### When to Use Feature Tests

- Testing HTTP endpoints and API responses.
- Validating middleware, routes, and controllers together.
- Ensuring layers (controllers, services, models) integrate properly.

### Laravel Example: Task Creation Endpoint

**Controller Action: `app/Http/Controllers/TaskController.php`**
```php
<?php

namespace App\Http\Controllers;

use App\Models\Task;
use Illuminate\Http\Request;

class TaskController extends Controller
{
    public function store(Request $request)
    {
        $data = $request->validate([
            'title'       => 'required|string|max:255',
            'description' => 'nullable|string',
            'status'      => 'required|in:pending,in_progress,completed',
            'priority'    => 'required|in:low,medium,high',
            'due_date'    => 'required|date',
            'assigned_to' => 'required|exists:users,id',
        ]);
        $task = Task::create($data);
        return response()->json($task, 201);
    }
}
```

**Feature Test: `tests/Feature/TaskCreationTest.php`**
```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class TaskCreationTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function a_valid_request_creates_a_task()
    {
        $user = User::factory()->create();
        $this->actingAs($user);

        $teamMember = User::factory()->create();

        $data = [
            'title'       => 'Complete Documentation',
            'description' => 'Write unit and feature tests for the application.',
            'status'      => 'pending',
            'priority'    => 'high',
            'due_date'    => now()->addWeek()->toDateString(),
            'assigned_to' => $teamMember->id,
        ];

        $response = $this->postJson('/api/tasks', $data);

        $response->assertStatus(201);
        $this->assertDatabaseHas('tasks', ['title' => 'Complete Documentation']);
    }
}
```

**Key Points:**
- Simulates a real HTTP request (route, middleware, validation, controller, DB).
- Tests the full flow from request to response.

---

## What Is Mocking?

**Mocking** replaces real objects or dependencies with "fake" objects to isolate the code under test. It’s critical for unit tests—especially when dealing with external APIs, slow resources, or complex services.

### Key Concepts

- **Simulated Objects**: Mimic real dependencies in a controlled way.
- **Expectations**: Define how mocks should be used (methods, parameters, calls).
- **Controlled Responses**: Set mocks to return values or throw exceptions.
- **Isolation**: Prevent tests from relying on external systems or side effects.

### When to Use Mocking

1. **External Services/APIs**: Payment gateways, message services, etc.
2. **Database Calls/File Systems**: Avoid real DB in unit tests; use in-memory DB for feature tests.
3. **Complex Dependencies**: Email, logging, caching.
4. **Controlling Side Effects**: Simulate errors or edge cases.

### Laravel/PHPUnit Example: Mocking an External Service

**Scenario**: Test a `NotificationService` that sends emails via an external provider.

**Service Using a Dependency**
```php
<?php

namespace App\Services;

use App\Services\ExternalNotifier;

class NotificationService
{
    protected $notifier;

    public function __construct(ExternalNotifier $notifier)
    {
        $this->notifier = $notifier;
    }

    public function sendWelcomeNotification(string $email): bool
    {
        $message = "Welcome to our application!";
        return $this->notifier->send($email, $message);
    }
}
```

**External Notifier**
```php
<?php

namespace App\Services;

class ExternalNotifier
{
    public function send(string $email, string $message): bool
    {
        // Actual code to send notification (e.g., via API)
    }
}
```

**Unit Test with Mock**
```php
<?php

namespace Tests\Unit;

use Tests\TestCase;
use App\Services\NotificationService;
use App\Services\ExternalNotifier;
use Mockery;

class NotificationServiceTest extends TestCase
{
    public function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }

    /** @test */
    public function it_sends_a_welcome_notification()
    {
        $notifierMock = Mockery::mock(ExternalNotifier::class);
        $notifierMock->shouldReceive('send')
            ->once()
            ->with('user@example.com', 'Welcome to our application!')
            ->andReturn(true);

        $service = new NotificationService($notifierMock);

        $result = $service->sendWelcomeNotification('user@example.com');

        $this->assertTrue($result);
    }
}
```

**Key Points:**
- Create a mock for `ExternalNotifier`.
- Set expectations (method called, parameters, response).
- Inject mock into `NotificationService` for isolation.

---

## When **Not** to Use Mocks

- **Over-Mocking**: Excess mocks can make tests fragile and too closely tied to implementation.
- **Feature/Integration Tests**: Use real components to validate actual interactions.

---

## Summary & Best Practices

- **Unit Tests**: Isolate and verify logic; use mocks for dependencies.
- **Feature Tests**: Simulate real user scenarios; test actual integration.
- **Mocking**: Valuable for unit tests, but limit use in integration tests.
- **Test Naming**: Use descriptive method names (e.g., `it_calculates_the_discount_correctly`).
- **Test Coverage**: Aim for high coverage but prioritize critical paths and business logic.
- **Continuous Integration**: Run tests in CI to catch regressions early.
- **Use Factories**: Laravel model factories simplify test data creation.
- **Database Refresh**: Use `RefreshDatabase` trait for clean state in feature tests.

---

## Further Enhancements

- **Parameterization**: Use data providers to test multiple cases in unit tests.
- **Custom Assertions**: Simplify complex checks with custom assertions.
- **Trait Usage**: Share common setup logic via traits.
- **Test Organization**: Group tests by domain (e.g., billing, user management).
- **Code Coverage Tools**: Leverage tools like Xdebug or PHPStan for coverage reports.
- **Performance**: Keep unit tests fast; use feature tests sparingly for slow paths.
- **Documentation**: Document test intentions and scenarios for maintainability.

---

## Resources

- [Laravel Testing Documentation](https://laravel.com/docs/testing)
- [PHPUnit Documentation](https://phpunit.de/documentation.html)
- [Mockery Documentation](https://docs.mockery.io/)

---

**Invest in both unit and feature tests for resilient, maintainable Laravel applications. Happy testing!**