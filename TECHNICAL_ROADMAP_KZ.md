# Жабық видеокурс платформасы — Техникалық ТЗ + Developer Roadmap

## 1) Толық архитектура (high-level)
- **Backend**: Laravel 12 (PHP 8.3), MVC + Service Layer + Policy-based authorization.
- **Admin**: Filament v3 panel (`/admin`).
- **User Front**: Blade + Vite + Alpine.js (жеңіл, SSR-friendly).
- **DB**: MySQL 8 / MariaDB 10.6+.
- **Cache/Session/Queue**: бастапқыда `database` драйверлері (әлсіз хостингке тұрақты), кейін Redis optional.
- **Storage**: local `public` disk (тек суреттер/логотип), видео файл сақталмайды.
- **Video**: YouTube embed privacy mode (`youtube-nocookie.com`) + қатаң UI-overlay және sandbox/CSP шектеулері.
- **Security**: HTTPS-only, Cloudflare proxy/WAF, rate-limit, signed routes (қажет жерде), audit logs.

## 2) Laravel папка құрылымы
```text
app/
  Actions/
    CourseRequest/ApproveCourseRequestAction.php
    CourseRequest/RejectCourseRequestAction.php
    CourseRequest/BlockEnrollmentAction.php
  Enums/
    CourseStatus.php
    LessonStatus.php
    RequestStatus.php
    Role.php
  Http/
    Controllers/
      Web/
        CourseController.php
        LessonController.php
        DashboardController.php
        ProfileController.php
    Middleware/
      EnsureUserNotBlocked.php
      EnsureCourseAccess.php
      SecurityHeaders.php
  Models/
    User.php Course.php Lesson.php CourseRequest.php Enrollment.php LessonProgress.php SiteSetting.php ActivityLog.php
  Policies/
    CoursePolicy.php LessonPolicy.php EnrollmentPolicy.php
  Services/
    Youtube/
      YoutubeUrlParser.php
    Progress/
      CourseProgressService.php
    Security/
      ActivityLogger.php
      AccessGuardService.php
  Filament/
    Resources/
      CourseResource.php LessonResource.php UserResource.php CourseRequestResource.php EnrollmentResource.php SiteSettingResource.php ActivityLogResource.php
routes/
  web.php
  auth.php
database/
  migrations/
  seeders/
resources/views/
  courses/ lessons/ profile/ components/
```

## 3) Дерекқор схемасы
- `users`: id, name, email(unique), password, role(enum:admin,user), is_blocked(bool), blocked_at, email_verified_at, last_login_at, timestamps.
- `courses`: id, title, slug(unique), description(text), cover_path(nullable), status(enum:active,hidden), is_published(bool), sort_order, created_by, timestamps, softDeletes.
- `lessons`: id, course_id(fk), title, description, youtube_url, youtube_video_id(index), sort_order, status(enum:active,hidden), duration_seconds(nullable), timestamps, softDeletes.
- `course_requests`: id, user_id, course_id, status(enum pending/approved/rejected/removed/blocked), reviewed_by(nullable), reviewed_at(nullable), note(nullable), unique(user_id,course_id), timestamps.
- `enrollments`: id, user_id, course_id, status(enum approved/removed/blocked), approved_by, approved_at, removed_at(nullable), blocked_at(nullable), unique(user_id,course_id), timestamps.
- `lesson_progress`: id, user_id, course_id, lesson_id, is_completed(bool), completed_at(nullable), watched_seconds(default 0), unique(user_id,lesson_id), timestamps.
- `site_settings`: id, site_name, logo_path(nullable), logo_dark_path(nullable), default_course_placeholder, maintenance_notice(nullable), timestamps.
- `activity_logs`: id, actor_id(nullable), actor_role, action, subject_type, subject_id, ip, user_agent, meta(json), created_at(index).

## 4) Migration құрылымы (реті)
1. users
2. courses
3. lessons
4. course_requests
5. enrollments
6. lesson_progress
7. site_settings
8. activity_logs
9. add_indexes_foreign_keys

## 5) Model байланыстары
- User:
  - hasMany(CourseRequest)
  - belongsToMany(Course, 'enrollments')->withPivot(status,...)
  - hasMany(LessonProgress)
- Course:
  - hasMany(Lesson)->orderBy(sort_order)
  - hasMany(CourseRequest)
  - hasMany(Enrollment)
- Lesson:
  - belongsTo(Course)
  - hasMany(LessonProgress)
- Enrollment:
  - belongsTo(User), belongsTo(Course)
- LessonProgress:
  - belongsTo(User), belongsTo(Course), belongsTo(Lesson)

## 6) Filament Resource тізімі
1. Dashboard Widgets (stats + latest)
2. CourseResource
3. LessonResource (course relation manager)
4. UserResource
5. CourseRequestResource
6. EnrollmentResource
7. SiteSettingResource
8. ActivityLogResource (read-only)

## 7) Route құрылымы
```php
Route::middleware(['auth','verified','not_blocked'])->group(function () {
    Route::get('/dashboard', DashboardController::class)->name('dashboard');

    Route::get('/courses', [CourseController::class,'index'])->name('courses.index');
    Route::post('/courses/{course}/request', [CourseController::class,'requestAccess'])->name('courses.request');
    Route::get('/my-courses', [CourseController::class,'myCourses'])->name('courses.my');

    Route::middleware(['can:view,course','course.access'])->group(function () {
        Route::get('/courses/{course}', [CourseController::class,'show'])->name('courses.show');
        Route::get('/courses/{course}/lessons/{lesson}', [LessonController::class,'show'])->name('lessons.show');
        Route::post('/lessons/{lesson}/complete', [LessonController::class,'complete'])->name('lessons.complete');
    });

    Route::get('/profile', [ProfileController::class,'edit'])->name('profile.edit');
    Route::patch('/profile/password', [ProfileController::class,'password'])->name('profile.password');
});
```

## 8) Middleware және Policy
- Middleware:
  - `EnsureUserNotBlocked`
  - `EnsureCourseAccess` (enrollment approved + not removed/blocked)
  - `SecurityHeaders` (CSP, X-Frame-Options, Referrer-Policy)
- Policy:
  - CoursePolicy: viewAny/view/create/update/delete/requestAccess
  - LessonPolicy: view/create/update/delete/reorder
  - EnrollmentPolicy: approve/reject/remove/block

## 9) Authorization & рөл логикасы
- `users.role = admin|user`.
- Gate::before: admin толық рұқсат.
- User тек:
  - active курстарды көру;
  - тек own approved enrollment бар курсқа кіру.
- Blocked user: login болуы мүмкін, бірақ course area deny (немесе толық logout стратегиясы).

## 10) YouTube Video ID парсер
```php
final class YoutubeUrlParser
{
    public function parse(string $url): ?string
    {
        $patterns = [
            '/youtube\.com\/watch\?v=([\w-]{11})/i',
            '/youtu\.be\/([\w-]{11})/i',
            '/youtube\.com\/embed\/([\w-]{11})/i',
        ];

        foreach ($patterns as $p) {
            if (preg_match($p, $url, $m)) {
                return $m[1];
            }
        }

        return null;
    }
}
```
- Validation: parse null болса — reject.
- DB-де `youtube_video_id` бөлек сақталады.

## 11) Видео беті қауіпсіздігі (маңызды)
### YouTube ресми шектеуі туралы ашық ескерту
- YouTube embed-ті толық «брендсіз және сыртқа шықпайтын» ету **100% мүмкін емес** (ToS/плеер шектеулері).
- Бірақ бизнес талапқа **ең жақын** шешім:
  1) `youtube-nocookie.com/embed/{id}`;
  2) `controls=1, modestbranding=1, rel=0, iv_load_policy=3, fs=0, disablekb=1`;
  3) iframe үстіне custom overlay/interaction guard;
  4) context menu disable + `pointer-events` режимін басқару;
  5) CSP тек YouTube embed домендерін allow;
  6) сыртқы link ашылса `target=_self` intercept + confirm modal.

```html
<iframe
  src="https://www.youtube-nocookie.com/embed/{{ $videoId }}?rel=0&modestbranding=1&iv_load_policy=3&fs=0"
  referrerpolicy="strict-origin-when-cross-origin"
  allow="autoplay; encrypted-media; picture-in-picture"
  sandbox="allow-scripts allow-same-origin allow-presentation"
></iframe>
```
- Direct lesson URL қорғауы: auth + course.access middleware + policy.

## 12) Қолданушыны курсқа қабылдау алгоритмі
1. User request жасайды (`pending`).
2. Admin approve етсе:
   - `course_requests.status=approved`
   - `enrollments` upsert(status=approved, approved_at).
3. Reject: `course_requests.status=rejected`, enrollment жасалмайды.
4. Remove: enrollment `removed`, request `removed`.
5. Block: enrollment `blocked`, request `blocked`, user-course deny.

## 13) Прогресс есептеу
- Lesson complete endpoint `lesson_progress.is_completed=true`.
- Formula:
  - `total = active lessons count`
  - `done = completed active lessons count`
  - `percent = total>0 ? round(done/total*100) : 0`
- Курс карточкасында кештелген percent (query with subselect).

## 14) Admin UI
- Dashboard KPI cards + 2 tables (latest users/requests).
- Курстар: CRUD, cover upload, status toggle, lessons relation manager.
- Requests: filters by status, bulk approve/reject.
- Enrollments: approved/removed/blocked tabs.
- Settings: site_name/logo.
- Logs: read-only, searchable by actor/action/date.

## 15) User cabinet UI
- Менің курстарым (approved)
- Қолжетімді курстар (active + requested емес)
- Күтіп тұрған сұраныстар
- Прогресс бары (per course)
- Profile + password update

## 16) Файл жүктеу қауіпсіздігі
- Validation:
  - cover: `image|mimes:jpg,jpeg,png,webp|max:2048`
  - logo: `mimes:jpg,jpeg,png,webp,svg|max:2048`
- Image optimization: Intervention Image (resize + quality).
- Randomized filename + mime sniffing + SVG sanitize (if allowed).

## 17) Cloudflare + SSL
- Full (strict) SSL mode.
- Always Use HTTPS, HSTS.
- WAF managed rules ON.
- Rate limit `/login`, `/register`, `/password/*`.
- Bot Fight Mode moderate.

## 18) Әлсіз хостингке оптимизация
- Nginx + PHP-FPM tuned (pm=ondemand).
- OPcache enabled.
- `php artisan config:cache route:cache view:cache`.
- DB индекстер міндетті (status, foreign keys, composite unique).
- Queue workload аз: mail ғана queue-ға.
- Frontend minimal JS, no SPA overhead.
- Cron: `schedule:run` + nightly backup.

## 19) Қабылдау критерийлері (UAT)
- Approve етілмеген user lesson page аша алмайды (403/redirect).
- Blocked status бірден күшіне енеді.
- Прогресс % дұрыс есептеледі.
- Course cover/logo upload шектеулері істейді.
- Admin logs барлық critical action-ды жазады.
- Lighthouse mobile perf >= 80 (cached pages).

## 20) Іске асыру кезеңдері (roadmap)
1. **Foundation (1-2 күн)**: Laravel, auth, roles, base layout.
2. **Core DB + Models (1 күн)**: migrations, relations, seeders.
3. **Admin Panel (2-3 күн)**: Filament resources + dashboard.
4. **User Cabinet (2 күн)**: courses, requests, my courses.
5. **Access Control (1-2 күн)**: policies, middleware, guards.
6. **Video Security Layer (1 күн)**: embed wrapper + headers.
7. **Progress Tracking (1 күн)**.
8. **Settings/Logo/Logs (1 күн)**.
9. **Hardening + Perf (1 күн)**.
10. **QA/UAT + Deploy (1 күн)**.

---

## Қысқа Laravel 12 код блоктары
### Enrollment тексеру сервисі
```php
public function hasApprovedAccess(User $user, Course $course): bool
{
    return Enrollment::query()
        ->where('user_id', $user->id)
        ->where('course_id', $course->id)
        ->where('status', 'approved')
        ->exists();
}
```

### Lesson complete endpoint
```php
public function complete(Lesson $lesson): RedirectResponse
{
    $user = auth()->user();

    LessonProgress::updateOrCreate(
        ['user_id' => $user->id, 'lesson_id' => $lesson->id],
        [
            'course_id' => $lesson->course_id,
            'is_completed' => true,
            'completed_at' => now(),
        ]
    );

    return back()->with('status', 'Сабақ аяқталды.');
}
```

### Rate limit (login)
```php
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->ip().'|'.$request->input('email'));
});
```
