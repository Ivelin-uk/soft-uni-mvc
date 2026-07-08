# Задание 3: Export на таблици към Excel

## Цел

Да добавите възможност данните от index таблиците в приложението да се експортират към
Excel файл (`.xlsx`). Export-ът трябва да уважава текущото търсене, филтриране и сортиране,
които потребителят е избрал в страницата.

Задачата тръгва от проекта след Задание 1: index страниците вече трябва да имат
`searchString`, филтри, `sortOrder` и pagination.

---

## Част 1: Основни понятия

### 1.1 Какво значи export към Excel

Потребителят трябва да може да натисне бутон **Export Excel** от дадена index страница и да
получи `.xlsx` файл със същите данни, които вижда според активните филтри и сортировка.

Важно: export-ът не трябва да взима само текущата страница от pagination-а. Ако филтърът
намира 83 служители, а таблицата показва страница 1 с 10 реда, Excel файлът трябва да
съдържа всичките 83 филтрирани реда.

### 1.2 Библиотека за Excel файлове

Инсталирайте NuGet пакета `ClosedXML`, който позволява създаване на `.xlsx` файлове без
инсталиран Microsoft Excel:

```bash
dotnet add package ClosedXML
```

Примерна идея за създаване на workbook:

```csharp
using ClosedXML.Excel;

using var workbook = new XLWorkbook();
var worksheet = workbook.Worksheets.Add("Employees");

worksheet.Cell(1, 1).Value = "First Name";
worksheet.Cell(1, 2).Value = "Last Name";

worksheet.Cell(2, 1).Value = "Guy";
worksheet.Cell(2, 2).Value = "Gilbert";

using var stream = new MemoryStream();
workbook.SaveAs(stream);

return File(
    stream.ToArray(),
    "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "employees.xlsx");
```

### 1.3 Не дублирайте цялата LINQ логика

В `EmployeesController.Index` вече има логика за:

- търсене;
- филтриране;
- сортиране;
- pagination.

Export action-ът ще има нужда от същата заявка, но без pagination. Затова изнесете общата
част в private метод, който връща `IQueryable<Employee>`.

Примерна структура:

```csharp
private IQueryable<Employee> BuildEmployeesQuery(string? searchString, int? departmentId, string? sortOrder)
{
    var employees = _context.Employees
        .Include(e => e.Department)
        .Include(e => e.Manager)
        .AsQueryable();

    // search
    // filter
    // sort

    return employees;
}
```

После `Index` използва този метод и накрая прилага pagination:

```csharp
var employees = BuildEmployeesQuery(searchString, departmentId, sortOrder);
var paginatedEmployees = await PaginatedList<Employee>.CreateAsync(employees, pageNumber ?? 1, 10);
return View(paginatedEmployees);
```

А export action-ът използва същия метод, но прави `ToListAsync()`:

```csharp
var employees = await BuildEmployeesQuery(searchString, departmentId, sortOrder).ToListAsync();
```

---

## Част 2: Задачата

Добавете export към Excel за index страниците на:

- **Employees** (`EmployeesController`, `Views/Employees/Index.cshtml`)
- **Departments** (`DepartmentsController`, `Views/Departments/Index.cshtml`)
- **Projects** (`ProjectsController`, `Views/Projects/Index.cshtml`)
- **Towns** (`TownsController`, `Views/Towns/Index.cshtml`)
- **Addresses** (`AddressesController`, `Views/Addresses/Index.cshtml`)
- **EmployeeProjects** (`EmployeeProjectsController`, `Views/EmployeeProjects/Index.cshtml`)

### Стъпки

1. Инсталирайте `ClosedXML`.
2. Във всеки controller изнесете общата query логика от `Index` в private метод.
3. Добавете action `ExportExcel`, който приема същите GET параметри като `Index`
   (`searchString`, съответния филтър, `sortOrder`).
4. В `ExportExcel` заредете всички филтрирани и сортирани редове без pagination.
5. Създайте Excel workbook с worksheet за съответната таблица.
6. Добавете header ред с имената на колоните.
7. Добавете редовете с данни.
8. Върнете файла с `File(...)` и правилния MIME type.
9. Във всяка index страница добавете бутон **Export Excel**, който пази текущите параметри
   чрез `asp-route-*`.

Пример за бутон:

```html
<a asp-action="ExportExcel"
   asp-route-searchString="@ViewData["CurrentFilter"]"
   asp-route-departmentId="@ViewData["CurrentDepartment"]"
   asp-route-sortOrder="@ViewData["CurrentSort"]"
   class="btn btn-success">
    Export Excel
</a>
```

### Минимални колони в Excel файловете

| Страница | Колони в Excel |
|---|---|
| Employees | First Name, Last Name, Job Title, Department, Manager, Hire Date, Salary |
| Departments | Name, Manager |
| Projects | Name, Description, Start Date, End Date |
| Towns | Name |
| Addresses | Address Text, Town |
| EmployeeProjects | Employee, Project |

---

## Критерии за приемане

- На всяка index страница има бутон **Export Excel**.
- Export-ът сваля `.xlsx` файл, който се отваря коректно в Excel, LibreOffice или Google
  Sheets.
- Excel файлът съдържа всички филтрирани резултати, а не само текущата страница от
  pagination-а.
- Ако потребителят търси, филтрира или сортира таблицата, export-ът запазва същите условия.
- Header редът в Excel файла е ясен и четим.
- Датите са форматирани като `yyyy-MM-dd`.
- Приложението компилира без грешки: `dotnet build`.
- Ръчно тествайте поне три случая: export без филтър, export с филтър, export след сортиране
  в низходящ ред.

## Съвети

- Не копирайте голям блок LINQ логика два пъти в един controller. Ако `Index` и
  `ExportExcel` имат различна логика, много лесно ще започнат да връщат различни резултати.
- Не слагайте pagination в export action-а.
- За свързани данни (`Department`, `Manager`, `Town`, `Project`) не забравяйте `.Include(...)`,
  иначе навигационните property-та може да са празни.
- Може да използвате `worksheet.Columns().AdjustToContents();`, за да направите колоните
  автоматично достатъчно широки.
