# Backend rules

Server-side patterns: Laravel, PHP frameworks, APIs, authentication, databases. Applies whenever the task touches backend / API code.

---

### Laravel custom middleware needs explicit priority to run before `Authenticate`
**When:** Laravel + adding custom middleware that must run before the framework's `Authenticate` middleware (e.g., auto-login via URL token, request token swap, feature-flagged auth)
**Do:** Add both the custom middleware and `\Illuminate\Auth\Middleware\Authenticate::class` to `$middleware->priority([...])` in `bootstrap/app.php`, with the custom one listed first:
```php
$middleware->priority([
    HandleCheckoutToken::class,
    \Illuminate\Auth\Middleware\Authenticate::class,
]);
```
**Don't:** Rely on route-level ordering (`->middleware(['custom', 'auth'])`) — Laravel's priority list overrides that, and `Authenticate` wins by default.
**Why:** Laravel sorts middleware by a built-in priority list. Middleware not in the list gets bucketed after prioritized entries — so `Authenticate` runs first regardless of declaration order on the route.

---

### Laravel Sanctum v4.3 has no `isExpired()` — check `expires_at->isPast()` manually
**When:** Laravel Sanctum v4.3 + checking whether a `PersonalAccessToken` is expired
**Do:** Use the manual null-safe check:
```php
$isExpired = $accessToken->expires_at && $accessToken->expires_at->isPast();
```
**Don't:** Call `$accessToken->isExpired()` — throws `BadMethodCallException`, the method does not exist in v4.3.x.
**Why:** `isExpired()` was never added to `PersonalAccessToken` in Sanctum v4.3. `expires_at` is a nullable Carbon instance on the model, and `isPast()` is the standard Carbon idiom.
