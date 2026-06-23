---
layout: post
title: "Reporting Examples Cookbook: Eight Real-World Patterns"
date: 2026-06-20
last_modified: 2026-06-20
comments: true
enable_syntax_highlighting: true
---

The examples that ship with Majorsilence Reporting cover the eight most common scenarios you will encounter when embedding a report engine in a .NET application. This post walks through each one with the actual example code, explaining the pattern and when to reach for it.

All examples follow the same three-step structure:

```
RdlEngineConfigInit() → Parse(rdlXml) → RunGetData() → RunRender()
```

The variation is in how data gets in and how many formats come out.

---

## 1. SQLite to PDF — the minimal case

**Example project:** `ExportSqliteToPdf`

The simplest path: a SQLite database, an RDL file, and a PDF output. Everything else builds on this pattern.

```csharp
using Majorsilence.Reporting.Rdl;

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var rdlPath = Path.Combine(baseDir, "Products.rdl");
var dbPath  = Path.Combine(baseDir, "sqlitetestdb2.db");
var outPath = Path.Combine(baseDir, "products.pdf");

// Inject the absolute database path before Parse().
// The engine validates the query schema at parse time, so it needs a
// real, resolvable connection string before RDLParser.Parse() returns.
var rdlXml = (await File.ReadAllTextAsync(rdlPath))
    .Replace("sqlitetestdb2.db", dbPath);

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems)
        Console.Error.WriteLine(err);
    return 1;
}

await report.RunGetData();

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

**Key points:**

- `RdlEngineConfigInit()` loads data provider registrations from `RdlEngineConfig.xml`. Call it once at application startup — before any `RDLParser` is constructed.
- The string-replace pattern for the database path is intentional. `RDLParser.Parse()` opens the database to validate the query schema, so it needs an absolute, resolvable path. The RDL file stores a short placeholder (just the filename); the application swaps it for the full path at runtime.
- `ErrorMaxSeverity > 4` checks for errors that prevent rendering. Values 1–4 are warnings; 5+ are fatal.
- `RunGetData()` executes the SQL. `RunRender()` uses the in-memory result set — no second database hit.

---

## 2. Parameterised Reports — filtering by value

**Example project:** `SqliteWithParameters`

The same report file produces different PDFs by changing the parameter value at runtime.

```csharp
using Majorsilence.Reporting.Rdl;

var country = args.Length > 0 ? args[0] : "Germany";

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var rdlPath = Path.Combine(baseDir, "CustomersByCountry.rdl");
var dbPath  = Path.Combine(baseDir, "sqlitetestdb2.db");
var outPath = Path.Combine(baseDir, $"customers-{country}.pdf");

// Inject both the DB path and the filter value before Parse().
// The SQL in the RDL uses a literal 'Germany' placeholder so the engine
// can validate the schema without needing a runtime parameter binding.
var safeCountry = country.Replace("'", "''");
var rdlXml = (await File.ReadAllTextAsync(rdlPath))
    .Replace("sqlitetestdb2.db", dbPath)
    .Replace("'Germany'", $"'{safeCountry}'");

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

// Report parameters are still passed to RunGetData so expressions
// in the report (page titles, headers) can reference them.
var parameters = new Dictionary<string, string> { ["Country"] = country };
await report.RunGetData(parameters);

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

**Why inject the value before `Parse()` instead of passing it as a parameter?**

The RDL engine validates the SQL query at parse time by running it against the database. If the SQL contains a placeholder parameter (`@Country`) and the engine cannot resolve it at parse time, schema validation fails. The pattern used here injects the value directly into the SQL string before parsing, avoiding this constraint. The parameter is still passed to `RunGetData` so it is available to report expressions.

Note the SQL-injection protection: `country.Replace("'", "''")` escapes single quotes before interpolating into the SQL. Always do this, or use parameterised queries where the engine supports them.

---

## 3. One Fetch, Many Formats

**Example project:** `ExportMultipleFormats`

Query the database exactly once, then render to as many output formats as you need. This is the pattern for batch export jobs.

```csharp
using Majorsilence.Reporting.Rdl;

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var dbPath  = Path.Combine(baseDir, "sqlitetestdb2.db");

var rdlXml = (await File.ReadAllTextAsync(Path.Combine(baseDir, "Orders.rdl")))
    .Replace("sqlitetestdb2.db", dbPath);

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

// Query the database exactly once
await report.RunGetData();

// Render to each format from the same in-memory data — no extra DB hit
var formats = new[]
{
    ("orders.pdf",  OutputPresentationType.PDF),
    ("orders.xlsx", OutputPresentationType.Excel2007),
    ("orders.csv",  OutputPresentationType.CSV),
    ("orders.html", OutputPresentationType.HTML),
};

foreach (var (filename, format) in formats)
{
    var ofs = new OneFileStreamGen(Path.Combine(baseDir, filename), true);
    await report.RunRender(ofs, format);
    Console.WriteLine($"Written: {filename}");
}
```

`RunRender()` rebuilds the page layout from the fetched data each time it is called. It does not re-execute the SQL. For CPU-intensive reports this means you pay the layout cost once per format, but you avoid the database round-trip entirely after the first call.

**All available `OutputPresentationType` values:**

| Value | Output |
|---|---|
| `PDF` | PDF 1.4 |
| `Excel2007` | XLSX (OpenXML) |
| `HTML` | Single HTML file |
| `CSV` | Comma-separated values |
| `RTF` | Rich Text Format |
| `XML` | Data as XML |
| `TIFF` | Multi-page TIFF image |
| `MHT` | MIME HTML archive |

---

## 4. Inject Data from C# Objects

**Example project:** `SetDataFromCode`

Skip the SQL query at runtime entirely. Pass any `IEnumerable<T>` — from an API call, a LINQ query, a microservice response — directly into the report as if it were query results.

```csharp
using Majorsilence.Reporting.Rdl;

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var dbPath  = Path.Combine(baseDir, "sqlitetestdb2.db");
var outPath = Path.Combine(baseDir, "sales-report.pdf");

// The RDL still references a database — it is used only at Parse() time
// for schema validation. At runtime, SetData() replaces the query result.
var rdlXml = (await File.ReadAllTextAsync(Path.Combine(baseDir, "SalesReport.rdl")))
    .Replace("sqlitetestdb2.db", dbPath);

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

// Build data from any source — API, service layer, LINQ, in-memory list
var salesData = new List<SaleRecord>
{
    new("Chai",                  "North America",  1250.00m, 50),
    new("Chang",                 "North America",   980.50m, 42),
    new("Aniseed Syrup",         "Europe",          432.00m, 24),
    new("Chef Anton's Cajun",    "Europe",         1875.25m, 75),
    new("Grandma's Boysenberry", "Asia Pacific",    640.00m, 32),
    // ... more rows
};

// Property names must match the <Field Name="..."> values in the RDL exactly
await report.DataSets["Data"].SetData(salesData);

// RunGetData is still required — it processes parameters and resolves sub-reports
await report.RunGetData(null);

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);

// Record type — property names drive the field binding
record SaleRecord(string Product, string Region, decimal Amount, int Quantity);
```

**When to use this pattern:**

- Your application already fetches the data (e.g. from an HTTP API or a domain service) and you do not want a second database round-trip.
- You are writing unit tests for report output — no live database needed.
- The data comes from a source the RDL engine cannot query directly (a gRPC service, a message bus, an in-process computation).
- You need to transform or enrich data before rendering (join multiple sources, apply business logic, aggregate differently than SQL allows).

If you do not want the engine to hit a real database at parse time, set `SkipDatabaseSchemaValidation = true` on the `RDLParser` before calling `Parse()`:

```csharp
var rdlp = new RDLParser(rdlXml)
{
    Folder = baseDir,
    SkipDatabaseSchemaValidation = true,
};
using var report = await rdlp.Parse();
```

---

## 5. JSON Files as a Data Source

**Example project:** `JsonToPdf`

No database at all. The JSON provider reads a local `.json` file as the data source. Useful for configuration-driven reports, reports generated from REST API exports, or scenarios where the data team delivers JSON instead of database access.

```csharp
using Majorsilence.Reporting.Rdl;

RdlEngineConfig.RdlEngineConfigInit();

var baseDir  = AppContext.BaseDirectory;
var jsonPath = Path.Combine(baseDir, "employees.json");
var outPath  = Path.Combine(baseDir, "employees.pdf");

// Inject the absolute JSON file path into the RDL before parsing
var rdlXml = (await File.ReadAllTextAsync(Path.Combine(baseDir, "Employees.rdl")))
    .Replace("employees.json", jsonPath);

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

await report.RunGetData();

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

The RDL data source declaration looks like this:

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
    <!-- List the fields to project; nested objects use underscore notation -->
    <CommandText>columns=EmployeeID,FirstName,LastName,Department,Contact_Email</CommandText>
  </Query>
</DataSet>
```

**Nested object access:** a JSON property `contact.email` is referenced in the RDL as `Contact_Email` — dots become underscores. Arrays of objects map to report rows naturally.

---

## 6. Product Catalog with Per-Row QR Codes

**Example project:** `ProductQrCodes`

Every row in the table gets its own QR code, generated from data fields at render time. The technique works for any barcode type and any content — product IDs, URLs, tracking codes, invoice numbers.

```csharp
using Majorsilence.Reporting.Rdl;

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var dbPath  = Path.Combine(baseDir, "sqlitetestdb2.db");
var outPath = Path.Combine(baseDir, "products-qr.pdf");

var rdlXml = (await File.ReadAllTextAsync(Path.Combine(baseDir, "ProductsWithQrCode.rdl")))
    .Replace("sqlitetestdb2.db", dbPath);

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

await report.RunGetData();

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

The QR code is declared inside the table's detail row in the RDL. The `Code` property uses a field expression so each row encodes its own data:

```xml
<CustomReportItem Name="ProductQR">
  <Type>QrCode</Type>
  <Top>2pt</Top><Left>2pt</Left>
  <Width>60pt</Width><Height>60pt</Height>
  <CustomProperties>
    <CustomProperty>
      <Name>Code</Name>
      <!-- Encode a structured string combining multiple fields -->
      <Value>="ID:" &amp; Fields!ProductID.Value &amp; "|" &amp; Fields!ProductName.Value</Value>
    </CustomProperty>
  </CustomProperties>
  <Source>Embedded</Source>
</CustomReportItem>
```

The expression combines `ProductID` and `ProductName` into a single scannable string. You could equally encode a URL, a serialised JSON payload, or any string your scanner expects.

---

## 7. All Barcode Types on One Page

**Example project:** `BarcodeShowcase`

Renders all six supported CRI barcode types to a single PDF — useful as a visual reference when deciding which format to use.

```csharp
using Majorsilence.Reporting.Rdl;

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var rdlPath = Path.Combine(baseDir, "BarcodeShowcase.rdl");
var outPath = Path.Combine(baseDir, "barcode-showcase.pdf");

// No database — all barcode values are static literals in the RDL
var rdlXml = await File.ReadAllTextAsync(rdlPath);

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

await report.RunGetData();

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

The RDL places one `<CustomReportItem>` per format, each with a `<Type>` element matching the registered name:

```xml
<!-- QR Code — square, equal Width/Height -->
<CustomReportItem Name="QR1">
  <Type>QrCode</Type>
  <Width>72pt</Width><Height>72pt</Height>
  <CustomProperties>
    <CustomProperty><Name>Code</Name><Value>https://github.com/majorsilence/Reporting</Value></CustomProperty>
  </CustomProperties>
  <Source>Embedded</Source>
</CustomReportItem>

<!-- Code 128 — landscape, roughly 2:1 -->
<CustomReportItem Name="BC128">
  <Type>BarCode128</Type>
  <Width>144pt</Width><Height>72pt</Height>
  <CustomProperties>
    <CustomProperty><Name>Code</Name><Value>HELLO-WORLD-128</Value></CustomProperty>
  </CustomProperties>
  <Source>Embedded</Source>
</CustomReportItem>
```

**All supported CRI barcode types:**

| RDL type name | Format | Shape | Notes |
|---|---|---|---|
| `QrCode` | QR Code | Square | Use equal Width / Height |
| `BarCode128` | Code 128 | Landscape | General-purpose; very common |
| `BarCode39` | Code 39 | Landscape | Older format; limited character set |
| `DataMatrix` | Data Matrix | Square | High density, small footprint |
| `AztecCode` | Aztec | Square | Used in transport tickets |
| `Pdf417` | PDF 417 | Landscape | High capacity; boarding passes, IDs |
| `ITF-14` | ITF-14 | Landscape | Property name is `ITF14`; 13–14 digits only |

Barcode rendering requires the `Majorsilence.Reporting.RdlCri.SkiaSharp` package in addition to the core engine.

---

## 8. Dynamic Barcode Type at Runtime

**Example project:** `DynamicBarcodeType`

The barcode format and the value it encodes are both controlled by caller-supplied parameters. The same RDL file can produce a QR code, a Code 128, or a DataMatrix depending on what the application passes in.

```csharp
using Majorsilence.Reporting.Rdl;

var barcodeType  = args.Length > 0 ? args[0] : "QrCode";
var barcodeValue = args.Length > 1 ? args[1] : "https://github.com/majorsilence/Reporting";

RdlEngineConfig.RdlEngineConfigInit();

var baseDir = AppContext.BaseDirectory;
var outPath = Path.Combine(baseDir, $"barcode-{barcodeType}.pdf");

var rdlXml = await File.ReadAllTextAsync(Path.Combine(baseDir, "BarcodeDemo.rdl"));

var rdlp = new RDLParser(rdlXml) { Folder = baseDir };
using var report = await rdlp.Parse();

if (report.ErrorMaxSeverity > 4)
{
    foreach (var err in report.ErrorItems) Console.Error.WriteLine(err);
    return 1;
}

var parameters = new Dictionary<string, string>
{
    ["BarcodeType"]  = barcodeType,   // e.g. "QrCode", "BarCode128", "DataMatrix"
    ["BarcodeValue"] = barcodeValue,  // the content to encode
};
await report.RunGetData(parameters);

var ofs = new OneFileStreamGen(outPath, true);
await report.RunRender(ofs, OutputPresentationType.PDF);
```

The RDL uses parameter expressions for both the format and the value:

```xml
<CustomReportItem Name="DynamicBarcode">
  <Type>={?BarcodeType}</Type>
  <Width>144pt</Width><Height>144pt</Height>
  <CustomProperties>
    <CustomProperty>
      <Name>Code</Name>
      <Value>={?BarcodeValue}</Value>
    </CustomProperty>
  </CustomProperties>
  <Source>Embedded</Source>
</CustomReportItem>
```

`{?ParameterName}` is the RDL expression syntax for a report parameter. Both the type and the content are resolved at render time from the dictionary passed to `RunGetData`.

---

## 9. AOT — Replace `<Code>` VB with C# Delegates

**Example project:** `AotCodeProvider`

When publishing with `<PublishAot>true</PublishAot>`, the VB compiler (`VBCodeProvider`) is unavailable. `RegisterCodeProvider` lets you supply a C# delegate dictionary that the engine calls in its place. The RDL `<Code>` element can still contain VB source — it is used on standard JIT runtimes and ignored when a provider is registered.

```csharp
using Majorsilence.Reporting.Rdl;

// Register BEFORE RdlEngineConfigInit and before Parse()
RdlEngineConfig.RegisterCodeProvider(report =>
    new RdlCodeFunctions()
        .Add("Grade", args =>
        {
            double score = Convert.ToDouble(args[0]);
            return score >= 90 ? "A" :
                   score >= 80 ? "B" :
                   score >= 70 ? "C" :
                   score >= 60 ? "D" : "F";
        })
        .Add("Commentary", args =>
        {
            double score = Convert.ToDouble(args[0]);
            return score >= 90 ? "Excellent" :
                   score >= 70 ? "Good" :
                   score >= 50 ? "Needs Improvement" : "Failing";
        })
        .Add("IsPassing", args => Convert.ToDouble(args[0]) >= 60)
);

RdlEngineConfig.RdlEngineConfigInit();
// ... Parse, SetData, RunGetData, RunRender as normal
```

The RDL expressions are unchanged — `=Code.Grade(Fields!Score.Value)`, `=Code.IsPassing(...)`, etc. Method names are matched case-insensitively against the registered delegate names.

The `<Code>` element in the RDL can document the VB equivalents (for non-AOT use) while the registration takes effect on AOT runtimes:

```xml
<Code>
Function Grade(score As Object) As String
    ' used on non-AOT runtimes; ignored when RegisterCodeProvider is called
    If CDbl(score) >= 90 Then Return "A"
    ' ...
End Function
</Code>
```

---

## 10. AOT — Static Helper Methods via `RegisterType`

**Example project:** `AotStaticHelpers`

When expressions call static methods on a class in your assembly — `=MyNamespace.MyClass.Method(args)` — the trimmer may remove those methods from the published binary. `RegisterType` tells the trimmer the class's public methods and constructors must be preserved.

```csharp
using Majorsilence.Reporting.Rdl;

// Register BEFORE RdlEngineConfigInit and before Parse()
// The fully-qualified name must match how the class appears in the RDL expressions
RdlEngineConfig.RegisterType(
    "AotStaticHelpers.SalesHelpers",
    typeof(SalesHelpers));

RdlEngineConfig.RdlEngineConfigInit();
// ... Parse, SetData, RunGetData, RunRender as normal

public static class SalesHelpers
{
    public static string FormatMoney(object value) =>
        $"{Convert.ToDecimal(value):C}";

    public static string Tier(object amount)
    {
        decimal v = Convert.ToDecimal(amount);
        return v >= 2000 ? "Platinum" :
               v >= 1000 ? "Gold" :
               v >= 500  ? "Silver" : "Bronze";
    }

    public static string TierColor(object amount)
    {
        decimal v = Convert.ToDecimal(amount);
        return v >= 2000 ? "#4B0082" :
               v >= 1000 ? "#B8860B" :
               v >= 500  ? "#708090" : "#8B4513";
    }
}
```

The RDL uses the fully-qualified class name directly in expressions. No `<CodeModules>` element is required:

```xml
<Value>=AotStaticHelpers.SalesHelpers.FormatMoney(Fields!Amount.Value)</Value>
<Value>=AotStaticHelpers.SalesHelpers.Tier(Fields!Amount.Value)</Value>
<Color>=AotStaticHelpers.SalesHelpers.TierColor(Fields!Amount.Value)</Color>
```

> Method dispatch still uses `MethodInfo.Invoke` internally. `RegisterType` ensures those methods survive trimming; it does not eliminate the reflection call. For fully reflection-free custom logic use `RegisterCodeProvider` + `RdlCodeFunctions` (example 9).

---

## 11. AOT — Instance Helper Class via `RegisterInstanceFactory`

**Example project:** `AotInstanceHelpers`

When a report has a `<Classes>` element declaring an instance variable, the engine normally creates the instance via `Assembly.CreateInstance`. Under AOT this fails. Two registrations replace it:

```csharp
using Majorsilence.Reporting.Rdl;

// Step 1: factory replaces Assembly.CreateInstance
RdlEngineConfig.RegisterInstanceFactory(
    "AotInstanceHelpers.OrderFormatter",
    () => new OrderFormatter());

// Step 2: type registration preserves the public methods for reflection-based dispatch
RdlEngineConfig.RegisterType(
    "AotInstanceHelpers.OrderFormatter",
    typeof(OrderFormatter));

RdlEngineConfig.RdlEngineConfigInit();
// ... Parse, SetData, RunGetData, RunRender as normal

public class OrderFormatter
{
    public string FormatDate(object value) =>
        Convert.ToDateTime(value).ToString("MMM d, yyyy");

    public string FormatMoney(object value) =>
        $"{Convert.ToDecimal(value):C}";

    public string StatusColor(object status) =>
        (status?.ToString() ?? "") switch
        {
            "Delivered"  => "#C6EFCE",
            "Shipped"    => "#FFEB9C",
            "Processing" => "#FFCCCC",
            _            => "White",
        };
}
```

The `<Classes>` element in the RDL declares the class and assigns it an instance name:

```xml
<Classes>
  <Class>
    <ClassName>AotInstanceHelpers.OrderFormatter</ClassName>
    <InstanceName>Helper</InstanceName>
  </Class>
</Classes>
```

Report expressions then call methods via the instance name (`Helper`), not the class name:

```xml
<Value>=Helper.FormatDate(Fields!OrderDate.Value)</Value>
<Value>=Helper.FormatMoney(Fields!Amount.Value)</Value>
<BackgroundColor>=Helper.StatusColor(Fields!Status.Value)</BackgroundColor>
```

No `<CodeModules>` element is required — the registration bypasses assembly scanning entirely.

---

## Beyond RDL: The Low-Level PDF Canvas

The `MajorsilencePdfExample` project shows a second, lower-level library: `Majorsilence.Pdf`. This is a canvas-level PDF writer — you position text, draw shapes, and embed images explicitly, with no RDL or report definition involved.

```csharp
using Majorsilence.Pdf;

var registry = new FontRegistry()
    .AddDirectory(fontsDir)
    .AddFallback("NotoSans");

PdfDocument.Create()
    .WithVersion(PdfVersion.Pdf20)
    .WithTitle("Invoice #INV-2025-0042")
    .WithFontRegistry(registry)
    .AddPage(PageSizes.Letter, canvas =>
    {
        var heading = TextStyle.Default
            .WithFamily("LiberationSans").WithSize(32).WithBold()
            .WithColor(PdfColor.White);

        // Draw a coloured header bar
        canvas.DrawRectangle(0, 0, pageWidth, 80,
            ShapeStyle.Filled(PdfColor.FromHex("#1A56A0")));

        canvas.DrawText("INVOICE", margin, 52, heading);

        // Draw a table row
        canvas.DrawText(productName, col1, y, body);
        canvas.DrawText(price.ToString("C"), col2, y,
            body.WithAlignment(TextAlignment.Right));
    })
    .Save("invoice.pdf");
```

The example covers 17 scenarios including text styles, shapes, lines, multi-page documents, custom fonts, images, clickable annotations, an invoice layout, a dashboard with a bar chart and pie chart, Unicode with Hebrew and Arabic RTL text, password-protected PDFs, digitally signed PDFs, and PDF merging.

Use `Majorsilence.Pdf` when you need pixel-level control over layout, when you are generating documents that are not data-driven (certificates, cover pages, design-heavy invoices), or when you want to build your own higher-level layout engine on top of the primitive operations.

---

## Choosing the Right Approach

| Scenario | Reach for |
|---|---|
| Query a database, produce a formatted report | `Majorsilence.Reporting.Rdl` + RDL file |
| Report already designed in SSRS / RDL format | `Majorsilence.Reporting.Rdl` — RDL files are portable |
| Data comes from a C# object or API response | `SetData()` with `Majorsilence.Reporting.Rdl` |
| Barcodes or QR codes in table rows | `Majorsilence.Reporting.RdlCri.SkiaSharp` + CRI |
| Pixel-precise layout, certificates, design docs | `Majorsilence.Pdf` canvas API |
| Merge multiple PDFs into one | `Majorsilence.Pdf.PdfMerger` |
| Password-protect or digitally sign a document | `Majorsilence.Pdf` with `PdfSecurity` / `PdfSignatureOptions` |
| Publishing with `PublishAot` or `PublishTrimmed` | `RegisterCodeProvider` / `RegisterType` / `RegisterInstanceFactory` |

---

## Further Reading

- [Quick Start Guide](/posts/quick-start) — installation and the core render pipeline
- [Adding Charts to Reports](/posts/charts) — 8 chart types and how to configure them
- [Replacing Crystal Reports and SSRS](/posts/replacing-crystal-reports-ssrs) — migration guide
- [Native AOT and Trimming Support](/posts/aot-compatibility) — full AOT guide with all four registration APIs
- [GitHub Examples](https://github.com/majorsilence/Reporting/tree/main/Examples) — runnable source for all examples in this post
- [GitHub Wiki](https://github.com/majorsilence/Reporting/wiki) — full documentation
