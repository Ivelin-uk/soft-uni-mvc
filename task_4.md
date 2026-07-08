# Задание 4: Филтър за всяка колона и сортиране при клик върху заглавие

## Цел

Да надградите index таблиците така, че всяка важна колона да има собствен филтър, а клик
върху заглавието на колоната да сменя сортирането между възходящ и низходящ ред.

Задачата стъпва върху Задание 1 и Задание 3: вече имате index страници с pagination,
търсене, базово сортиране и export към Excel.

---

## Част 1: Основни понятия

### 1.1 Общ search vs. филтър по колона

Общото търсене (`searchString`) търси в няколко полета едновременно. Филтърът по колона е
по-точен: всяка колона има собствен input/select и влияе само на конкретното поле.

Пример:

- `firstName=Guy` филтрира само по `FirstName`;
- `jobTitle=Manager` филтрира само по `JobTitle`;
- `departmentId=3` филтрира само по избран department.

Тези филтри трябва да могат да се комбинират. Ако потребителят избере department и въведе
job title, резултатите трябва да отговарят и на двете условия.

### 1.2 Сортиране при клик върху заглавие

Всяко заглавие на колона трябва да е линк. При първи клик сортира възходящо, при втори клик
по същата колона сортира низходящо.

Пример:

```csharp
ViewData["FirstNameSortParam"] = sortOrder == "firstName" ? "firstName_desc" : "firstName";
```

После в LINQ:

```csharp
employees = sortOrder switch
{
    "firstName" => employees.OrderBy(e => e.FirstName),
    "firstName_desc" => employees.OrderByDescending(e => e.FirstName),
    _ => employees.OrderBy(e => e.EmployeeId),
};
```

В изгледа линкът трябва да пази всички активни филтри:

```html
<a asp-action="Index"
   asp-route-sortOrder="@ViewData["FirstNameSortParam"]"
   asp-route-firstName="@ViewData["CurrentFirstName"]"
   asp-route-lastName="@ViewData["CurrentLastName"]"
   asp-route-departmentId="@ViewData["CurrentDepartment"]">
    First Name
</a>
```

### 1.3 Pagination и export също пазят филтрите

Когато потребителят мине на следваща страница, сортира по друга колона или export-не към
Excel, всички активни филтри трябва да останат активни. Това означава, че трябва да
добавите новите `asp-route-*` параметри на три места:

- линковете за сортиране в header-а;
- pagination линковете;
- бутона **Export Excel**.

---

## Част 2: Задачата

Добавете филтър за всяка важна колона и клик-сортиране за index страниците на:

- **Employees**
- **Departments**
- **Projects**
- **Towns**
- **Addresses**
- **EmployeeProjects**

### Минимални изисквания по страница

| Страница | Филтри по колони | Сортиране по колони |
|---|---|---|
| Employees | First Name, Last Name, Job Title, Department, Manager, Hire Date от/до, Salary от/до | First Name, Last Name, Job Title, Department, Manager, Hire Date, Salary |
| Departments | Name, Manager | Name, Manager |
| Projects | Name, Description, Start Date от/до, End Date от/до | Name, Start Date, End Date |
| Towns | Name | Name |
| Addresses | Address Text, Town | Address Text, Town |
| EmployeeProjects | Employee, Project | Employee, Project |

За текстови полета използвайте `<input type="text">`. За свързани таблици използвайте
`<select>` с `SelectList`. За дати и числа използвайте два input-а: минимална и максимална
стойност.

---

## Част 3: Примерна имплементация за Employees

### 3.1 Променете signature-а на `Index`

```csharp
public async Task<IActionResult> Index(
    string? firstName,
    string? lastName,
    string? jobTitle,
    int? departmentId,
    int? managerId,
    DateTime? hireDateFrom,
    DateTime? hireDateTo,
    decimal? salaryFrom,
    decimal? salaryTo,
    string? sortOrder,
    int? pageNumber)
```

### 3.2 Запазете текущите стойности във `ViewData`

```csharp
ViewData["CurrentFirstName"] = firstName;
ViewData["CurrentLastName"] = lastName;
ViewData["CurrentJobTitle"] = jobTitle;
ViewData["CurrentDepartment"] = departmentId;
ViewData["CurrentManager"] = managerId;
ViewData["CurrentHireDateFrom"] = hireDateFrom?.ToString("yyyy-MM-dd");
ViewData["CurrentHireDateTo"] = hireDateTo?.ToString("yyyy-MM-dd");
ViewData["CurrentSalaryFrom"] = salaryFrom;
ViewData["CurrentSalaryTo"] = salaryTo;
ViewData["CurrentSort"] = sortOrder;
```

### 3.3 Приложете филтрите към `IQueryable`

```csharp
if (!string.IsNullOrWhiteSpace(firstName))
{
    employees = employees.Where(e => e.FirstName.Contains(firstName));
}

if (!string.IsNullOrWhiteSpace(lastName))
{
    employees = employees.Where(e => e.LastName.Contains(lastName));
}

if (departmentId.HasValue)
{
    employees = employees.Where(e => e.DepartmentId == departmentId);
}

if (hireDateFrom.HasValue)
{
    employees = employees.Where(e => e.HireDate >= hireDateFrom.Value);
}

if (hireDateTo.HasValue)
{
    employees = employees.Where(e => e.HireDate <= hireDateTo.Value);
}

if (salaryFrom.HasValue)
{
    employees = employees.Where(e => e.Salary >= salaryFrom.Value);
}

if (salaryTo.HasValue)
{
    employees = employees.Where(e => e.Salary <= salaryTo.Value);
}
```

### 3.4 Направете заглавията на таблицата линкове

Всяко заглавие на колона трябва да води към `Index` със съответния `sortOrder` и всички
текущи филтри.

Добра практика е да показвате малка стрелка до активната колона:

```html
@(ViewData["CurrentSort"]?.ToString() == "firstName" ? "↑" : "")
@(ViewData["CurrentSort"]?.ToString() == "firstName_desc" ? "↓" : "")
```

---

## Част 4: Критерии за приемане

- Всяка index таблица има отделни филтри за колоните, описани в таблицата с минимални
  изисквания.
- Филтрите могат да се комбинират помежду си.
- Клик върху заглавие на колона сортира възходящо; втори клик върху същата колона сортира
  низходящо.
- Активното сортиране е визуално ясно, например със стрелка нагоре/надолу.
- При смяна на страница всички филтри и текущото сортиране остават активни.
- При export към Excel всички филтри и текущото сортиране остават активни.
- Филтрирането и сортирането се изпълняват чрез `IQueryable`, преди pagination.
- Приложението компилира без грешки: `dotnet build`.
- Ръчно тествайте поне Employees и още две други страници с комбинация от два филтъра,
  сортиране и export.

## Съвети

- Започнете с една страница, например Employees, и чак след като работи стабилно, повторете
  модела за останалите.
- Не зареждайте всички редове с `ToListAsync()` преди филтрите. Това ще филтрира в паметта
  и ще стане бавно при повече данни.
- Внимавайте всички route параметри да имат еднакви имена в controller-а, формата,
  sorting линковете, pagination линковете и export бутона.
- Ако даден филтър е празен, просто не добавяйте `Where` условие за него.
