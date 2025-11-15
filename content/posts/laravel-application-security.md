---
title: "Best Practices for Laravel Application Security"
date: 2024-11-05
draft: false
description: "Comprehensive guide to securing your Laravel applications. Learn authentication, authorization, CSRF protection, SQL injection prevention, and more."
tags: ["Laravel", "Security", "Best Practices", "Web Development"]
categories: ["Web Development"]
---

## Best Practices for Laravel Application Security

Security is not optional—it's essential. This comprehensive guide covers everything you need to know to build secure Laravel applications and protect your users' data.

---

## 1. Authentication & Authorization

### Secure Password Handling

```php
// ✅ Always hash passwords
use Illuminate\Support\Facades\Hash;

class UserController extends Controller
{
    public function store(Request $request)
    {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password), // Bcrypt by default
        ]);

        return response()->json($user, 201);
    }

    public function checkPassword(Request $request)
    {
        $user = User::find($request->user_id);
        
        if (Hash::check($request->password, $user->password)) {
            // Password matches
        }
    }
}

// config/hashing.php - Configure bcrypt rounds
'bcrypt' => [
    'rounds' => env('BCRYPT_ROUNDS', 12), // Higher is more secure but slower
],
```

### Multi-Factor Authentication (MFA)

```php
use Laravel\Fortify\Features;

// fortify config
'features' => [
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
],

// Custom MFA implementation
class MfaService
{
    public function generateSecret(): string
    {
        return Google2FA::generateSecretKey();
    }

    public function verifyCode(User $user, string $code): bool
    {
        return Google2FA::verifyKey($user->two_factor_secret, $code);
    }

    public function generateQrCode(User $user): string
    {
        return Google2FA::getQRCodeInline(
            config('app.name'),
            $user->email,
            $user->two_factor_secret
        );
    }
}
```

### Role-Based Access Control (RBAC)

```php
// Using Laravel's Gates
// AuthServiceProvider
public function boot(): void
{
    Gate::define('delete-post', function (User $user, Post $post) {
        return $user->id === $post->user_id || $user->isAdmin();
    });

    Gate::define('manage-users', function (User $user) {
        return $user->hasRole('admin');
    });
}

// Controller
public function destroy(Post $post)
{
    $this->authorize('delete-post', $post);
    
    $post->delete();
    
    return response()->json(['message' => 'Post deleted']);
}

// Using Policies
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}

// Register in AuthServiceProvider
protected $policies = [
    Post::class => PostPolicy::class,
];
```

---

## 2. Protection Against Common Attacks

### SQL Injection Prevention

```php
// ❌ Never do this - SQL Injection vulnerable
$email = request('email');
$users = DB::select("SELECT * FROM users WHERE email = '{$email}'");

// ✅ Always use parameter binding
$email = request('email');
$users = DB::select('SELECT * FROM users WHERE email = ?', [$email]);

// ✅ Better: Use Eloquent ORM (automatically prevents SQL injection)
$users = User::where('email', request('email'))->get();

// ✅ Use Query Builder
$users = DB::table('users')
    ->where('email', request('email'))
    ->get();
```

### XSS (Cross-Site Scripting) Prevention

```php
// Blade automatically escapes output
// ✅ Escaped by default
<div>{{ $userInput }}</div> 

// ❌ Raw output - only for trusted content
<div>{!! $trustedHtml !!}</div>

// ✅ Use HTML Purifier for user-generated HTML
use HTMLPurifier;

class CommentService
{
    public function sanitizeComment(string $comment): string
    {
        $config = HTMLPurifier_Config::createDefault();
        $config->set('HTML.Allowed', 'p,b,i,strong,em,a[href],ul,ol,li');
        
        $purifier = new HTMLPurifier($config);
        return $purifier->purify($comment);
    }
}

// Controller
public function store(Request $request)
{
    $comment = Comment::create([
        'content' => app(CommentService::class)
            ->sanitizeComment($request->content),
    ]);

    return response()->json($comment);
}
```

### CSRF Protection

```php
// Laravel automatically protects POST, PUT, PATCH, DELETE requests
// ✅ Always include CSRF token in forms
<form method="POST" action="/profile">
    @csrf
    <!-- form fields -->
</form>

// ✅ For AJAX requests
<meta name="csrf-token" content="{{ csrf_token() }}">

<script>
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});

// Or with Axios
axios.defaults.headers.common['X-CSRF-TOKEN'] = 
    document.querySelector('meta[name="csrf-token"]').content;
</script>

// Exclude specific routes (use carefully!)
// VerifyCsrfToken middleware
protected $except = [
    'webhook/payment', // External webhooks
];
```

### Mass Assignment Protection

```php
// Model
class User extends Model
{
    // ✅ Whitelist approach (recommended)
    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    // Or blacklist approach
    protected $guarded = [
        'id',
        'is_admin',
        'remember_token',
    ];
}

// ❌ Never do this
User::create(request()->all()); // Dangerous!

// ✅ Always validate and specify fields
User::create($request->validated());

// Or be explicit
User::create([
    'name' => $request->name,
    'email' => $request->email,
    'password' => Hash::make($request->password),
]);
```

---

## 3. API Security

### API Authentication with Sanctum

```php
// Setup Sanctum
// User model
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// Login endpoint
class AuthController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        // Create token with abilities
        $token = $user->createToken('auth-token', [
            'user:read',
            'user:write',
        ])->plainTextToken;

        return response()->json([
            'token' => $token,
            'user' => $user,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();
        
        return response()->json(['message' => 'Logged out']);
    }
}

// Protected routes
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    });
});
```

### Rate Limiting

```php
// RouteServiceProvider
protected function configureRateLimiting()
{
    // Basic rate limiting
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)
            ->by($request->user()?->id ?: $request->ip());
    });

    // Login rate limiting
    RateLimiter::for('login', function (Request $request) {
        return Limit::perMinute(5)
            ->by($request->email . $request->ip())
            ->response(function () {
                return response()->json([
                    'message' => 'Too many login attempts.'
                ], 429);
            });
    });

    // Different limits based on user role
    RateLimiter::for('uploads', function (Request $request) {
        if ($request->user()?->isPremium()) {
            return Limit::perHour(1000);
        }
        
        return Limit::perHour(100);
    });
}

// Apply to routes
Route::middleware(['auth:sanctum', 'throttle:api'])
    ->get('/data', [DataController::class, 'index']);

Route::middleware('throttle:login')
    ->post('/login', [AuthController::class, 'login']);
```

### Input Validation & Sanitization

```php
class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255', 'min:3'],
            'content' => ['required', 'string', 'max:10000'],
            'category_id' => ['required', 'exists:categories,id'],
            'tags' => ['nullable', 'array', 'max:5'],
            'tags.*' => ['string', 'max:50'],
            'published_at' => ['nullable', 'date', 'after:now'],
            'image' => ['nullable', 'image', 'mimes:jpg,png,webp', 'max:2048'],
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => 'Post title is required',
            'content.max' => 'Content cannot exceed 10,000 characters',
        ];
    }

    // Custom validation
    public function withValidator($validator)
    {
        $validator->after(function ($validator) {
            if ($this->hasProfanity($this->title)) {
                $validator->errors()->add('title', 'Inappropriate content detected');
            }
        });
    }

    // Sanitize input
    protected function prepareForValidation(): void
    {
        $this->merge([
            'title' => strip_tags($this->title),
            'slug' => Str::slug($this->title),
        ]);
    }
}
```

---

## 4. Secure File Uploads

```php
class FileUploadService
{
    public function uploadFile(UploadedFile $file, string $disk = 'private'): string
    {
        // Validate file type
        $allowedMimes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
        
        if (!in_array($file->getMimeType(), $allowedMimes)) {
            throw new \InvalidArgumentException('Invalid file type');
        }

        // Validate file size (5MB max)
        if ($file->getSize() > 5 * 1024 * 1024) {
            throw new \InvalidArgumentException('File too large');
        }

        // Generate secure filename
        $filename = Str::uuid() . '.' . $file->getClientOriginalExtension();
        
        // Store file
        $path = $file->storeAs('uploads', $filename, $disk);

        // Scan for malware (if antivirus available)
        // $this->scanFile($path);

        return $path;
    }

    public function getSignedUrl(string $path): string
    {
        // Generate temporary signed URL
        return Storage::temporaryUrl(
            $path,
            now()->addMinutes(30)
        );
    }
}

// Controller
public function upload(Request $request)
{
    $request->validate([
        'file' => 'required|file|mimes:jpg,png,pdf|max:5120',
    ]);

    $path = app(FileUploadService::class)
        ->uploadFile($request->file('file'));

    return response()->json(['path' => $path]);
}

// Download with authorization
public function download(Request $request, string $filename)
{
    $this->authorize('download-file', $filename);

    if (!Storage::disk('private')->exists($filename)) {
        abort(404);
    }

    return Storage::disk('private')->download($filename);
}
```

---

## 5. Secure Configuration

### Environment Variables

```php
// ❌ Never commit .env file
// ❌ Never hardcode sensitive data

// ✅ Use environment variables
'database' => [
    'password' => env('DB_PASSWORD'),
],
'services' => [
    'stripe' => [
        'secret' => env('STRIPE_SECRET'),
    ],
],

// ✅ Validate required env variables
// AppServiceProvider
public function boot(): void
{
    $required = [
        'APP_KEY',
        'DB_PASSWORD',
        'MAIL_PASSWORD',
    ];

    foreach ($required as $var) {
        if (!env($var)) {
            throw new \RuntimeException("Missing required environment variable: {$var}");
        }
    }
}
```

### Security Headers

```php
// Middleware
class SecurityHeaders
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        return $response
            ->header('X-Content-Type-Options', 'nosniff')
            ->header('X-Frame-Options', 'SAMEORIGIN')
            ->header('X-XSS-Protection', '1; mode=block')
            ->header('Strict-Transport-Security', 'max-age=31536000; includeSubDomains')
            ->header('Content-Security-Policy', "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'")
            ->header('Referrer-Policy', 'strict-origin-when-cross-origin')
            ->header('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
    }
}

// Register in Kernel
protected $middleware = [
    \App\Http\Middleware\SecurityHeaders::class,
];
```

---

## 6. Logging & Monitoring

```php
// Log security events
class SecurityLogger
{
    public function logFailedLogin(string $email, string $ip): void
    {
        Log::channel('security')->warning('Failed login attempt', [
            'email' => $email,
            'ip' => $ip,
            'user_agent' => request()->userAgent(),
            'timestamp' => now(),
        ]);
    }

    public function logSuspiciousActivity(User $user, string $action): void
    {
        Log::channel('security')->alert('Suspicious activity detected', [
            'user_id' => $user->id,
            'action' => $action,
            'ip' => request()->ip(),
            'timestamp' => now(),
        ]);

        // Send notification to admin
        Notification::route('slack', config('logging.slack.url'))
            ->notify(new SuspiciousActivityDetected($user, $action));
    }
}

// Monitor authentication attempts
// EventServiceProvider
protected $listen = [
    \Illuminate\Auth\Events\Failed::class => [
        \App\Listeners\LogFailedLogin::class,
    ],
    \Illuminate\Auth\Events\Lockout::class => [
        \App\Listeners\NotifyAccountLockout::class,
    ],
];
```

---

## 7. Database Security

```php
// Use encrypted fields for sensitive data
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    protected $casts = [
        'social_security' => 'encrypted',
        'credit_card' => 'encrypted',
    ];

    // Or custom encryption
    protected function socialSecurity(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => decrypt($value),
            set: fn ($value) => encrypt($value),
        );
    }
}

// Backup encryption
'connections' => [
    'mysql' => [
        'options' => [
            PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_SSL_CA'),
            PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT => true,
        ],
    ],
],
```

---

## Security Checklist

- ✅ Use HTTPS everywhere (SSL/TLS)
- ✅ Keep Laravel and dependencies updated
- ✅ Hash all passwords with bcrypt
- ✅ Implement MFA for sensitive accounts
- ✅ Validate and sanitize all user input
- ✅ Protect against SQL injection with ORM
- ✅ Enable CSRF protection
- ✅ Implement rate limiting
- ✅ Use Content Security Policy headers
- ✅ Secure file uploads
- ✅ Never commit .env files
- ✅ Use signed URLs for private files
- ✅ Log security events
- ✅ Implement proper RBAC
- ✅ Encrypt sensitive database fields
- ✅ Regular security audits
- ✅ Keep backups encrypted

---

## Conclusion

Security is an ongoing process, not a one-time task. Regularly audit your code, keep dependencies updated, and stay informed about new vulnerabilities. Use Laravel's built-in security features—they're battle-tested and robust.

**Remember: The cost of preventing a breach is always less than recovering from one.**
