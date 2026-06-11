---
layout: post
title: Quick Start Guide
date: 2025-12-28
last_modified: 2026-06-01
comments: true
enable_syntax_highlighting: true
---

Majorsilence Reporting is an open-source .NET reporting framework for generating PDFs, Excel files, HTML, CSV, and more from SQL databases, JSON files, or C# objects. It runs cross-platform on .NET 8 and .NET 10, and is built around the open RDL (Report Definition Language) standard.

## Installation

Add the packages you need via NuGet:

```bash
# Core rendering engine (required)
dotnet add package Majorsilence.Reporting.RdlEngine

# Barcode / QR Code support via ZXing.Net (optional)
dotnet add package Majorsilence.Reporting.RdlCri

# Programmatic report & document creation (optional)
dotnet add package Majorsilence.Reporting.RdlCreator
```

On Linux and macOS, use the **SkiaSharp** build configuration (`Debug-DrawingCompat` / `Release-DrawingCompat`) for cross-platform rendering. Windows can use the default configuration.

---

## Key Concepts

**`RdlEngineConfigInit()`** — call this once at application startup before parsing any reports. It loads data provider registrations and CRI type mappings from `RdlEngineConfig.xml`.

**Parse-time DB connection** — when `RDLParser.Parse()` is called it opens the database to validate the query schema. This means the connection string inside the RDL must resolve to a real database *before* `Parse()` returns. The standard pattern is to inject the absolute path into the RDL XML string before calling `Parse()`.

**`RunGetData` / `RunRender`** — call `RunGetData` once to execute queries, then call `RunRender` once per output format. Each render pass rebuilds pages from the already-fetched data without re-querying the database.

---

## Render a SQLite report to PDF

```csharp
using Majorsilence.Reporting.Rdl;

// One-time setup
RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var dbPath  = Path.Combine(baseDir, "northwind.db");
var outPath = Path.Combine(baseDir, "products.pdf");

// Inject absolute DB path before Parse() so schema validation succeeds
var rdlXml = (await File.ReadAllTextAsync(Path.Combine(baseDir, "Products.rdl")))
    .Replace("northwind.db", dbPath);

var parser = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await parser.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems)
        Console.Error.WriteLine(err);
    return;
}

await report.RunGetData(null);

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

---

## Export to multiple formats in one pass

```csharp
await report.RunGetData(null);  // fetch data once

foreach (var (filename, format) in new[]
{
    ("report.pdf",  OutputPresentationType.PDF),
    ("report.xlsx", OutputPresentationType.Excel2007),
    ("report.csv",  OutputPresentationType.CSV),
    ("report.html", OutputPresentationType.HTML),
})
{
    var ofs = new OneFileStreamGen(Path.Combine(baseDir, filename), true);
    await report.RunRender(ofs, format);
    Console.WriteLine($"Written: {filename}");
}
```

---

## Read data from a JSON file

Declare the data source in your RDL with the `Json` provider. Nested objects are accessed using underscore notation (`Contact_Email` for `contact.email`).

```xml
<DataSource Name="DS1">
  <ConnectionProperties>
    <DataProvider>Json</DataProvider>
    <ConnectString>file=employees.json</ConnectString>
  </ConnectionProperties>
</DataSource>

<DataSet Name="Data">
  <Query>
    <DataSourceName>DS1</DataSourceName>
    <CommandText>columns=EmployeeID,FirstName,LastName,Department,Contact_Email</CommandText>
  </Query>
  <!-- Field elements for each column... -->
</DataSet>
```

In C#, inject the absolute path before parsing:

```csharp
var rdlXml = (await File.ReadAllTextAsync("Employees.rdl"))
    .Replace("employees.json", Path.Combine(baseDir, "employees.json"));
```

---

## Pass report parameters

Use `RunGetData(IDictionary<string, string>)` to supply parameter values. To filter by a SQL column, inject the value directly into the RDL SQL string before `Parse()` — passing a query parameter value post-parse will fail because the database tries to bind it during schema validation.

```csharp
var country = "Germany";

var rdlXml = (await File.ReadAllTextAsync("CustomersByCountry.rdl"))
    .Replace("northwind.db", dbPath)
    .Replace("'Germany'", $"'{country.Replace("'", "''")}'");  // inject filter

var parser = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await parser.Parse();

// Parameters are still passed for use in report expressions like page titles
var parameters = new Dictionary<string, string> { ["Country"] = country };
await report.RunGetData(parameters);
```

---

## Add a QR Code or barcode (CRI)

Reference `Majorsilence.Reporting.RdlCri` and place a `<CustomReportItem>` in your RDL:

```xml
<CustomReportItem Name="ProductQR">
  <Type>QrCode</Type>
  <Top>2pt</Top><Left>2pt</Left>
  <Width>72pt</Width><Height>72pt</Height>
  <CustomProperties>
    <CustomProperty>
      <Name>Code</Name>
      <!-- Expression bound to data field -->
      <Value>=Fields!ProductName.Value</Value>
    </CustomProperty>
  </CustomProperties>
  <Source>Embedded</Source>
</CustomReportItem>
```

**Supported types** (registered in `RdlEngineConfig.xml`):

| Type name | Format | Notes |
|---|---|---|
| `QrCode` | QR Code | Square; use equal Width/Height |
| `BarCode128` | Code 128 | Landscape, ~2:1 ratio |
| `BarCode39` | Code 39 | Landscape, ~2:1 ratio |
| `DataMatrix` | Data Matrix | Landscape, ~2:1 ratio |
| `AztecCode` | Aztec | Square |
| `Pdf417` | PDF 417 | Landscape |
| `ITF-14` | ITF-14 | **Property name is `ITF14`**; value must be 13–14 digits |

To select the barcode type at runtime, use a report parameter expression:

```xml
<Type>={?BarcodeType}</Type>
```

---

## Inject data from C# objects

Use `DataSet.SetData()` to bypass the SQL query and feed data directly from a collection. The RDL still needs a real database at parse time for schema validation; use a `LIMIT 0` stub query whose column aliases match your record's property names.

```csharp
record SaleRecord(string Product, string Region, decimal Amount, int Quantity);

var data = new List<SaleRecord>
{
    new("Widget A", "North", 1250.00m, 50),
    new("Widget B", "South",  840.50m, 33),
};

// Property names must match the RDL Field names exactly
await report.DataSets["Data"].SetData(data);
await report.RunGetData(null);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

---

## Create a report programmatically

Use `Majorsilence.Reporting.RdlCreator` to generate an RDL from a SQL query without a pre-made `.rdl` file:

```csharp
using Majorsilence.Reporting.RdlCreator;

RdlEngineConfig.RdlEngineConfigInit();

var create = new Create();
using var report = await create.GenerateRdl(
    dataProvider:    "Microsoft.Data.Sqlite",
    connectionString: $"Data Source={dbPath}",
    sql:             "SELECT ProductID, ProductName, UnitPrice FROM Products",
    pageHeaderText:  "Product Catalog");

var ofs = new Majorsilence.Reporting.Rdl.OneFileStreamGen("catalog.pdf", true);
await report.RunGetData(null);
await report.RunRender(ofs, Majorsilence.Reporting.Rdl.OutputPresentationType.PDF);
```

---

## Reference

- [Wiki home](https://github.com/majorsilence/Reporting/wiki) — full documentation
- [All wiki pages](https://github.com/majorsilence/Reporting/wiki/_pages)
- [Migration to v5](https://github.com/majorsilence/Reporting/wiki/Migration-to-v5)
- [JSON Data Provider](https://github.com/majorsilence/Reporting/wiki/Json-Data-Provider)
- [Dynamic connection strings](https://github.com/majorsilence/Reporting/wiki/Set-Connection-String---Runtime)
- [Contributing](https://github.com/majorsilence/Reporting/wiki/Contribute)
