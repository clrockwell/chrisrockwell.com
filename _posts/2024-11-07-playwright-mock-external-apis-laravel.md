---
layout: post
title: Mock External APIs in Laravel for E2E Testing with Playwright
---

> *Note:* This post is written for a Laravel 9 application. If you're using Laravel 11.x or higher, some things may have changed — be sure to check the latest documentation.

We’re lucky to have a wide array of testing tools available to us these days. When it comes to UI testing, my go-to is usually [Playwright](https://playwright.dev/), a powerful end-to-end testing framework. Recently, though, I found myself needing to mock an external API call during testing. Laravel’s [mocking capabilities](https://laravel.com/docs/9.x/mocking) make it easy to mock external services in feature tests, and Playwright's own [mocking functionality](https://playwright.dev/docs/mock) is similarly user-friendly for mocking browser-side API calls.

But what happens when your controller makes an external API call? That’s where things get tricky. I landed on two possible solutions:

1. **Add custom routes** to intercept API calls and route them to your application during testing.
2. **Mock your Service** by creating a separate implementation and swapping it in the service container during tests.

For my project, I opted for the first solution because it was straightforward, and I don’t anticipate needing it often. Let's dive into the details!

---
### Step 1: Set Up the Service

Let's assume you have a service provider that’s responsible for communicating with a 3rd-party API. Here’s an example:

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class ExternalApiServiceProvider extends ServiceProvider implements IExternalApiServiceProvider
{
    protected string $api_base;

    public function __construct($app, $api_base)
    {
        $this->api_base = $api_base;
        parent::__construct($app);
    }

    public function getData() : string
    {
        $response = Http::get($this->api_base . '/api/v2/external-endpoint');

        if ($response->successful()) {
            // Shoutout to https://github.com/cweiske/jsonmapper ;)
            return $response->body();
        }

        return null;
    }
}
```


In typical end-to-end (e2e) testing, you want to test everything from start to finish. But when you have external dependencies like APIs, things can get complicated. It’s essential to strike a balance between testing thoroughly and not relying on real external services.

> **Tip**: If possible, refactor to use unit tests instead. But when testing complex, interconnected UIs, this isn’t always feasible!

In this case, I decided that continuing with e2e testing was the right choice, but I needed a way to mock some of these external API calls. While Playwright can easily mock browser-side HTTP requests, it doesn’t offer a built-in solution for backend API calls. No problem — we’ll intercept these requests using a simple route trick.

---
### Step 2: Add Routes for Mocking External API Calls
We’ll create a new service provider that adds routes to intercept and mock the external API calls.

`root/tests/Playwright/PlaywrightHelper/PlaywrightExternalMockServiceProvider.php`
```php

namespace Tests\Playwright\PlaywrightHelper;

use Illuminate\Support\Facades\Route;
use Illuminate\Support\ServiceProvider;

class PlaywrightExternalMockServiceProvider extends ServiceProvider
{
    public function boot()
    {
        if ($this->app->environment('production')) {
            return;
        }

        $this->addRoutes();
    }

    protected function addRoutes()
    {
        Route::namespace('')
            ->middleware('web')
            ->prefix('__playwright_testing__')
            ->group(function() {
                // Add new route files here to override specific external calls.
                include_once __DIR__ . '/routes/external-api-service-provider.php';
            });
    }
}
```

This provider essentially loads a set of routes from an external PHP file that you can use to mock specific external API calls. Here's an example route file:

`root/tests/Playwright/PlaywrightHelper/routes/external-api-service-provider.php`

```php

use Illuminate\Support\Facades\Request;
use Illuminate\Support\Facades\Response;
use Illuminate\Support\Facades\Route;


Route::get('/api/v2/external-endpoint', function() {
    $request = Request::capture();
    $data = ['We did it!']

    return Response::json($data, 200);
});

```

---
### Step 3: Update Your `.env` to Use Local Routes

To ensure your tests use the mocked route, all you need to do is adjust the `EXTERNAL_API` configuration in your `.env` or `.env.testing` file:

**.env[.testing]**
```
EXTERNAL_API=http://localhost:8080/__playwright_testing__
```

Now, whenever you run your tests, any calls to the external API will hit `http://localhost:8080/__playwright_testing__/api/v2/external-endpoint`, giving you full control over the mock response.

---
### Wrapping Up
By routing external API calls to local mock routes, you can keep your e2e tests running smoothly without depending on external services. This approach works well for cases where you want to control the responses and ensure your tests are both fast and reliable.

Happy testing! 🚀
