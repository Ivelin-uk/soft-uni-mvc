# Задание 5: Сортиране на таблица при клик върху колона

## Цел

Да добавите сортиране на данните в index таблиците при клик върху заглавието на дадена
колона. При първи клик данните трябва да се подреждат във възходящ ред, а при повторен
клик върху същата колона — в низходящ ред.

---

## Изисквания

Добавете сортиране в index страниците на:

- **Employees**
- **Departments**
- **Projects**
- **Towns**
- **Addresses**
- **EmployeeProjects**

Заглавието на всяка колона, по която може да се сортира, трябва да бъде линк към action-а
`Index`. Линкът трябва да изпраща GET параметър `sortOrder`.

Пример:

```html
<a asp-action="Index" asp-route-sortOrder="@ViewData["NameSortParam"]">
    Name
</a>
```

---

## Логика в controller-а

Action-ът `Index` трябва да приема параметър `sortOrder`:

```csharp
public async Task<IActionResult> Index(string? sortOrder)
```

Създайте параметър за следващата посока на сортиране:

```csharp
ViewData["NameSortParam"] = sortOrder == "name" ? "name_desc" : "name";
ViewData["CurrentSort"] = sortOrder;
```

След това приложете сортирането върху `IQueryable`, преди извикването на `ToListAsync()` или
преди pagination:

```csharp
towns = sortOrder switch
{
    "name" => towns.OrderBy(t => t.Name),
    "name_desc" => towns.OrderByDescending(t => t.Name),
    _ => towns.OrderBy(t => t.TownId)
};
```

---

## Визуализация на посоката

До името на активната колона покажете стрелка:

- `↑` за възходящ ред;
- `↓` за низходящ ред.

Пример:

```html
<a asp-action="Index" asp-route-sortOrder="@ViewData["NameSortParam"]">
    Name
    @(ViewData["CurrentSort"]?.ToString() == "name" ? "↑" : "")
    @(ViewData["CurrentSort"]?.ToString() == "name_desc" ? "↓" : "")
</a>
```

---

## Колони за сортиране

| Страница | Колони |
|---|---|
| Employees | First Name, Last Name, Job Title, Department, Manager, Hire Date, Salary |
| Departments | Name, Manager |
| Projects | Name, Start Date, End Date |
| Towns | Name |
| Addresses | Address Text, Town |
| EmployeeProjects | Employee, Project |

Ако страницата има търсене, филтриране или pagination, линковете за сортиране трябва да
запазват всички текущи GET параметри чрез съответните `asp-route-*` атрибути.

---

## Критерии за приемане

- Заглавията на поддържаните колони са кликаеми.
- Първият клик сортира данните във възходящ ред.
- Вторият клик върху същата колона сортира данните в низходящ ред.
- До активната колона се вижда правилната стрелка.
- При избор на друга колона данните се сортират по нея.
- Търсенето, филтрите и pagination-ът се запазват при сортиране.
- Сортирането се изпълнява чрез `IQueryable` преди зареждането на данните.
- Приложението компилира без грешки с `dotnet build`.

## Тестване

Ръчно проверете поне следните случаи:

1. Клик върху текстова колона и проверка на възходящия ред.
2. Втори клик върху същата колона и проверка на низходящия ред.
3. Сортиране по колона със свързани данни, например Department, Manager или Town.
4. Смяна на страница след сортиране и проверка, че избраният ред се запазва.
