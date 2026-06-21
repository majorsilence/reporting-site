---
layout: post
title: "Native AOT and Trimming Support"
date: 2026-06-21
last_modified: 2026-06-21
comments: true
enable_syntax_highlighting: true
---

Majorsilence Reporting can be published with .NET's Native AOT compiler (`<PublishAot>true</PublishAot>`) and the Trim Analyzer (`<PublishTrimmed>true</PublishTrimmed>`). Most of the engine is already AOT-compatible. The features that rely on runtime code generation — the `<Code>` element's inline VB source, CodeModule assembly loading, and `<Classes>` instance creation — require a one-time registration call at application startup to replace the reflection-based path with pre-compiled C# code.

This page explains the four registration APIs, when to use each one, and shows a runnable example for each pattern.

---

## How it works

At startup, before calling `RdlEngineConfigInit()` or parsing any report, you register factories and types with `RdlEngineConfig`. When the engine encounters a feature that would normally use reflection or dynamic code, it checks the registrations first. If one is found it is used; if not, the engine falls back to the traditional path (which is still fully functional on non-AOT runtimes).

The same application binary therefore works in three modes without any code changes:

| Publish mode | `<Code>` element | CodeModules | `<Classes>` |
|---|---|---|---|
| Standard JIT | VB compiled at runtime | Loaded from disk assembly | Instance created via reflection |
| Trimmed (`PublishTrimmed`) | VB compiled at runtime | Registered type used | Registered factory used |
| Native AOT (`PublishAot`) | Registered C# delegates used | Registered type used | Registered factory used |

---

## Enabling AOT for your project

Add `<PublishAot>true</PublishAot>` (or `<PublishTrimmed>true</PublishTrimmed>` for trimming only) to your project file:

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

Then add the registration calls before `RdlEngineConfigInit()`:

```csharp
// 1. Register any needed factories
RdlEngineConfig.RegisterCodeProvider(...);
RdlEngineConfig.RegisterType(...);
RdlEngineConfig.RegisterInstanceFactory(...);
RdlEngineConfig.RegisterCustomReportItem<T>(...);

// 2. Then initialise the engine as usual
RdlEngineConfig.RdlEngineConfigInit();
```

---

## API 1 — `RegisterCodeProvider` — replace inline VB with C# delegates

**Use when:** your RDL has a `<Code>` element with inline VB source, and expressions like `=Code.FormatValue(Fields!X.Value)` call into it.

Under AOT the VB compiler (`VBCodeProvider`) is unavailable. `RegisterCodeProvider` lets you supply a C# delegate dictionary that the engine calls instead.

```csharp
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
```

The delegate names are case-insensitive and must match the method names called in RDL expressions. The `args` array contains the evaluated arguments in order.

**RDL side:** the `<Code>` element can still contain VB source. It is used on non-AOT runtimes when no `RegisterCodeProvider` call is present. On AOT runtimes (or whenever `RegisterCodeProvider` has been called) the VB source is ignored and the registered delegates are used instead.

```xml
<Code>
Function Grade(score As Object) As String
    If CDbl(score) >= 90 Then Return "A"
    ' ... etc
End Function
</Code>
```

The expression syntax in the report body is unchanged:

```xml
<Value>=Code.Grade(Fields!Score.Value)</Value>
<Value>=IIf(Code.IsPassing(Fields!Score.Value), "Pass", "Fail")</Value>
```

**`RdlCodeFunctions` async overload:** if your delegate needs to call an async API (database, HTTP, etc.) use the async signature:

```csharp
.Add("LivePrice", async args =>
{
    string ticker = args[0]?.ToString() ?? "";
    return await priceService.GetCurrentPrice(ticker);
})
```

**Full example:** `Examples/AotCodeProvider` — generates a student score-card PDF using `Code.Grade`, `Code.Commentary`, and `Code.IsPassing`.

---

## API 2 — `RegisterType` — preserve static helper class methods

**Use when:** your RDL expressions call static methods on a class in your assembly, using the fully-qualified form `=MyNamespace.MyClass.Method(args)`.

Without registration the trimmer may remove those methods from the published binary, causing a `MissingMethodException` at render time. `RegisterType` tells the trimmer the class's public methods and constructors must be preserved.

```csharp
RdlEngineConfig.RegisterType(
    "AotStaticHelpers.SalesHelpers",  // fully-qualified class name as it appears in the RDL expression
    typeof(SalesHelpers));            // the [DynamicallyAccessedMembers] annotation pins the methods
```

The registration key must match exactly how the class is referenced in the RDL. No `<CodeModules>` element is needed in the RDL — the registration bypasses assembly loading entirely.

**RDL expressions:**

```xml
<Value>=AotStaticHelpers.SalesHelpers.FormatMoney(Fields!Amount.Value)</Value>
<Value>=AotStaticHelpers.SalesHelpers.Tier(Fields!Amount.Value)</Value>
<Color>=AotStaticHelpers.SalesHelpers.TierColor(Fields!Amount.Value)</Color>
```

**The helper class:**

```csharp
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
}
```

> **Note:** method dispatch still uses reflection internally (`MethodInfo.Invoke`). The `RegisterType` registration ensures those methods survive trimming; it does not eliminate the reflection call itself. For fully AOT-compatible custom logic without any reflection use `RegisterCodeProvider` + `RdlCodeFunctions` instead.

**Full example:** `Examples/AotStaticHelpers` — generates a sales report with tiered formatting driven by static helper methods.

---

## API 3 — `RegisterInstanceFactory` + `RegisterType` — `<Classes>` element instances

**Use when:** your RDL has a `<Classes>` element declaring an instance variable, and expressions call methods on it: `=Helper.FormatDate(...)`.

Two registrations are needed:

```csharp
// Creates the instance without Assembly.CreateInstance
RdlEngineConfig.RegisterInstanceFactory(
    "AotInstanceHelpers.OrderFormatter",
    () => new OrderFormatter());

// Preserves the public methods so reflection can invoke them
RdlEngineConfig.RegisterType(
    "AotInstanceHelpers.OrderFormatter",
    typeof(OrderFormatter));
```

**RDL declaration:**

```xml
<Classes>
  <Class>
    <ClassName>AotInstanceHelpers.OrderFormatter</ClassName>
    <InstanceName>Helper</InstanceName>
  </Class>
</Classes>
```

**RDL expressions use the instance name** (`Helper`), not the class name:

```xml
<Value>=Helper.FormatDate(Fields!OrderDate.Value)</Value>
<Value>=Helper.FormatMoney(Fields!Amount.Value)</Value>
<BackgroundColor>=Helper.StatusColor(Fields!Status.Value)</BackgroundColor>
```

**The helper class:**

```csharp
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

The instance is created fresh per report execution. A new `OrderFormatter` is constructed each time the report runs.

> **Why two calls?** `RegisterInstanceFactory` handles _construction_ (bypasses `Assembly.CreateInstance`). `RegisterType` handles _method preservation_ (tells the trimmer which methods to keep). Both are needed for full trim safety.

**Full example:** `Examples/AotInstanceHelpers` — generates an order report with date formatting, currency formatting, and status-based conditional background colours driven by instance methods.

---

## API 4 — `RegisterCustomReportItem<T>` — AOT-safe CRI plugins

**Use when:** your report uses a custom report item (barcode, chart, etc.) that you have implemented in your own assembly.

The engine normally creates CRI instances via `Activator.CreateInstance(Type)` after scanning `RdlEngineConfig.xml`. Under AOT this fails because the type may not be preserved. Use `RegisterCustomReportItem<T>` instead:

```csharp
// Generic overload — the new() constraint makes the constructor reachable for the trimmer
RdlEngineConfig.RegisterCustomReportItem<MyBarcodeCri>("BarCode128");

// Factory overload — fully AOT-safe, no Type object required
RdlEngineConfig.RegisterCustomReportItem("BarCode128", () => new MyBarcodeCri());
```

The built-in barcode CRIs (`QrCode`, `BarCode128`, etc.) from `Majorsilence.Reporting.RdlCri.SkiaSharp` are already registered by `RdlEngineConfigInit()` and need no additional work.

---

## What is not (yet) AOT-compatible

A small number of features remain JIT-only and log an error if called under native AOT without the registration workaround:

| Feature | Reason |
|---|---|
| `<Code>` element without `RegisterCodeProvider` | Requires `VBCodeProvider` at runtime |
| `<CodeModules>` with no `RegisterType` | Loads assemblies from disk via `Assembly.LoadFrom` |
| OleDB data provider | OleDB is Windows-only and not AOT-compatible |

If you call one of these paths under native AOT you will receive a `PlatformNotSupportedException` with a message that explains which registration call to add.

---

## Putting it all together

A single application startup block can register everything at once:

```csharp
// Replace <Code> element VB with C# delegates
RdlEngineConfig.RegisterCodeProvider(report =>
    new RdlCodeFunctions()
        .Add("FormatCurrency", args => $"{Convert.ToDecimal(args[0]):C}")
        .Add("IsEligible",     args => (int)Convert.ToInt32(args[0]) >= 18)
);

// Preserve a static utility class used in expressions
RdlEngineConfig.RegisterType(
    "MyApp.ReportHelpers",
    typeof(ReportHelpers));

// Register instance class used via <Classes> element
RdlEngineConfig.RegisterInstanceFactory(
    "MyApp.OrderFormatter",
    () => new OrderFormatter());
RdlEngineConfig.RegisterType(
    "MyApp.OrderFormatter",
    typeof(OrderFormatter));

// Register a custom CRI plugin
RdlEngineConfig.RegisterCustomReportItem<MyQrCodeCri>("QrCode");

// Then initialise as usual
RdlEngineConfig.RdlEngineConfigInit();
```

All registrations are additive and backward-compatible. The same binary runs correctly on standard JIT runtimes — the engine simply uses the registered path when it is present and falls back to the traditional path when it is not.

---

## Further Reading

- [Quick Start Guide](/posts/quick-start) — the core render pipeline
- [Examples Cookbook](/posts/example-cookbook) — runnable patterns including the three AOT examples
- [GitHub Examples](https://github.com/majorsilence/Reporting/tree/main/Examples) — source for `AotCodeProvider`, `AotStaticHelpers`, `AotInstanceHelpers`
