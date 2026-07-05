# Задание 2: Потребители, роли и оторизация

## Цел

Да добавите система за автентикация (login/logout) към приложението, с роля `Admin`,
която да вижда всичко, и обикновени (неавтентикирани или логнати с друга роля)
потребители, които да имат достъп само до `Home/Index`, `Home/Privacy` и страницата за
вход. Всеки опит за достъп до друг ресурс трябва да връща грешка **403 Forbidden**.

Задачата тръгва от готовия проект след Задание 1 (или директно от `main`, ако не сте
работили по Задание 1) — предполага се, че вече имате работеща локална база `softuni` и
приложението стартира с `dotnet run`.

---

## Част 1: Основни понятия

В проекта засега няма никаква автентикация — всеки може да отваря всеки адрес. Преди да
пишете код, е важно да разберете няколко неща, които ASP.NET Core Ви дава наготово.

### 1.1 Автентикация vs. оторизация

- **Автентикация** (authentication) отговаря на въпроса "Кой си ти?" — процесът на login,
  при който потвърждаваме самоличността с имейл + парола.
- **Оторизация** (authorization) отговаря на въпроса "Имаш ли право да правиш това?" — след
  като знаем кой е потребителят, решаваме дали да му позволим достъп до конкретен ресурс
  (например само `Admin` да вижда `/Employees`).

Двете са отделни middleware-и в ASP.NET Core: `app.UseAuthentication()` (кой е
потребителят) винаги трябва да е **преди** `app.UseAuthorization()` (има ли право) в
`Program.cs`.

### 1.2 Cookie автентикация

Няма да ползваме пълния ASP.NET Core Identity (той идва с готови таблици, UI, email
confirmation и т.н. — прекалено много магия за целите на тази задача). Вместо това ще
използваме директно вградената **cookie автентикация**, която е достатъчна за
login/logout с роли:

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/Account/Login";
    });
```

При успешен login "подписваме" потребителя със `SignInAsync`, като му прикачваме
**claims** (име, роля):

```csharp
var claims = new List<Claim>
{
    new Claim(ClaimTypes.Email, user.Email),
    new Claim(ClaimTypes.Role, user.Role),
};
var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(identity));
```

ASP.NET Core криптира тези claims в бисквитка (cookie), която браузърът праща обратно на
всяка следваща заявка. Middleware-ът я разчита и попълва `HttpContext.User` — оттам идва
`User.Identity.IsAuthenticated` и `User.IsInRole("Admin")`, които ще ползвате в контролерите
и изгледите.

Logout е просто обратното:

```csharp
await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
```

### 1.3 Хеширане на пароли

**Никога** не пазете паролата като plain text в базата. Класът
[`PasswordHasher<TUser>`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.identity.passwordhasher-1)
(namespace `Microsoft.AspNetCore.Identity`) е част от споделения ASP.NET Core framework —
не се налага да добавяте нов NuGet пакет, за да го ползвате в този проект.

```csharp
var hasher = new PasswordHasher<User>();

// при създаване на потребител
user.PasswordHash = hasher.HashPassword(user, "plainTextPassword");

// при login
var result = hasher.VerifyHashedPassword(user, user.PasswordHash, enteredPassword);
if (result == PasswordVerificationResult.Failed)
{
    // грешна парола
}
```

`HashPassword` произвежда различен резултат всеки път (заради случайна сол), затова не
сравнявайте хешове с `==` — винаги минавайте през `VerifyHashedPassword`.

### 1.4 `[Authorize]`, роли и 403 вместо redirect

По подразбиране, ако неавтентикиран потребител опита да отвори адрес с `[Authorize]`,
ASP.NET Core го пренасочва (HTTP 302) към `LoginPath`. Заданието изисква нещо по-строго:
**директна грешка 403**, без пренасочване, независимо дали потребителят изобщо не е
логнат, или е логнат, но няма роля `Admin`. Постига се с две събития в конфигурацията на
бисквитката:

```csharp
.AddCookie(options =>
{
    options.LoginPath = "/Account/Login";
    options.Events.OnRedirectToLogin = context =>
    {
        context.Response.StatusCode = StatusCodes.Status403Forbidden;
        return Task.CompletedTask;
    };
    options.Events.OnRedirectToAccessDenied = context =>
    {
        context.Response.StatusCode = StatusCodes.Status403Forbidden;
        return Task.CompletedTask;
    };
});
```

После, за да защитите даден контролер, е достатъчен атрибут над класа:

```csharp
[Authorize(Roles = "Admin")]
public class EmployeesController : Controller
```

А там, където достъпът трябва да е свободен за всички (Home, страницата за вход), ползвате
`[AllowAnonymous]`.

---

## Част 2: Задачата

### 2.1 Таблица `users` в базата данни

Добавете таблица `users` в `softuni` базата — по същия принцип, по който вече изглеждат
останалите таблици (вижте `softuni.sql`), с snake_case имена на колоните:

```sql
CREATE TABLE `users` (
  `user_id` INT NOT NULL AUTO_INCREMENT,
  `email` VARCHAR(100) NOT NULL,
  `password_hash` VARCHAR(255) NOT NULL,
  `role` VARCHAR(20) NOT NULL,
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `UQ_users_email` (`email`)
);
```

Съответно добавете модела [Models/User.cs](Models/User.cs):

```csharp
namespace soft_uni_mvc.Models;

public partial class User
{
    public int UserId { get; set; }

    public string Email { get; set; } = null!;

    public string PasswordHash { get; set; } = null!;

    public string Role { get; set; } = null!;
}
```

И го регистрирайте в [Data/SoftUniContext.cs](Data/SoftUniContext.cs) — добавете
`DbSet<User> Users` и mapping в `OnModelCreating`, по същия начин, по който са мапнати
`Towns` или `Departments` (виж редове 183-195):

```csharp
modelBuilder.Entity<User>(entity =>
{
    entity.HasKey(e => e.UserId).HasName("PRIMARY");

    entity.ToTable("users");

    entity.HasIndex(e => e.Email, "UQ_users_email").IsUnique();

    entity.Property(e => e.UserId).HasColumnName("user_id");
    entity.Property(e => e.Email).HasMaxLength(100).HasColumnName("email");
    entity.Property(e => e.PasswordHash).HasMaxLength(255).HasColumnName("password_hash");
    entity.Property(e => e.Role).HasMaxLength(20).HasColumnName("role");
});
```

### 2.2 Добавяне на admin потребител

Понеже паролата трябва да е хеширана, а хешът се генерира с код (не можете просто да
напишете паролата в SQL), най-лесният начин е да добавите еднократен "seed" метод, който
се изпълнява при стартиране на приложението и създава admin потребителя, ако той все още
не съществува. В `Program.cs`, след `var app = builder.Build();`:

```csharp
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<SoftUniContext>();
    if (!db.Users.Any(u => u.Role == "Admin"))
    {
        var hasher = new PasswordHasher<User>();
        var admin = new User { Email = "admin@softuni.bg", Role = "Admin" };
        admin.PasswordHash = hasher.HashPassword(admin, "ChangeMe123!");
        db.Users.Add(admin);
        db.SaveChanges();
    }
}
```

Това гарантира, че администраторският акаунт съществува веднага след `dotnet run`, без да
се налага ръчно да смятате хеш и да пишете `INSERT` заявка.

### 2.3 Login / Logout

Създайте `Controllers/AccountController.cs` с три action метода:

- `GET Login` — `[AllowAnonymous]`, връща празна форма (view model с `Email`,
  `Password`).
- `POST Login` — `[AllowAnonymous]`, намира потребителя по имейл, проверява паролата с
  `VerifyHashedPassword`, при успех прави `SignInAsync` с claims за имейл и роля и
  пренасочва към `Home/Index`; при грешка добавя `ModelState` грешка и връща същия view.
- `POST Logout` — `[Authorize]`, `SignOutAsync` и пренасочва към `Home/Index`.

Добавете `Views/Account/Login.cshtml` с форма за имейл + парола (по образец на другите
`Create.cshtml` изгледи в проекта — `asp-for`, `<div class="text-danger" asp-validation-summary="All">` за грешки).

В [Views/Shared/_Layout.cshtml](Views/Shared/_Layout.cshtml) добавете в navbar-а линк
"Login", ако потребителят не е логнат, или имейла му + бутон "Logout" (form с `POST`), ако е логнат — проверете с `User.Identity!.IsAuthenticated`.

### 2.4 Оторизация по роли

- `HomeController` (`Index`, `Privacy`) и `AccountController.Login` остават достъпни за
  всички — маркирайте ги с `[AllowAnonymous]` (или просто не слагайте `[Authorize]` върху
  тях, стига по подразбиране да няма глобален authorization filter).
- `AccountController.Logout` изисква логнат потребител — `[Authorize]`.
- Всички останали контролери (`Employees`, `Departments`, `Projects`, `Addresses`,
  `Towns`, `EmployeeProjects`) трябва да са достъпни само за `Admin` —
  `[Authorize(Roles = "Admin")]` над класа.

Не забравяйте `app.UseAuthentication();` **преди** `app.UseAuthorization();` в
`Program.cs` — без ред, оторизацията винаги ще вижда анонимен потребител.

| Адрес | Анонимен / обикновен потребител | Admin |
|---|---|---|
| `/Home/Index` | ✅ | ✅ |
| `/Home/Privacy` | ✅ | ✅ |
| `/Account/Login` | ✅ | ✅ |
| `/Account/Logout` | 403 (не е логнат) | ✅ |
| `/Employees`, `/Departments`, `/Projects`, `/Addresses`, `/Towns`, `/EmployeeProjects` | **403** | ✅ |

---

## Критерии за приемане

- Таблица `users` съществува, с уникален `email`, `password_hash` (никога plain text
  парола) и `role`.
- В базата има поне един потребител с роля `Admin`, създаден автоматично при
  стартиране на приложението.
- Login формата приема имейл + парола, проверява ги срещу базата и при успех логва
  потребителя (cookie); при грешни данни показва съобщение за грешка, без да разкрива
  дали проблемът е в имейла или паролата.
- Logout прекратява сесията (след него защитените адреси отново връщат 403).
- Admin потребителят вижда и може да отваря всички изгледи в приложението.
- Обикновен (неавтентикиран или без роля `Admin`) потребител вижда само Home, Privacy и
  Login; всяка друга заявка (директно въвеждане на адрес, не само през навигацията) връща
  **HTTP 403**, а не пренасочване към login страницата.
- Приложението компилира и стартира без грешки: `dotnet build`.
- Ръчно тествайте: (1) отворете `/Employees` без да сте логнати — очаквайте 403; (2)
  логнете се като admin, отворете `/Employees` — очаквайте успех; (3) излезте (Logout) и
  пробвайте пак `/Employees` — отново 403.

## Съвети

- Не добавяйте пълния ASP.NET Core Identity (`AddIdentity`, `UserManager`,
  `SignInManager`) — прекалено много е за обхвата на тази задача и ще Ви създаде
  допълнителни таблици и миграции, които не са част от изискването. Cookie
  автентикацията "на ръка", описана по-горе, е достатъчна.
- `PasswordHasher<TUser>` изисква generic параметър (модела Ви `User`), но реално не чете
  никое от полетата му при hashing/verify — подава се само за типова коректност.
- Ако след промени в `Program.cs` всички заявки започнат да връщат 403 неочаквано,
  проверете реда на `UseAuthentication`/`UseAuthorization`, и че `HomeController`/
  `AccountController.Login` наистина имат `[AllowAnonymous]`.
- Тествайте в incognito прозорец на браузъра, за да сте сигурни, че не виждате стара
  бисквитка от предишен login.