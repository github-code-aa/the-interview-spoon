# B01 – .NET Runtime, Hosting & Middleware

**How to use:** Read each topic's *Study notes* first, then try to answer every ladder question aloud before opening the answer. Rungs go Easy → Follow-up → Medium → Hard → Tricky; stop where you get stuck and re-read.
Finish each topic with *Interview traps* and *Cheat sheet* as a last-minute recall list. Code targets C# 12 / .NET 8 unless noted.

## Table of Contents
1. Topic 1: .NET Framework vs .NET Core vs modern .NET vs .NET Standard
2. Topic 2: CLR, CTS, CLS, Assemblies, IL, JIT, GAC, Versioning
3. Topic 3: SDK vs Runtime, csproj, TFMs, NuGet, Central Package Management
4. Topic 4: Program.cs, WebApplication Builder, Generic Host
5. Topic 5: Kestrel, IIS Hosting, Reverse Proxy, HTTP/2 and HTTP/3
6. Topic 6: Request Pipeline & Middleware
7. Topic 7: Routing, Endpoint Routing & the Request Lifecycle
8. Topic 8: Minimal APIs vs Controllers
9. Topic 9: Static Files, Environments, launchSettings
10. Topic 10: App Lifetime, Graceful Shutdown, IHostedService
11. Topic 11: WebApplicationFactory & Integration Testing Hosting Model
12. Topic 12: HttpContext, Items, Features, IHttpContextAccessor
13. Topic 13: Filters vs Middleware; ASP.NET Core vs ASP.NET MVC 5
14. Topic 14: Assembly Loading & AssemblyLoadContext
15. Topic 15: Publishing Options (FDD, Self-contained, Single-file, Trimmed, AOT)
16. Topic 16: .NET 8 / .NET 9 Highlights

---

## Topic 1: .NET Framework vs .NET Core vs modern .NET vs .NET Standard

**Study notes**

- **.NET Framework** (2002–, last 4.8/4.8.1): Windows-only, machine-wide install, ships with Windows, in-place updates, includes WinForms/WPF/ASP.NET (System.Web)/WCF server/Web Forms. Maintenance mode (security fixes only).
- **.NET Core** (1.0 in 2016 → 3.1): cross-platform, open source, side-by-side install, modular (NuGet-delivered), high performance, ASP.NET Core.
- **.NET 5+** ("modern .NET"): the "Core" name was dropped; .NET 5 unified the line. Even versions are LTS (6, 8, 10: 3 years), odd are STS (now 18–24 months of support; STS was extended to 24 months starting with .NET 9).
- **.NET Standard**: a *contract* (API specification), not an implementation. 1.x–2.0/2.1. Needed so one library could target Framework + Core + Xamarin. .NET Standard 2.1 is NOT supported by .NET Framework. Today: prefer multi-targeting (`net8.0;netstandard2.0`) or just `net8.0`; use netstandard2.0 only when you must support .NET Framework.

```
.NET Standard 2.0 (API contract)
   ^ implemented by
.NET Framework 4.6.1+   .NET Core 2.0+   .NET 5+   Mono/Xamarin
```

| Aspect | .NET Framework | .NET (Core) 5+ |
|---|---|---|
| OS | Windows | Win/Linux/macOS |
| Install | Machine-wide, one per major | Side-by-side, per-app possible |
| Web | ASP.NET (System.Web, IIS) | ASP.NET Core (Kestrel) |
| Config | web.config / app.config | appsettings.json, env vars |
| Perf | Baseline | Much faster, active investment |
| New features | No | Yes |

### 1.1 [Easy] What is the difference between .NET Framework and .NET (Core)?
**Answer**
.NET Framework is the original Windows-only, machine-wide runtime in maintenance mode. Modern .NET (formerly .NET Core) is cross-platform, open source, side-by-side installable, faster, and where all new features land.

**Explanation**
Framework apps rely on a globally installed runtime that Windows Update patches. Modern .NET apps can carry their own runtime (self-contained) or use a shared one chosen by version. Many Framework-only technologies (Web Forms, WCF server, AppDomains for isolation, Remoting) are not in modern .NET.

**Code**
```csharp
using System.Runtime.InteropServices;

Console.WriteLine(RuntimeInformation.FrameworkDescription);
// Output (example): .NET 8.0.x
// On .NET Framework 4.8 it would print: .NET Framework 4.8.x
```

### 1.2 [Follow-up] What is .NET Standard and do you still need it?
**Answer**
.NET Standard is a versioned specification of APIs that all conforming runtimes must implement, letting one library binary run on several runtimes. It's mostly obsolete: new libraries target `net8.0` (or multi-target); `netstandard2.0` remains useful only to support .NET Framework consumers.

**Explanation**
Because it is only a contract, there is no ".NET Standard runtime". Starting with .NET 5 the unified BCL makes `netX.0` TFMs a superset. A library targeting `netstandard2.0` has a limited API surface (no Span-based newer APIs unless polyfilled by packages).

**Code**
```xml
<!-- Library supporting old and new consumers -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks>
    <LangVersion>12.0</LangVersion>
  </PropertyGroup>
</Project>
```

### 1.3 [Medium] How do you decide whether to migrate a .NET Framework app to modern .NET, and what blocks you?
**Answer**
Check dependencies (NuGet packages targeting netstandard/net6+), usage of System.Web, WCF server, Web Forms, AppDomains, Remoting, and registry/COM/GAC reliance. Use the .NET Upgrade Assistant and try-convert, and migrate incrementally (e.g., YARP strangler-fig, shared netstandard2.0 libraries).

**Explanation**
Typical order: move class libraries to `netstandard2.0` first, then the host app. System.Web has a compatibility shim (`System.Web` adapters) for incremental migration of ASP.NET apps. Config moves from web.config to `IConfiguration`. Windows-specific APIs remain available via `Microsoft.Windows.Compatibility` pack, but make the app Windows-only.

**Code**
```csharp
// Replace ConfigurationManager.AppSettings["Key"] usage
var builder = WebApplication.CreateBuilder(args);
string? key = builder.Configuration["Key"];
Console.WriteLine(key ?? "(not set)");
```

### 1.4 [Hard] Why can a library targeting netstandard2.1 not be used from .NET Framework 4.8, and what does that say about .NET Standard's design?
**Answer**
.NET Framework 4.8 implements at most .NET Standard 2.0, so a netstandard2.1 package fails restore (NU1202). .NET Standard 2.1 was designed for .NET Core 3.0+/Mono and the Framework team decided not to implement it, which effectively froze the "one standard" story at 2.0.

**Explanation**
This is why the guidance became: use `net8.0` for new work, `netstandard2.0` for max reach. Each TFM implies a set of netstandard versions (see the compat table); NuGet picks the "nearest" compatible asset from `lib/` folders. Multi-targeting with `#if NET8_0_OR_GREATER` lets you use new APIs when available.

**Gotchas / follow-ups**
- `netstandard2.0` consumers on .NET Framework may need binding redirects (auto-generated for SDK-style projects via `AutoGenerateBindingRedirects`).
- Follow-up: "What is a polyfill?" – packages like `Microsoft.Bcl.AsyncInterfaces` or `PolySharp` backport newer types/attributes.

**Code**
```csharp
public static class Hasher
{
    public static int Compute(string s)
    {
#if NET8_0_OR_GREATER
        return string.GetHashCode(s.AsSpan(), StringComparison.Ordinal);
#else
        return s.GetHashCode();
#endif
    }
}
```

### 1.5 [Tricky] An app on .NET Framework and a library on net8.0 — can the app reference the library?
**Answer**
No. A .NET Framework project cannot reference a `net8.0`-only project/package, because Framework cannot load assemblies compiled against the modern .NET reference assemblies (`System.Runtime` facades differ). The library must also target `netstandard2.0` or a `net4x` TFM.

**Explanation**
NuGet raises NU1201/NU1202 incompatibility. The opposite direction works: a net8.0 app can reference netstandard2.0 or even (with warnings, NU1701 "package restored using .NET Framework") net4x libraries — but this is unsupported-by-design and may fail at runtime if the library touches missing APIs.

**Gotchas / follow-ups**
- NU1701 is a warning you should treat seriously: it means "may not work".
- Process-level bridge when you can't migrate a dependency: out-of-process (gRPC / named pipes) wrapper.

**Code**
```xml
<!-- Allowed: net8.0 app uses a netstandard2.0 lib -->
<ItemGroup>
  <ProjectReference Include="..\Shared\Shared.csproj" />
</ItemGroup>
<!-- Shared.csproj: <TargetFramework>netstandard2.0</TargetFramework> -->
```

**Interview traps**
- ".NET Core" is not the current name; say ".NET 8" (or ".NET Core 3.1 and earlier").
- .NET Standard isn't a runtime; "install .NET Standard" is a nonsense phrase.
- STS vs LTS: even-numbered ≠ always LTS forever — check the support policy page.
- netstandard2.1 ≠ supported on .NET Framework.

**Cheat sheet**
- Framework = Windows, in-box, maintenance only.
- .NET 5+ = unified, cross-platform, side-by-side.
- netstandard2.0 for widest reach; `net8.0` for new libraries.
- Multi-target + `#if` for best of both.
- Net Framework can't reference net8.0-only libs.

---

## Topic 2: CLR, CTS, CLS, Assemblies, IL, JIT, GAC, Versioning

**Study notes**

- **CLR**: the execution engine – JIT, garbage collector, type loader, exception handling, security/verification, interop. In modern .NET it's CoreCLR.
- **CTS (Common Type System)**: defines types (value/reference, classes, interfaces, delegates, enums) and rules so languages interoperate.
- **CLS (Common Language Specification)**: a subset of CTS rules a library should follow to be consumable by all .NET languages (e.g., no unsigned public params, no case-only differences). Enforced via `[assembly: CLSCompliant(true)]`.
- **Assembly**: unit of deployment/versioning (.dll/.exe) containing IL, metadata, manifest (identity: name, version, culture, public key token), resources.
- **IL (CIL/MSIL)**: CPU-independent bytecode. **JIT** compiles IL to native per method at first call. Tiered compilation (default): tier-0 quick unoptimized → hot methods re-JITted at tier-1; OSR (on-stack replacement) for hot loops (.NET 7+); Dynamic PGO on by default in .NET 8. **ReadyToRun (R2R)** precompiles to reduce startup; **NativeAOT** removes the JIT entirely.
- **GAC**: machine-wide store for strong-named assemblies in .NET Framework (`%windir%\Microsoft.NET\assembly`). Doesn't exist in modern .NET (use deps.json/probing and NuGet).
- **Versioning**: `AssemblyVersion` (binding identity), `FileVersion` (Win32 file info), `InformationalVersion` (display, e.g., semver + commit).

```
C# source --Roslyn--> IL + metadata (.dll)
                         |
                  CLR loads assembly
                         |
              JIT (tier0 -> tier1) --> native code
```

### 2.1 [Easy] What does the CLR do, and what is IL?
**Answer**
The CLR is the runtime that loads assemblies, JIT-compiles IL to machine code, manages memory with the GC, and handles exceptions and type safety. IL is the CPU-independent intermediate language the C# compiler emits, which makes the same assembly portable across architectures/OSes.

**Explanation**
Compilation is two-stage: Roslyn produces IL + metadata; the JIT converts each method to native code on first call. This allows runtime-specific optimizations (e.g., using AVX2 if the CPU supports it).

**Code**
```csharp
using System.Reflection;

Assembly asm = typeof(string).Assembly;
Console.WriteLine(asm.GetName().Name);
// Output: System.Private.CoreLib
```

### 2.2 [Follow-up] Explain CTS vs CLS with an example.
**Answer**
CTS defines every type the runtime understands; CLS is the common subset that guarantees cross-language usability. For example `uint` is a valid CTS type but not CLS-compliant, so a public API exposing it may be unusable from some languages.

**Explanation**
Marking an assembly `CLSCompliant(true)` makes the compiler warn (CS3001/CS3002) when public members use non-compliant types or differ only by case. This matters for libraries consumed by VB.NET/F#; for private C# apps it's mostly irrelevant.

**Code**
```csharp
using System;

[assembly: CLSCompliant(true)]

public class Calc
{
    public int Add(int a, int b) => a + b;

    // warning CS3001: Argument type 'uint' is not CLS-compliant
    public uint AddU(uint a, uint b) => a + b;
}
```

### 2.3 [Medium] What is tiered compilation, and how do ReadyToRun and Dynamic PGO fit in?
**Answer**
Tiered compilation first JITs methods quickly with minimal optimization (tier-0) and, after a method is called ~30 times (and a short delay), re-JITs hot ones at tier-1 with full optimization. R2R images ship precompiled code to cut startup (treated like tier-0 and still promoted), and Dynamic PGO uses instrumentation in tier-0 to guide tier-1 (guarded devirtualization, hot/cold layout).

**Explanation**
Benefits: fast startup + good steady-state throughput. Loops with `OSR` can transition mid-execution. You can opt out with `<TieredCompilation>false</TieredCompilation>` or tune `TieredPGO`. Benchmarks must include warm-up for this reason.

**Code**
```csharp
using System.Diagnostics;

static long Work(int n) { long s = 0; for (int i = 0; i < n; i++) s += i % 7; return s; }

for (int round = 0; round < 3; round++)
{
    var sw = Stopwatch.StartNew();
    Work(50_000_000);
    Console.WriteLine($"round {round}: {sw.ElapsedMilliseconds} ms");
}
// Output: times vary; later rounds are typically faster as tier-1 kicks in.
```

### 2.4 [Hard] How do strong names, the GAC, and binding redirects work in .NET Framework, and what replaces them in .NET Core+?
**Answer**
In Framework, a strong-named assembly (name+version+culture+public key token) can be installed in the GAC; the loader binds by full identity, and binding redirects in config map old versions to new. Modern .NET ignores strong-name validation, has no GAC, and resolves assemblies via `*.deps.json`, the app directory, and the shared framework — with the "highest assembly version wins" rule handled at build time.

**Explanation**
Resolution in .NET Core: the host reads `app.runtimeconfig.json` (framework reference) and `app.deps.json` to build the TPA (Trusted Platform Assemblies) list. Version conflicts are generally resolved by choosing the higher version; mismatches show up as `FileLoadException` ("located assembly's manifest definition does not match"). Strong naming is kept only for identity, not security.

**Gotchas / follow-ups**
- Strong names are NOT a security boundary (don't be fooled by "signed").
- `AssemblyVersion` changes break binding in Framework; many libs keep it fixed (e.g., 1.0.0.0) and bump `FileVersion`.

**Code**
```csharp
using System.Reflection;

var name = typeof(Program).Assembly.GetName();
Console.WriteLine(name.Version);           // AssemblyVersion
var info = typeof(Program).Assembly
    .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?.InformationalVersion;
Console.WriteLine(info);                   // e.g. 1.2.3+abcdef (SourceLink adds commit)
```

### 2.5 [Tricky] If the JIT compiles per method on first call, why can the first request be slow and what can you do?
**Answer**
Each method on the request path must be JIT-compiled (plus type loading, static init, DI container building, and JSON serializer metadata generation) on first use. Mitigate with ReadyToRun, warm-up calls on startup, source-generated serializers, and tiered PGO/NativeAOT where applicable.

**Explanation**
Tier-0 JIT is fast but startup still adds up across thousands of framework methods; the BCL itself is R2R-compiled which is why trivial apps start quickly. Warm-up can be done in an `IHostedService` before marking readiness healthy. Also remember that Kubernetes/Azure probes should only pass after warm-up.

**Gotchas / follow-ups**
- Disabling tiered compilation can *increase* startup time.
- `TieredCompilationQuickJitForLoops` + OSR is on by default in .NET 7+.

**Code**
```csharp
using Microsoft.Extensions.Hosting;

public sealed class WarmUp : IHostedService
{
    public Task StartAsync(CancellationToken ct)
    {
        System.Text.Json.JsonSerializer.Serialize(new { a = 1 }); // primes serializer
        return Task.CompletedTask;
    }
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

**Interview traps**
- "The GAC exists in .NET Core" – false.
- Confusing `AssemblyVersion`, `FileVersion`, `InformationalVersion`.
- Saying IL is interpreted – it's JIT-compiled (or AOT-compiled), not interpreted.
- Believing strong naming provides security.

**Cheat sheet**
- Roslyn → IL → JIT → native.
- CTS = types; CLS = cross-language subset.
- Assembly identity = name+version+culture+token.
- Tier-0 → tier-1 after ~30 calls; PGO on by default in .NET 8.
- No GAC in .NET Core+; deps.json drives resolution.

---

## Topic 3: SDK vs Runtime, csproj, TFMs, NuGet, Central Package Management

**Study notes**

- **SDK** = compiler (Roslyn), MSBuild, CLI (`dotnet build/run/publish/test`), templates, plus runtimes. **Runtime** = what's needed to *run* apps. Runtimes: `Microsoft.NETCore.App` (base), `Microsoft.AspNetCore.App` (web), `Microsoft.WindowsDesktop.App`. Hosting bundle (Windows) = ASP.NET Core runtime + IIS module (ANCM).
- **global.json** pins SDK version (`rollForward` policy). SDK can build for older TFMs.
- **SDK-style csproj**: short; globbing includes `**/*.cs` automatically; `PackageReference` for NuGet; `ProjectReference` for projects.
- **TFM** examples: `net8.0`, `net8.0-windows`, `netstandard2.0`, `net48`. `ImplicitUsings`, `Nullable` are opt-in properties.
- **NuGet**: restore resolves transitive deps (lowest applicable version wins, "nearest wins" for conflicts), writes `obj/project.assets.json`; lock files via `RestorePackagesWithLockFile`.
- **Central Package Management (CPM)**: `Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` and `<PackageVersion Include=... Version=... />`; projects then use `<PackageReference Include="X" />` without Version.
- `Directory.Build.props/targets` share MSBuild properties across a repo.

### 3.1 [Easy] What's the difference between the .NET SDK and the runtime?
**Answer**
The SDK is for developers: compilers, CLI, MSBuild, templates. The runtime only executes already-built apps. Production servers usually need only the (ASP.NET Core) runtime.

**Explanation**
`dotnet --list-sdks` and `dotnet --list-runtimes` show what's installed. A "framework-dependent" app needs a matching runtime (major version matching, minor/patch roll forward).

**Code**
```csharp
// Run: dotnet --info   (shell)
// In code, the running runtime version:
Console.WriteLine(Environment.Version);
// Output (example): 8.0.x
```

### 3.2 [Follow-up] Show a modern SDK-style csproj and explain key properties.
**Answer**
It declares the SDK, `TargetFramework`, and optionally `Nullable` and `ImplicitUsings`; files are globbed automatically, packages come via `PackageReference`. This replaces old verbose csproj files that listed every file.

**Explanation**
`Microsoft.NET.Sdk.Web` adds web defaults (Kestrel, appsettings publishing, implicit usings for web namespaces). `InvariantGlobalization` removes ICU dependency (useful in slim containers). `TreatWarningsAsErrors`, `AnalysisLevel`, and `Deterministic` are common extras.

**Code**
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
  </ItemGroup>
</Project>
```

### 3.3 [Medium] How does Central Package Management work and why use it?
**Answer**
Put all versions in one `Directory.Packages.props`, enable `ManagePackageVersionsCentrally`, and projects reference packages without versions. It keeps versions consistent across a solution and makes upgrades a one-file change.

**Explanation**
Per-project overrides use `VersionOverride` (can be disabled). `GlobalPackageReference` adds a package to every project (e.g., analyzers). Transitive pinning (`CentralPackageTransitivePinningEnabled`) forces versions of transitive deps, useful for security fixes.

**Code**
```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
  </ItemGroup>
</Project>

<!-- Api.csproj -->
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" />
</ItemGroup>
```

### 3.4 [Hard] Two packages need different versions of the same transitive dependency. How does NuGet resolve it?
**Answer**
NuGet picks the lowest version that satisfies all constraints, with "nearest wins": a direct reference beats a transitive one. If the direct version is lower than a transitive minimum, you get a downgrade error (NU1605 as an error), so you raise the direct reference.

**Explanation**
Because dependency ranges mean "≥ version", the resolution is deterministic and avoids "latest" surprises. Direct references protect you but can mask incompatibility. At runtime only one version of an assembly is loaded per load context, so conflicts across major versions manifest as `MissingMethodException`/`FileLoadException`.

**Gotchas / follow-ups**
- `dotnet list package --include-transitive` and `dotnet nuget why` (newer SDKs) help diagnose.
- `dotnet list package --vulnerable` for CVEs; NuGet Audit is on by default from .NET 8 SDK.

**Code**
```xml
<!-- Pin a vulnerable transitive dependency explicitly -->
<ItemGroup>
  <PackageReference Include="System.Text.Encodings.Web" Version="8.0.0" />
</ItemGroup>
```

### 3.5 [Tricky] `global.json` says SDK 8.0.100 but only 8.0.300 is installed. What happens?
**Answer**
Default `rollForward` is `latestPatch`, which permits newer *patch* levels within the same feature band only (8.0.1xx), so 8.0.300 (a different feature band) is rejected and the build fails. Set `"rollForward": "latestFeature"` (or `latestMajor`) to allow it.

**Explanation**
The SDK version scheme is `major.minor.FBpp` (feature band hundreds digit + patch). `allowPrerelease` controls preview SDK selection. Failing to find a compatible SDK produces "A compatible .NET SDK was not found".

**Gotchas / follow-ups**
- Pinning SDK does not pin the *runtime* an app runs on; that's `runtimeconfig.json`/`TargetFramework`.

**Code**
```json
{
  "sdk": {
    "version": "8.0.100",
    "rollForward": "latestFeature"
  }
}
```

**Interview traps**
- Installing the SDK on prod unnecessarily; runtime/hosting bundle is enough.
- Forgetting the Hosting Bundle for IIS.
- Putting `Version` on a `PackageReference` when CPM is on (NU1008 error).
- Assuming NuGet picks the newest version – it picks the lowest satisfying.

**Cheat sheet**
- SDK builds; runtime runs.
- TFM = `net8.0`, `net8.0-windows`, `netstandard2.0`.
- CPM = `Directory.Packages.props` + `ManagePackageVersionsCentrally`.
- Nearest wins; lowest applicable version.
- `global.json` pins SDK with `rollForward`.

---

## Topic 4: Program.cs, WebApplication Builder, Generic Host

**Study notes**

- Since .NET 6: **minimal hosting model** – top-level `Program.cs` using `WebApplication.CreateBuilder(args)` replacing `Startup` (`ConfigureServices`/`Configure`). `Startup` still works with `Host.CreateDefaultBuilder().ConfigureWebHostDefaults`.
- `WebApplicationBuilder` exposes `Services`, `Configuration`, `Environment`, `Logging`, `Host`, `WebHost`. `builder.Build()` creates `WebApplication` which is both `IHost` and `IApplicationBuilder`/`IEndpointRouteBuilder`.
- **Generic Host** (`Microsoft.Extensions.Hosting`): DI, configuration, logging, lifetime, hosted services. Web = Generic Host + Kestrel (`GenerateWebHostBuilder`).
- Configuration sources order (later wins): appsettings.json → appsettings.{Env}.json → user secrets (Development) → environment variables → command line.
- Phases: build time (register services, DI container is built at `Build()`; modifying `Services` after Build throws) → run time (middleware pipeline).
- `WebApplication` auto-adds `UseRouting`/`UseEndpoints` implicitly (see Topic 7), `UseDeveloperExceptionPage` in Development, and `UseHttpsRedirection` isn't automatic.

### 4.1 [Easy] What does `WebApplication.CreateBuilder(args)` set up?
**Answer**
It configures defaults: Kestrel, configuration sources (appsettings, env vars, user secrets in Dev, command line), logging providers (console, debug, EventSource), DI with scope validation in Development, and the content root. `Build()` then creates the app.

**Explanation**
It's equivalent to the old `Host.CreateDefaultBuilder` + `ConfigureWebHostDefaults` combo, but streamlined. `ValidateScopes` and `ValidateOnBuild` are on in Development, catching captive dependency errors.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<TimeProvider>(TimeProvider.System);
var app = builder.Build();
app.MapGet("/", (TimeProvider tp) => tp.GetUtcNow());
app.Run();
```

### 4.2 [Follow-up] How do you map the old Startup class to Program.cs?
**Answer**
`ConfigureServices` becomes `builder.Services.*` calls before `Build()`; `Configure` becomes `app.Use*`/`app.Map*` calls after it. Environment checks use `app.Environment`.

**Explanation**
Both models are supported; you can even keep `Startup` via `builder.WebHost.UseStartup<Startup>()`, but that's discouraged for new code. With top-level statements, tests reference the entry point via `public partial class Program { }` (Topic 11).

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();           // was ConfigureServices
var app = builder.Build();
if (!app.Environment.IsDevelopment())
    app.UseExceptionHandler("/error");       // was Configure
app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

### 4.3 [Medium] How is the Generic Host used for non-web apps (worker services)?
**Answer**
`Host.CreateApplicationBuilder(args)` gives the same DI, config, and logging; add `AddHostedService<T>()` and `Run()`. It handles SIGTERM/Ctrl+C via `IHostApplicationLifetime` and graceful shutdown.

**Explanation**
Worker template uses `BackgroundService`. For Windows services use `AddWindowsService()`, and for systemd `AddSystemd()`. The web host is just a hosted service (`GenericWebHostService`) running Kestrel inside the generic host.

**Code**
```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<Ticker>();
await builder.Build().RunAsync();

sealed class Ticker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        while (await timer.WaitForNextTickAsync(stoppingToken))
            Console.WriteLine("tick");
    }
}
```

### 4.4 [Hard] What happens at `builder.Build()`, and why does modifying services afterward fail?
**Answer**
`Build()` finalizes the `IServiceCollection` and builds the `ServiceProvider` (the collection becomes read-only), merges configuration into the app's `IConfiguration`, and constructs the `WebApplication`. Mutating `builder.Services` afterwards throws `InvalidOperationException` because the builder is already consumed.

**Explanation**
Configuration is also frozen into the app's `ConfigurationManager`; `WebApplicationBuilder.Configuration` is a `ConfigurationManager` (both `IConfigurationBuilder` and `IConfigurationRoot`) so config values are readable *during* builder phase — handy for conditionally registering services. Note: values read early may not include sources added later.

**Gotchas / follow-ups**
- You can't resolve services during the build phase; to use a service while configuring, use options configuration with `IOptions` + `AddOptions<T>().Configure<TDep>(...)`.
- Building a temporary provider (`BuildServiceProvider()` in `ConfigureServices`) creates duplicate singletons — an anti-pattern (ASP0000 analyzer).

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
string? cs = builder.Configuration.GetConnectionString("Default"); // readable pre-Build
var app = builder.Build();
// builder.Services.AddSingleton<object>(); // would throw InvalidOperationException
app.Run();
```

### 4.5 [Tricky] In what order are configuration sources applied and what happens with `ASPNETCORE_ENVIRONMENT` and hierarchical keys?
**Answer**
Later sources override earlier: appsettings.json, appsettings.{Environment}.json, user secrets, environment variables, command-line arguments. Environment variables use `__` as the hierarchy separator (`ConnectionStrings__Default`), because `:` isn't valid in all shells.

**Explanation**
Arrays can be tricky: overriding `Items:0` via env var merges by index, not replacing the whole array. `ASPNETCORE_` and `DOTNET_` prefixed env vars configure the host itself (e.g., `ASPNETCORE_URLS`, `DOTNET_ENVIRONMENT`). `appsettings.json` content in `bin` is copied by default; `appsettings.{Env}.json` is selected by the environment name (case-insensitive on Windows, case-sensitive file names on Linux!).

**Gotchas / follow-ups**
- On Linux `appsettings.production.json` ≠ `appsettings.Production.json`.
- Azure App Service maps `:` to `__` for app settings.

**Code**
```csharp
// set ConnectionStrings__Default=Server=...
var builder = WebApplication.CreateBuilder(args);
Console.WriteLine(builder.Configuration["ConnectionStrings:Default"]);
// Output: Server=... (env var overrides appsettings.json)
```

**Interview traps**
- Calling `BuildServiceProvider()` inside registration code.
- Thinking `Startup` was removed (it still works).
- Registering services after `Build()`.
- Linux case-sensitivity on appsettings file names.

**Cheat sheet**
- `CreateBuilder` → register services → `Build()` → configure pipeline → `Run()`.
- `WebApplication` = IHost + IApplicationBuilder + IEndpointRouteBuilder.
- Config order: json → env-json → secrets → env vars → CLI.
- Env var hierarchy uses `__`.
- Worker = `Host.CreateApplicationBuilder` + `BackgroundService`.

---

## Topic 5: Kestrel, IIS Hosting, Reverse Proxy, HTTP/2 and HTTP/3

**Study notes**

- **Kestrel**: cross-platform managed web server, default for ASP.NET Core; can be edge-facing but commonly behind a reverse proxy (IIS, nginx, YARP, Azure Front Door/App Gateway) for TLS offload, load balancing, hardening.
- **IIS in-process**: app runs *inside* `w3wp.exe` using `IISHttpServer` (no Kestrel); best perf; ASP.NET Core Module V2 (ANCM) loads CLR. Default since 3.0. **Out-of-process**: IIS acts as reverse proxy to Kestrel in `dotnet.exe` process (loopback). **HTTP.sys**: Windows-only server (kernel-mode), for Windows auth edge scenarios/port sharing.
- Protocols: HTTP/1.1, HTTP/2 (TLS ALPN; default on in Kestrel), HTTP/3 (QUIC, .NET 7+ stable; needs `UseHttps` + OS support: msquic/libmsquic; Windows 11/Server 2022, Linux libmsquic). Enable via `ListenOptions.Protocols = HttpProtocols.Http1AndHttp2AndHttp3`.
- Forwarded headers: behind a proxy, use `UseForwardedHeaders` (`X-Forwarded-For/Proto/Host`) so `Request.Scheme`/`RemoteIpAddress` are right; must configure `KnownProxies`/`KnownNetworks` (security).
- Limits: `MaxRequestBodySize` (default 30,000,000 bytes ≈ 28.6 MB), request headers timeout, min data rates.

```
Client --TLS--> [IIS / nginx / YARP / LB] --HTTP--> Kestrel --> app
Client --------> IIS (w3wp.exe: app runs in-proc, IISHttpServer)
```

### 5.1 [Easy] What is Kestrel and why is it used?
**Answer**
Kestrel is ASP.NET Core's default, cross-platform, high-performance web server built on managed sockets. It's used everywhere (Windows, Linux, containers) and can be exposed directly or behind a reverse proxy.

**Explanation**
Kestrel handles HTTP parsing, connections, TLS, and HTTP/2/3. It's not tied to IIS, which is why ASP.NET Core runs on Linux containers.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureKestrel(o =>
{
    o.ListenAnyIP(5000);
    o.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
});
var app = builder.Build();
app.MapGet("/", () => "hello");
app.Run();
```

### 5.2 [Follow-up] Compare IIS in-process vs out-of-process hosting.
**Answer**
In-process hosts the app inside the IIS worker process (w3wp.exe) via `IISHttpServer`, avoiding a proxy hop and giving significantly better throughput. Out-of-process runs Kestrel in a separate `dotnet.exe` and IIS forwards requests over HTTP.

**Explanation**
In-process: one app per app pool; `Process.GetCurrentProcess().ProcessName` is `w3wp`; Kestrel config is ignored. Out-of-process: needed when you want Kestrel specifics or a different bitness. Set via `<AspNetCoreHostingModel>` / `hostingModel` attribute in web.config (`InProcess` / `OutOfProcess`).

**Code**
```xml
<!-- csproj -->
<PropertyGroup>
  <AspNetCoreHostingModel>InProcess</AspNetCoreHostingModel>
</PropertyGroup>
```
```csharp
// Which server is active?
var app = WebApplication.Create(args);
app.MapGet("/server", (IServiceProvider sp) =>
    sp.GetRequiredService<Microsoft.AspNetCore.Hosting.Server.IServer>().GetType().Name);
// Output (in-proc IIS): IISHttpServer ; (Kestrel): KestrelServer
app.Run();
```

### 5.3 [Medium] Why use a reverse proxy in front of Kestrel, and what must you configure?
**Answer**
For TLS termination, load balancing, request filtering, static caching, and an extra security layer. You must configure forwarded headers so the app sees the original client IP and scheme, and restrict trusted proxies.

**Explanation**
Without it, `Request.IsHttps` is false and redirects/links generate `http://`. In containers/hosted platforms the `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true` env var enables the middleware (auto-enabled on Azure App Service/IIS integration). Place `UseForwardedHeaders` **first** in the pipeline.

**Code**
```csharp
using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownProxies.Add(System.Net.IPAddress.Parse("10.0.0.4"));
});
var app = builder.Build();
app.UseForwardedHeaders();   // before auth, redirection, etc.
app.MapGet("/ip", (HttpContext c) => c.Connection.RemoteIpAddress?.ToString());
app.Run();
```

### 5.4 [Hard] How do HTTP/2 and HTTP/3 work in Kestrel, and what are the prerequisites?
**Answer**
HTTP/2 multiplexes streams over one TCP+TLS connection (negotiated via ALPN) and is on by default over HTTPS. HTTP/3 runs over QUIC/UDP, requires TLS 1.3, OS-level msquic support, and an `Alt-Svc` header so clients upgrade; enable with `HttpProtocols.Http1AndHttp2AndHttp3`.

**Explanation**
HTTP/2 over cleartext (h2c) needs prior-knowledge and an HTTP/2-only endpoint (gRPC without TLS). Mixed `Http1AndHttp2` on a non-TLS port defaults to HTTP/1.1 only. HTTP/3 removes TCP head-of-line blocking; the first request is always H1/H2, then the server advertises h3 via `Alt-Svc`. Reverse proxies terminating TLS may downgrade to HTTP/1.1 toward Kestrel.

**Gotchas / follow-ups**
- Firewalls must allow UDP 443 for HTTP/3.
- In-process IIS supports HTTP/2 (IIS-managed); HTTP/3 support depends on IIS version.

**Code**
```csharp
using Microsoft.AspNetCore.Server.Kestrel.Core;

var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureKestrel(o =>
{
    o.ConfigureEndpointDefaults(lo =>
        lo.Protocols = HttpProtocols.Http1AndHttp2AndHttp3);
});
var app = builder.Build();
app.MapGet("/p", (HttpRequest r) => r.Protocol);  // e.g. HTTP/2 or HTTP/3
app.Run();
```

### 5.5 [Tricky] A 50 MB upload fails with 413 behind IIS but works locally on Kestrel. Why?
**Answer**
Each layer has its own limit: Kestrel's `MaxRequestBodySize` (~28.6 MB default), IIS's `maxAllowedContentLength` (30,000,000 bytes default) in web.config, and form options (`MultipartBodyLengthLimit` 128 MB). Raise all that apply.

**Explanation**
In-process IIS ignores Kestrel config, so `IISServerOptions.MaxRequestBodySize` applies in addition to IIS request filtering. The per-endpoint attributes `[RequestSizeLimit]` and `[DisableRequestSizeLimit]` override Kestrel/IIS server option limits (but not IIS request filtering `maxAllowedContentLength`).

**Gotchas / follow-ups**
- Multipart forms: also check `FormOptions`.
- Streaming uploads (don't buffer) avoid memory blowups.

**Code**
```csharp
builder.Services.Configure<IISServerOptions>(o => o.MaxRequestBodySize = 100_000_000);
builder.Services.Configure<Microsoft.AspNetCore.Http.Features.FormOptions>(o =>
    o.MultipartBodyLengthLimit = 100_000_000);
// web.config: <requestLimits maxAllowedContentLength="100000000" />
```

**Interview traps**
- "Kestrel isn't production ready" – outdated since 2.0.
- Forgetting `UseForwardedHeaders` order and trusted proxies.
- In-process IIS ignores `ConfigureKestrel`.
- HTTP/3 needs UDP and QUIC support.

**Cheat sheet**
- Kestrel = default cross-plat server.
- IIS in-proc = IISHttpServer in w3wp (fast); out-of-proc = proxy to Kestrel.
- Behind proxy → forwarded headers first.
- H2 via ALPN; H3 via QUIC + Alt-Svc.
- Body limits exist at Kestrel, IIS, and form layers.

---

## Topic 6: Request Pipeline & Middleware

**Study notes**

- Pipeline = chain of `RequestDelegate` (`HttpContext → Task`). Each middleware receives `next`, can act before/after, or short-circuit by not calling `next`.
- `app.Use(...)` adds middleware that may call next; `app.Run(...)` terminal (no next); `app.Map(path, branch)` branches by path prefix (strips prefix to `PathBase`); `MapWhen(predicate, branch)` branches by condition; `UseWhen` branches and **rejoins** the main pipeline.
- **Order matters**; recommended: ExceptionHandler → HSTS → HttpsRedirection → StaticFiles → Routing → CORS → Authentication → Authorization → (RateLimiter, ResponseCaching/OutputCache as needed) → custom → endpoints.
- **Conventional middleware**: class with ctor taking `RequestDelegate next` (+ singleton-lifetime deps) and `InvokeAsync(HttpContext, ...)` taking scoped deps as method params. Instance is created **once** (singleton).
- **Factory-based `IMiddleware`**: implement `InvokeAsync(HttpContext, RequestDelegate)`; resolved from DI per request (`IMiddlewareFactory`), so lifetime is whatever you register (must register in DI, e.g., scoped/transient).

```
Request -> [ExceptionHandler] -> [HttpsRedirection] -> [StaticFiles] -> [Routing]
        -> [Auth] -> [Authorization] -> [Endpoint] 
Response <- (flows back through same middleware in reverse)
```

### 6.1 [Easy] What is middleware, and how do Use and Run differ?
**Answer**
Middleware is a component in the HTTP pipeline that handles a request and optionally passes it on. `Use` can call `next` to continue; `Run` is terminal and never calls next.

**Explanation**
Code before `await next()` runs on the way in, code after runs on the way out, so middleware can wrap the rest of the pipeline (timing, error handling, logging).

**Code**
```csharp
var app = WebApplication.Create(args);

app.Use(async (ctx, next) =>
{
    Console.WriteLine("1 in");
    await next(ctx);
    Console.WriteLine("1 out");
});
app.Run(async ctx =>
{
    Console.WriteLine("terminal");
    await ctx.Response.WriteAsync("hi");
});
app.Run();
// Output for one request:
// 1 in
// terminal
// 1 out
```

### 6.2 [Follow-up] Why does order matter? Give an example of a wrong order.
**Answer**
Each middleware only sees what earlier ones have set up and only affects later ones. For instance, `UseAuthorization` before `UseAuthentication` means no user is identified, so everything looks anonymous and protected endpoints return 401/403.

**Explanation**
Other classic errors: `UseCors` after endpoint execution (CORS headers missing), exception handler placed late (misses exceptions from earlier components), `UseStaticFiles` after auth (static files protected unintentionally), `UseRouting` after `UseAuthorization` (authorization can't see endpoint metadata). Note with `WebApplication`, routing is added automatically at the start unless you call `UseRouting` yourself to position it.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();
builder.Services.AddCors();
var app = builder.Build();

app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.MapGet("/secure", () => "ok").RequireAuthorization();
app.Run();
```

### 6.3 [Medium] Write custom middleware both conventionally and with IMiddleware. How do they differ?
**Answer**
Conventional middleware is constructed once with `RequestDelegate`, uses a method named `Invoke`/`InvokeAsync`, and gets scoped services via method parameters. `IMiddleware` is DI-activated per request (lifetime you choose), so it can take scoped services in its constructor, but it must be registered in the container.

**Explanation**
Injecting a scoped service into a *conventional* middleware constructor causes a captive dependency (error in Development with scope validation). `IMiddleware` doesn't take `next` in its ctor, which makes it testable and strongly typed. Conventional is slightly faster/less allocation since no factory resolves it.

**Code**
```csharp
using Microsoft.AspNetCore.Http;

public sealed class TimingMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext ctx, ILogger<TimingMiddleware> log /* scoped/transient OK here */)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        await next(ctx);
        log.LogInformation("{Path} took {Ms} ms", ctx.Request.Path, sw.ElapsedMilliseconds);
    }
}

public sealed class CorrelationMiddleware : IMiddleware
{
    public Task InvokeAsync(HttpContext ctx, RequestDelegate next)
    {
        ctx.Response.Headers["X-Correlation-Id"] = Guid.NewGuid().ToString();
        return next(ctx);
    }
}

// Program.cs
// builder.Services.AddTransient<CorrelationMiddleware>();   // required for IMiddleware
// app.UseMiddleware<TimingMiddleware>();
// app.UseMiddleware<CorrelationMiddleware>();
```

### 6.4 [Hard] Explain Map, MapWhen, and UseWhen, and what happens to the path.
**Answer**
`Map` creates a branch for a path prefix, moving the matched prefix into `PathBase` and not rejoining. `MapWhen` branches on a predicate without altering the path and doesn't rejoin. `UseWhen` branches on a predicate but **rejoins** the main pipeline if the branch doesn't short-circuit.

**Explanation**
Branches are built with their own `IApplicationBuilder` (`ApplicationBuilder.New()`), so middleware registered in the branch isn't in the main pipeline. Since `Map` modifies `Path`/`PathBase`, link generation and static files can behave differently; after the branch, original values are restored. Prefer endpoint routing for route-based logic and `UseWhen` for conditional cross-cutting middleware.

**Gotchas / follow-ups**
- Anything registered after `Map` in the main pipeline never runs for matched requests.
- `app.Map("/a", ...)` also matches `/a/b` but not `/ab`.

**Code**
```csharp
var app = WebApplication.Create(args);

app.UseWhen(c => c.Request.Path.StartsWithSegments("/api"),
    branch => branch.Use((ctx, next) =>
    {
        ctx.Response.Headers["X-Api"] = "1";
        return next(ctx);   // rejoins main pipeline
    }));

app.Map("/health", b => b.Run(c => c.Response.WriteAsync("ok")));   // terminal branch

app.MapGet("/api/x", () => "x");
app.Run();
```

### 6.5 [Tricky] Why does setting a response header after `await next()` sometimes throw or get ignored?
**Answer**
Once the response has started (headers sent, `Response.HasStarted == true`), you can't modify status code or headers; attempts throw `InvalidOperationException` or are silently ignored. Use `Response.OnStarting` to set headers just before they're flushed.

**Explanation**
Any write to `Response.Body` larger than the buffer (or `FlushAsync`) starts the response. Middleware that wants to add headers after the endpoint ran must register `OnStarting` callbacks **before** calling next. Also never write to the response in middleware and then call `next`, as downstream may attempt headers.

**Gotchas / follow-ups**
- Exception handler middleware can't change a response that already started; it will just abort the connection.
- Don't capture `HttpContext` in callbacks beyond the request (use the state parameter).

**Code**
```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(state =>
    {
        var c = (HttpContext)state;
        c.Response.Headers["X-Served-By"] = "api-1";
        return Task.CompletedTask;
    }, ctx);

    await next(ctx);
    // ctx.Response.Headers["X-Late"] = "1"; // throws if HasStarted
});
```

**Interview traps**
- Injecting scoped services into a conventional middleware **constructor**.
- Forgetting to `AddTransient<MyIMiddleware>()` (runtime InvalidOperationException).
- Not calling `await next()` unintentionally (short-circuits).
- Writing to the response, then calling next.

**Cheat sheet**
- `Use` = continue, `Run` = terminal, `Map` = path branch, `UseWhen` = conditional + rejoin.
- Order: Exception → HTTPS → Static → Routing → CORS → AuthN → AuthZ → Endpoint.
- Conventional = singleton, scoped deps via `InvokeAsync` params.
- `IMiddleware` = DI-resolved per request, must be registered.
- `OnStarting` for late headers.

---

## Topic 7: Routing, Endpoint Routing & the Request Lifecycle

**Study notes**

- **Endpoint routing** (since 3.0): routing is split into **matching** (`UseRouting` → `EndpointRoutingMiddleware` selects an `Endpoint` and sets it on `HttpContext.GetEndpoint()`) and **execution** (`UseEndpoints`/`MapXxx` → `EndpointMiddleware` runs `endpoint.RequestDelegate`). Middleware in between (CORS, authn/authz, rate limiting) can read endpoint **metadata** (`[Authorize]`, `RequireCors`, etc.).
- In `WebApplication`, `UseRouting`/`UseEndpoints` are added automatically (routing at the start of the pipeline, endpoints at the end), unless you call `UseRouting` explicitly.
- Route templates: `{id}`, `{id:int}`, `{name?}`, `{*path}` catch-all, constraints (`int`, `guid`, `min(1)`, `regex`), defaults, `{controller=Home}`. Matching uses a DFA-based matcher; ambiguity → `AmbiguousMatchException`. Precedence: literal segments > parameters with constraints > unconstrained > catch-all.
- Attribute routing (controllers `[Route]`, `[HttpGet]`) vs conventional (`MapControllerRoute`) vs minimal (`MapGet`).
- Link generation: `LinkGenerator`, `Url.Action`, `TypedResults`' `Results.CreatedAtRoute`.

**End-to-end lifecycle (controller request)**
```
Socket -> Kestrel (parse HTTP) -> HttpContext created (IHttpApplication)
 -> Middleware pipeline: Exception -> HTTPS -> Static -> Routing(match)
    -> CORS -> AuthN -> AuthZ -> (other mw) -> EndpointMiddleware
 -> MVC: Authorization filter -> Resource filter -> Model binding & validation
    -> Action filter -> Action method -> Action result filter -> Result execution
    (-> exception filter on error)
 -> Response written -> middleware unwinds -> Kestrel sends -> HttpContext disposed/pooled
```

### 7.1 [Easy] What is endpoint routing?
**Answer**
Endpoint routing selects which endpoint (controller action, minimal API handler, Razor page, etc.) matches the request in one middleware, and executes it in a later one. This lets middleware in between inspect the chosen endpoint's metadata.

**Explanation**
That's how `UseAuthorization` knows the endpoint has `[Authorize]`. Before 3.0, MVC's routing was internal to MVC middleware, so earlier middleware couldn't know the target action.

**Code**
```csharp
var app = WebApplication.Create(args);
app.Use(async (ctx, next) =>
{
    // Works because routing ran earlier (auto-added by WebApplication)
    var ep = ctx.GetEndpoint();
    Console.WriteLine(ep?.DisplayName);
    await next(ctx);
});
app.MapGet("/hello", () => "hi").WithName("hello");
app.Run();
// Output (GET /hello): HTTP: GET /hello
```

### 7.2 [Follow-up] What route constraints exist and what happens if none match?
**Answer**
Constraints like `{id:int}`, `:guid`, `:min(1)`, `:regex(...)`, `:alpha` restrict matching; a non-matching constraint means the route doesn't match, so the request falls through to other endpoints or returns 404. Constraints aren't validation — they select routes, not produce 400s.

**Explanation**
Use model validation / `[FromRoute]` + validation for business rules. Overlapping templates like `/users/{id:int}` and `/users/{name}` are fine because constrained routes have higher precedence. Two equal-precedence matches → `AmbiguousMatchException` (HTTP 500).

**Code**
```csharp
app.MapGet("/users/{id:int}", (int id) => $"id {id}");
app.MapGet("/users/{name}", (string name) => $"name {name}");
// GET /users/5     -> id 5
// GET /users/bob   -> name bob
```

### 7.3 [Medium] Walk through a request's lifecycle in an ASP.NET Core MVC app.
**Answer**
Kestrel parses the request and creates `HttpContext`; it flows through middleware (exception handler, HTTPS, static files, routing, CORS, authN/authZ). `EndpointMiddleware` invokes the MVC action invoker: authorization → resource filters → model binding/validation → action filters → action → result filters → result execution (e.g., JSON formatting), then the response unwinds back through middleware.

**Explanation**
Exception filters wrap action/result execution. Short-circuiting is possible at each stage (e.g., returning early from an auth filter). Output formatting uses content negotiation (`Accept` header → formatter). DI scope is created per request (`RequestServices`) at the start of hosting and disposed at the end.

**Code**
```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController, Route("api/[controller]")]
public class ItemsController : ControllerBase
{
    [HttpGet("{id:int}")]
    public ActionResult<object> Get(int id) =>
        id > 0 ? Ok(new { id }) : NotFound();
}
```

### 7.4 [Hard] How does the endpoint matcher decide between candidates, and how can you customize it?
**Answer**
The DFA matcher builds a state machine from literal segments, then filters candidates by order (`Order`), precedence (literals > constrained params > params > catch-all), then policies like HTTP method (`HttpMethodMatcherPolicy`) and host. Customize via `MatcherPolicy` / `IEndpointSelectorPolicy` or `DynamicRouteValueTransformer`.

**Explanation**
Ties after all of this produce `AmbiguousMatchException` listing candidates. `Order` property in attribute routing can break ties. `RouteGroupBuilder` (`MapGroup`) allows prefixing and applying metadata/filters to groups of minimal API endpoints. Conventional routes and attribute routes can coexist but attribute routes on a controller disable conventional matching for that action.

**Gotchas / follow-ups**
- Case-insensitive route matching by default.
- Trailing slash: `/x` and `/x/` both match by default.

**Code**
```csharp
var api = app.MapGroup("/api/v1").RequireAuthorization();
api.MapGet("/orders/{id:guid}", (Guid id) => Results.Ok(id)).WithName("GetOrder");

app.MapGet("/link", (LinkGenerator lg, HttpContext ctx) =>
    lg.GetPathByName(ctx, "GetOrder", new { id = Guid.Empty }));
// Output: /api/v1/orders/00000000-0000-0000-0000-000000000000
```

### 7.5 [Tricky] Why might `app.UseAuthorization()` ignore `[Authorize]` on a minimal API/controller, and what does `UseRouting` placement change?
**Answer**
Authorization middleware relies on the endpoint selected by routing; if `UseAuthorization` runs before routing has selected the endpoint (e.g., you manually call `UseRouting` after it), no authorization metadata is seen and the request is allowed — or you get an exception telling you to move it. Authentication/authorization must be between `UseRouting` and the endpoints.

**Explanation**
With `WebApplication`, routing is auto-inserted at the pipeline start, so if you never call `UseRouting` manually the default placement is correct. If you call it explicitly, any middleware before it won't see an endpoint. Fallback policy (`SetFallbackPolicy`) can enforce auth for endpoints without metadata.

**Gotchas / follow-ups**
- `Short-circuit` endpoints (`ShortCircuit()` in .NET 8) skip the remaining middleware after routing.
- Static files served before routing aren't subject to authorization (including the fallback policy).

**Code**
```csharp
builder.Services.AddAuthorization(o =>
    o.FallbackPolicy = new Microsoft.AspNetCore.Authorization.AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser().Build());
// Now every endpoint requires auth unless [AllowAnonymous]/.AllowAnonymous()
app.MapGet("/public", () => "open").AllowAnonymous();
```

**Interview traps**
- Believing routing and endpoint execution are the same middleware.
- Treating constraints as validation (they yield 404, not 400).
- `UseAuthorization` placed before routing when `UseRouting` is called explicitly.
- Forgetting that `Map` branch and endpoint routing differ.

**Cheat sheet**
- Match (UseRouting) ≠ Execute (UseEndpoints).
- `HttpContext.GetEndpoint()` exposes metadata.
- Precedence: literal > constrained > unconstrained > catch-all.
- Lifecycle: Kestrel → middleware → MVC filters → action → result → unwind.
- `MapGroup` = shared prefix + metadata.

---

## Topic 8: Minimal APIs vs Controllers

**Study notes**

- **Minimal APIs** (6+): `MapGet/MapPost/...` with lambdas/method groups; parameter binding from route/query/header/body/services (inferred), `TypedResults`/`Results`, endpoint filters (`AddEndpointFilter`), `MapGroup`, OpenAPI via `WithOpenApi`/Swashbuckle. .NET 7 added filters & groups; .NET 8 added form binding with antiforgery and `[AsParameters]` improvements, `ShortCircuit`, Native AOT-friendly request delegate generator.
- **Controllers**: class-based MVC with attributes, filters pipeline, model binding/validation (`[ApiController]` auto 400 + ProblemDetails), content negotiation, formatters, conventions, `ApplicationParts`.
- Validation: controllers have built-in `ModelState` + DataAnnotations; minimal APIs have no built-in validation in .NET 8 (use filters/FluentValidation; built-in `AddValidation` arrived in .NET 10).
- Perf: minimal APIs have less overhead (no MVC filter pipeline, model binding sophistication), and work with NativeAOT. Controllers aren't AOT-ready (reflection-heavy).
- Choice is organizational, not binary: both can coexist in the same app.

| Feature | Minimal API | Controllers |
|---|---|---|
| Boilerplate | Low | Higher |
| Filters | Endpoint filters | Full MVC filter pipeline |
| Validation | Manual/library | Built-in |
| Content negotiation | JSON by default | Full |
| NativeAOT | Supported | Not supported |

### 8.1 [Easy] What are minimal APIs and how do they differ from controllers?
**Answer**
Minimal APIs define HTTP endpoints directly with lambdas/methods on `WebApplication` with little ceremony. Controllers are classes with attributes and the full MVC pipeline (filters, model validation, formatters).

**Explanation**
Both use endpoint routing and share middleware. Minimal APIs infer parameter sources, and return types convert to responses automatically (string → text, object → JSON, `IResult` → custom).

**Code**
```csharp
var app = WebApplication.Create(args);
app.MapGet("/todos/{id}", (int id) => TypedResults.Ok(new { id, title = "x" }));
app.MapPost("/todos", (Todo t) => TypedResults.Created($"/todos/{t.Id}", t));
app.Run();

record Todo(int Id, string Title);
```

### 8.2 [Follow-up] How does minimal API parameter binding decide where a parameter comes from?
**Answer**
Order of inference: explicit attribute (`[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromServices]`, `[FromForm]`) → special types (`HttpContext`, `HttpRequest`, `CancellationToken`, `ClaimsPrincipal`) → route parameter by name → services registered in DI (inferred) → simple types from query → complex type from body (JSON) for POST/PUT/PATCH.

**Explanation**
If it's ambiguous or can't be inferred (e.g., a complex type on GET) startup throws. Use `[AsParameters]` to bind a parameter object. Parameters that are nullable/optional are optional; required missing values return 400. Only one body parameter allowed.

**Code**
```csharp
app.MapGet("/search", ([AsParameters] SearchArgs a, ILogger<Program> log) =>
{
    log.LogInformation("q={Q}", a.Q);
    return Results.Ok(a);
});

record SearchArgs(string Q, int Page = 1);
```

### 8.3 [Medium] How do endpoint filters work, and how would you add validation to a minimal API?
**Answer**
`AddEndpointFilter` wraps the handler with an `EndpointFilterDelegate` — you get `EndpointFilterInvocationContext` (arguments, HttpContext), call `next`, and can short-circuit with a result. Validation is commonly implemented as a reusable filter that inspects arguments with DataAnnotations or FluentValidation.

**Explanation**
Filters run after routing/binding and are scoped to the endpoint or `MapGroup`. They're distinct from MVC filters and middleware. Order of registration = order of execution (outermost first).

**Code**
```csharp
using System.ComponentModel.DataAnnotations;

app.MapPost("/people", (Person p) => Results.Ok(p))
   .AddEndpointFilter(async (ctx, next) =>
   {
       var p = ctx.GetArgument<Person>(0);
       var results = new List<ValidationResult>();
       if (!Validator.TryValidateObject(p, new ValidationContext(p), results, true))
           return Results.ValidationProblem(
               results.ToDictionary(r => r.MemberNames.FirstOrDefault() ?? "", r => new[] { r.ErrorMessage ?? "" }));
       return await next(ctx);
   });

record Person([property: Required] string Name);
```

### 8.4 [Hard] Why are minimal APIs AOT-friendly and controllers not? What is the Request Delegate Generator?
**Answer**
Controllers rely on runtime reflection for discovery, model binding, and filter creation, which the trimmer/AOT can't statically analyze. For minimal APIs, .NET 8's Request Delegate Generator (RDG, a source generator) emits compile-time code for binding and invocation, avoiding `Expression` tree compilation at runtime; JSON uses source-generated `JsonSerializerContext`.

**Explanation**
`WebApplication.CreateSlimBuilder` / `CreateEmptyBuilder` reduce default features for AOT. Without RDG, minimal APIs use reflection plus expression trees compiled at first call (slow start). With AOT, you must register a `JsonSerializerContext` in `JsonOptions.SerializerOptions.TypeInfoResolverChain`.

**Gotchas / follow-ups**
- Anonymous types don't work with source-generated JSON; use named records.
- `EnableRequestDelegateGenerator` is automatically on with `PublishAot`.

**Code**
```csharp
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
var app = builder.Build();
app.MapGet("/t", () => new Todo(1, "x"));
app.Run();

record Todo(int Id, string Title);

[JsonSerializable(typeof(Todo))]
partial class AppJsonContext : JsonSerializerContext { }
```

### 8.5 [Tricky] A minimal API handler with a complex-type parameter on a GET returns 500/throws at startup. Why?
**Answer**
On GET (and DELETE etc.) a complex type isn't inferred from the body, since those methods typically have no body; the framework can't infer a source and throws at startup (an `InvalidOperationException` saying body inference isn't allowed for that method). Use `[AsParameters]`, `[FromBody]` explicitly (discouraged on GET), or bind from the query.

**Explanation**
Inference is `HttpMethod`-aware. Also, a type with `TryParse`/`BindAsync` static methods is treated as a bindable custom type from route/query. A service parameter that's not registered in DI is a common 500 ("Failure to infer how to bind parameter"), since it falls back to body inference.

**Gotchas / follow-ups**
- Mark DI params with `[FromServices]` to get clear errors.

**Code**
```csharp
// Throws at startup (InvalidOperationException) for GET:
// app.MapGet("/bad", (Filter f) => f);

app.MapGet("/ok", ([AsParameters] Filter f) => f);
record Filter(string? Name, int Skip = 0);
```

**Interview traps**
- "Minimal APIs can't do auth/filters/validation" – they can; validation is just manual in .NET 8.
- Only one `[FromBody]` per handler.
- Anonymous types with source-generated JSON/AOT.
- Unregistered service param is inferred as body → confusing error.

**Cheat sheet**
- Minimal: lambdas, `TypedResults`, endpoint filters, `MapGroup`.
- Controllers: MVC filters, `[ApiController]` auto-400, content negotiation.
- Both share routing and middleware; can coexist.
- AOT → minimal APIs + RDG + JSON source generator.
- Use `[AsParameters]` to bundle params.

---

## Topic 9: Static Files, Environments, launchSettings

**Study notes**

- **Static files**: `app.UseStaticFiles()` serves from **web root** (`wwwroot` by default); content root is the app's base folder. Not served by default from other folders — configure `StaticFileOptions` with a `FileProvider`. `UseDefaultFiles()` only rewrites `/` to `index.html` and must come **before** `UseStaticFiles`; `UseFileServer()` combines static files + default files (+ optional directory browsing). Unknown extensions return 404 unless `ServeUnknownFileTypes`; map types with `FileExtensionContentTypeProvider`.
- **Environment**: `ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT`), default `Production`. Helpers: `IsDevelopment()`, `IsStaging()`, `IsProduction()`, `IsEnvironment("X")`. Drives `appsettings.{Env}.json`, developer exception page, user secrets (Development only by default).
- **launchSettings.json** (`Properties/`): used only by `dotnet run` (profile selection) and Visual Studio/IDEs; **not published** and not read in production. Holds `commandName` (Project/IISExpress), `applicationUrl`, `environmentVariables`, `launchBrowser`. `dotnet run --no-launch-profile` ignores it.
- Production env vars: set via OS, container env, App Service settings, web.config `<environmentVariables>`.

### 9.1 [Easy] How do you serve static files in ASP.NET Core?
**Answer**
Call `app.UseStaticFiles()`; files under `wwwroot` become available at the site root. To serve a default page use `UseDefaultFiles()` before it, or `UseFileServer()`.

**Explanation**
Static files middleware short-circuits for matched files, so place it early (before auth) if files are public, or serve protected files through an endpoint with authorization.

**Code**
```csharp
var app = WebApplication.Create(args);
app.UseDefaultFiles();   // maps "/" -> "/index.html" (must precede UseStaticFiles)
app.UseStaticFiles();    // serves wwwroot/*
app.Run();
```

### 9.2 [Follow-up] How do you serve files from outside wwwroot or set cache headers?
**Answer**
Use `StaticFileOptions` with a `PhysicalFileProvider` and `RequestPath`, and `OnPrepareResponse` to add caching headers.

**Explanation**
Be careful exposing directories: only files under the provider root are accessible, and directory browsing is off unless you enable it. For versioned assets, use long `Cache-Control` with `asp-append-version`/content hashes.

**Code**
```csharp
using Microsoft.Extensions.FileProviders;

var app = WebApplication.Create(args);
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(Path.Combine(app.Environment.ContentRootPath, "assets")),
    RequestPath = "/assets",
    OnPrepareResponse = ctx =>
        ctx.Context.Response.Headers.CacheControl = "public,max-age=31536000"
});
app.Run();
```

### 9.3 [Medium] How do environments work and how do you safely configure per-environment behavior?
**Answer**
The host reads `ASPNETCORE_ENVIRONMENT`/`DOTNET_ENVIRONMENT` (default Production) and exposes it through `IWebHostEnvironment`. Use it to load `appsettings.{Env}.json`, enable dev-only middleware, and keep secrets in user secrets/Key Vault rather than files.

**Explanation**
Prefer configuration/feature flags over environment `if` blocks when behavior should be tunable. Don't use `Staging` as production-lite blindly; names are arbitrary strings, case-insensitive compare via `IsEnvironment`.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
    app.UseExceptionHandler("/error");

app.MapGet("/env", (IWebHostEnvironment env) => env.EnvironmentName);
app.Run();
// Output (no env var set): Production
```

### 9.4 [Hard] Why does the app run as Development locally but Production when deployed, and where does launchSettings fit?
**Answer**
Locally, `dotnet run` applies the first launch profile (`launchSettings.json`) which sets `ASPNETCORE_ENVIRONMENT=Development`. That file isn't published or consumed by the runtime, so in a deployed host the variable is unset and the app defaults to Production.

**Explanation**
Also, `dotnet run` only uses profiles of `commandName: Project`; IDEs choose their own. When running `dotnet MyApp.dll` or from a container there is no launch profile. Developer exception page, detailed errors, and user secrets therefore disappear in Production — a design feature for security.

**Gotchas / follow-ups**
- Never commit secrets to launchSettings.json (it's in source control).
- `applicationUrl` in launchSettings is overridden by `ASPNETCORE_URLS` or `UseUrls`; Kestrel's `appsettings` `Kestrel:Endpoints` has its own priority rules.

**Code**
```json
{
  "profiles": {
    "Api": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:7001;http://localhost:5001",
      "environmentVariables": { "ASPNETCORE_ENVIRONMENT": "Development" }
    }
  }
}
```

### 9.5 [Tricky] `UseDefaultFiles` is called after `UseStaticFiles` and `/` returns 404. Why?
**Answer**
`UseDefaultFiles` only rewrites the URL (`/` → `/index.html`); the actual file response is produced by `UseStaticFiles`, which therefore has to run *after* the rewrite. If the order is reversed, static files sees `/` first, finds nothing, and the rewrite is too late.

**Explanation**
This is why `UseFileServer` is safer. Also remember default file names list (`default.htm`, `index.html` etc.) is configurable via `DefaultFilesOptions.DefaultFileNames`.

**Gotchas / follow-ups**
- SPA fallback: use `app.MapFallbackToFile("index.html")` instead of rewriting.

**Code**
```csharp
app.UseFileServer();                 // correct, single call
app.MapFallbackToFile("index.html"); // SPA client-side routes
```

**Interview traps**
- Thinking launchSettings.json works in production.
- Putting files outside wwwroot and expecting them to be served.
- Order of `UseDefaultFiles` vs `UseStaticFiles`.
- Linux file name casing for `appsettings.{Env}.json`.

**Cheat sheet**
- Web root = `wwwroot`; content root = app folder.
- Env default = Production.
- launchSettings = dev-only.
- `UseDefaultFiles` before `UseStaticFiles`, or `UseFileServer`.
- SPA → `MapFallbackToFile`.

---

## Topic 10: App Lifetime, Graceful Shutdown, IHostedService

**Study notes**

- `IHostApplicationLifetime`: `ApplicationStarted`, `ApplicationStopping`, `ApplicationStopped` tokens, plus `StopApplication()`.
- **Startup order**: hosted services' `StartAsync` run sequentially in registration order, and in the generic web host they run **before** the server starts listening. A slow `StartAsync` delays readiness.
- **Shutdown**: SIGTERM/Ctrl+C → `ApplicationStopping` → `StopAsync` on hosted services in **reverse** order → server drains in-flight requests → `ApplicationStopped`. Total time bounded by `HostOptions.ShutdownTimeout` (default 30 s in .NET 6+; it was 5 s earlier). Kubernetes `terminationGracePeriodSeconds` should exceed it.
- `BackgroundService` implements `IHostedService`; `ExecuteAsync` runs on a background task; return when `stoppingToken` triggers. If it throws, default behavior in .NET 6+ is to **stop the host** (`BackgroundServiceExceptionBehavior.StopHost`).
- .NET 8: `IHostedLifecycleService` (StartingAsync/StartedAsync/StoppingAsync/StoppedAsync), `HostOptions.ServicesStartConcurrently`/`ServicesStopConcurrently`.
- Hosted services are singletons: use `IServiceScopeFactory` to get scoped services (DbContext).

```
SIGTERM -> ApplicationStopping -> StopAsync(reverse order, bounded by ShutdownTimeout)
        -> server stops accepting, drains requests -> ApplicationStopped -> process exit
```

### 10.1 [Easy] What is IHostedService and BackgroundService?
**Answer**
`IHostedService` defines `StartAsync`/`StopAsync` for long-running work managed by the host. `BackgroundService` is a base class where you just override `ExecuteAsync` and honor the stopping token.

**Explanation**
Register with `AddHostedService<T>()`. Use for queue consumers, schedulers, warm-up, and polling tasks inside web apps or worker services.

**Code**
```csharp
public sealed class Heartbeat(ILogger<Heartbeat> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            log.LogInformation("beat");
            try { await Task.Delay(TimeSpan.FromSeconds(5), ct); }
            catch (OperationCanceledException) { }
        }
    }
}
// builder.Services.AddHostedService<Heartbeat>();
```

### 10.2 [Follow-up] How do you use a scoped service (like DbContext) from a BackgroundService?
**Answer**
Inject `IServiceScopeFactory`, create a scope per unit of work, and resolve the scoped service from it. The hosted service itself is a singleton, so constructor-injecting a scoped service is a captive dependency.

**Explanation**
Create the scope per iteration/message so DbContext doesn't live forever (change tracker growth, stale data, thread-safety). Use `await using var scope` (`CreateAsyncScope`) to dispose properly.

**Code**
```csharp
public sealed class Cleaner(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));
        while (await timer.WaitForNextTickAsync(ct))
        {
            await using var scope = scopes.CreateAsyncScope();
            var svc = scope.ServiceProvider.GetRequiredService<ICleanupService>();
            await svc.RunAsync(ct);
        }
    }
}
public interface ICleanupService { Task RunAsync(CancellationToken ct); }
```

### 10.3 [Medium] Describe graceful shutdown and how to make long-running work cooperate.
**Answer**
On SIGTERM the host raises `ApplicationStopping`, calls `StopAsync` on hosted services in reverse order, and Kestrel stops accepting connections and waits for in-flight requests, all within `ShutdownTimeout`. Background work must observe the `stoppingToken` and finish or checkpoint quickly.

**Explanation**
Exceeding the timeout leads to forced termination. Raise `ShutdownTimeout` if you need to drain queues, and align with orchestrator grace periods. Ctrl+C triggers the same flow. Don't swallow `OperationCanceledException` incorrectly or block in `StopAsync`.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(45));
var app = builder.Build();
var life = app.Services.GetRequiredService<IHostApplicationLifetime>();
life.ApplicationStopping.Register(() => Console.WriteLine("stopping: drain work"));
app.Run();
```

### 10.4 [Hard] What happens when `ExecuteAsync` throws, and what is the start-up ordering gotcha with `BackgroundService`?
**Answer**
Since .NET 6 an unhandled exception from `ExecuteAsync` is logged and the host **stops** (`BackgroundServiceExceptionBehavior.StopHost`; earlier versions swallowed it silently). Also, `BackgroundService.StartAsync` returns once the *synchronous part* of `ExecuteAsync` reaches its first `await`; long synchronous work before the first await blocks host startup.

**Explanation**
Therefore, put `await Task.Yield()` at the top, or move initialization into awaited async code, if the initial section is CPU-heavy. You can change behavior with `HostOptions.BackgroundServiceExceptionBehavior = Ignore`, but better to catch/log/retry inside the loop. The exception surfaces through `ExecuteTask`.

**Gotchas / follow-ups**
- Use `IHostedLifecycleService.StartedAsync` (8+) for work after all services started.
- Order: `StartAsync` in registration order; `StopAsync` in reverse.

**Code**
```csharp
public sealed class Worker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await Task.Yield(); // don't block host startup with sync work below
        while (!ct.IsCancellationRequested)
        {
            try { await DoOnceAsync(ct); }
            catch (Exception ex) when (ex is not OperationCanceledException)
            { /* log, back off; don't let the host die unintentionally */
              await Task.Delay(1000, ct); }
        }
    }
    static Task DoOnceAsync(CancellationToken ct) => Task.Delay(100, ct);
}
```

### 10.5 [Tricky] In-flight request during shutdown: does `ApplicationStopping` cancel `HttpContext.RequestAborted`?
**Answer**
No. `RequestAborted` fires only when the client disconnects or the request is aborted. During graceful shutdown Kestrel stops accepting new connections and waits for in-flight requests to complete (up to the timeout), after which they're aborted.

**Explanation**
If you need requests to stop early on shutdown, link `RequestAborted` with `IHostApplicationLifetime.ApplicationStopping` using `CancellationTokenSource.CreateLinkedTokenSource`. Kestrel's `HostOptions.ShutdownTimeout` plus Kestrel's own shutdown timer both matter.

**Gotchas / follow-ups**
- Long-lived connections (SignalR, SSE) need explicit handling or they hold up shutdown until the timeout.

**Code**
```csharp
app.MapGet("/long", async (HttpContext ctx, IHostApplicationLifetime life) =>
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(
        ctx.RequestAborted, life.ApplicationStopping);
    await Task.Delay(TimeSpan.FromMinutes(5), cts.Token);
    return Results.Ok();
});
```

**Interview traps**
- Injecting scoped services into a hosted service ctor.
- Heavy synchronous code before the first `await` in `ExecuteAsync`.
- Assuming exceptions in `ExecuteAsync` are ignored (they stop the host since .NET 6).
- Orchestrator grace period shorter than `ShutdownTimeout`.

**Cheat sheet**
- Start in order; stop in reverse.
- `ShutdownTimeout` default 30 s (.NET 6+).
- `IServiceScopeFactory` for scoped deps.
- .NET 8: `IHostedLifecycleService`, concurrent start/stop options.
- `RequestAborted` ≠ shutdown signal.

---

## Topic 11: WebApplicationFactory & Integration Testing Hosting Model

**Study notes**

- `Microsoft.AspNetCore.Mvc.Testing` provides `WebApplicationFactory<TEntryPoint>`: boots the real app (real `Program.cs`, DI, middleware) in-memory with **TestServer** (no sockets, no Kestrel), and gives an `HttpClient` (`CreateClient()`) that calls it directly.
- With top-level statements, the generated `Program` class is `internal`: add `public partial class Program { }` (or `InternalsVisibleTo`) so the test project can reference it.
- Customize: `WithWebHostBuilder(b => b.ConfigureTestServices(...))` to replace services (DB, external HTTP clients, time); `UseEnvironment`, `ConfigureAppConfiguration`. The factory's default environment is **Development**.
- Content root resolution uses `WebApplicationFactoryContentRoot` attribute and project's .deps/`.runtimeconfig`.
- Use fixtures: `IClassFixture<WebApplicationFactory<Program>>` (xUnit) shares one server per class; each `WithWebHostBuilder` creates a derived factory (new host).
- For real Kestrel/browsers (Playwright), you need a real listening host (custom factory or `Testcontainers`/`dotnet run`), because TestServer has no real port.

### 11.1 [Easy] What is WebApplicationFactory?
**Answer**
It's a test helper that spins up your ASP.NET Core app in memory using TestServer and returns an `HttpClient` to send requests to it. It lets you do integration tests of the whole pipeline without deploying.

**Explanation**
Tests exercise routing, middleware, DI, serialization, and auth together. It's much faster than hosting a real server and avoids port conflicts.

**Code**
```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

public class SmokeTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Root_returns_200()
    {
        var client = factory.CreateClient();
        var res = await client.GetAsync("/");
        Assert.Equal(System.Net.HttpStatusCode.OK, res.StatusCode);
    }
}

// In the web project (Program.cs), at the bottom:
// public partial class Program { }
```

### 11.2 [Follow-up] How do you replace a database or external service in tests?
**Answer**
Use `WithWebHostBuilder` + `ConfigureTestServices` to remove the registered descriptor and add a test double (in-memory/SQLite/Testcontainers DB, fake HTTP handler). Because `ConfigureTestServices` runs after the app's own registrations, the replacement wins.

**Explanation**
For EF Core, remove `DbContextOptions<T>` before re-adding the context with a different provider (otherwise two providers are configured and EF throws). Prefer a real database in a container for fidelity; the InMemory provider doesn't enforce relational behavior (FKs, transactions).

**Code**
```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

public class ApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(Microsoft.AspNetCore.Hosting.IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDb>>();
            services.AddDbContext<AppDb>(o => o.UseSqlite("DataSource=:memory:"));
            services.AddSingleton<TimeProvider>(new FakeTime());
        });
    }
}
public class AppDb(DbContextOptions<AppDb> o) : DbContext(o) { }
public sealed class FakeTime : TimeProvider { }
```

### 11.3 [Medium] How do you fake authentication in integration tests?
**Answer**
Register a test `AuthenticationHandler<AuthenticationSchemeOptions>` as the default scheme in `ConfigureTestServices` that produces a fixed `ClaimsPrincipal`, so `[Authorize]` endpoints pass without real tokens. Alternatively generate real signed JWTs using a test key.

**Explanation**
The fake handler can read headers (e.g., `X-Test-Role`) so tests can vary identity per request. This exercises authorization policies while bypassing identity provider calls.

**Code**
```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

public sealed class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> opts, ILoggerFactory logger, UrlEncoder enc)
    : AuthenticationHandler<AuthenticationSchemeOptions>(opts, logger, enc)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var id = new ClaimsIdentity([new Claim(ClaimTypes.Name, "tester")], "Test");
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(id), "Test");
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
// services.AddAuthentication("Test").AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
```

### 11.4 [Hard] Why can test configuration overrides be ignored with minimal hosting, and how do you avoid it?
**Answer**
With the minimal hosting model, `Program.cs` may read `builder.Configuration` values eagerly while building services, before the factory's `ConfigureAppConfiguration` callbacks are applied, so test overrides can arrive too late for such code. Use `builder.UseSetting(...)`/environment variables set before the host is created, or structure code to use the Options pattern (resolved lazily).

**Explanation**
Lazy consumers (`IOptions<T>`, `IConfiguration` read at runtime) see the overrides; eager reads in top-level code may not. This is a well-known pitfall with `WebApplicationFactory` + `WebApplication.CreateBuilder` and has had behavior changes across releases, so verify against your target version.

**Gotchas / follow-ups**
- Prefer `ConfigureTestServices` for services and `UseSetting` for early configuration.
- Each `WithWebHostBuilder` call creates a new host — an expensive operation; cache via a shared factory.
- Static state in `Program` leaks across tests in the same process.

**Code**
```csharp
var factory = new WebApplicationFactory<Program>()
    .WithWebHostBuilder(b =>
    {
        b.UseSetting("ConnectionStrings:Default", "DataSource=:memory:"); // applied early
        b.UseEnvironment("Testing");
    });
var client = factory.CreateClient();
```

### 11.5 [Tricky] Tests pass with TestServer but fail in production with HTTPS redirect / Kestrel limits. Why?
**Answer**
TestServer bypasses Kestrel and the network: no TLS, no body-size limits, no HTTP/2, no forwarded headers, no IIS module. It's a close, not perfect, simulation of the pipeline.

**Explanation**
`UseHttpsRedirection` can cause 307s in tests (client created with `AllowAutoRedirect` default true; set `BaseAddress` to `https://localhost`). For Kestrel-specific behavior or browser tests, host on a real port (e.g., a custom `WebApplicationFactory` that starts Kestrel, or run the app out-of-process).

**Gotchas / follow-ups**
- `WebApplicationFactoryClientOptions { AllowAutoRedirect = false }` to assert redirects.

**Code**
```csharp
var client = factory.CreateClient(new WebApplicationFactoryClientOptions
{
    AllowAutoRedirect = false,
    BaseAddress = new Uri("https://localhost")
});
```

**Interview traps**
- Forgetting `public partial class Program { }`.
- Using the InMemory EF provider and trusting relational behavior.
- Assuming TestServer == Kestrel.
- Expecting `ConfigureAppConfiguration` to affect eager reads in `Program.cs`.

**Cheat sheet**
- `WebApplicationFactory<Program>` + TestServer = in-memory host.
- `ConfigureTestServices` runs after app registrations.
- Default env = Development.
- `CreateClient` options: `AllowAutoRedirect`, `BaseAddress`.
- Real Kestrel needs a different approach.

---

## Topic 12: HttpContext, Items, Features, IHttpContextAccessor

**Study notes**

- `HttpContext` encapsulates one request/response: `Request`, `Response`, `User`, `Connection`, `RequestServices` (scoped provider), `Items`, `Features`, `Session`, `TraceIdentifier`, `RequestAborted`, `WebSockets`.
- **Items**: `IDictionary<object, object?>` for per-request data passed between middleware/filters/handlers. Cleared with the request.
- **Features**: `IFeatureCollection` — low-level interfaces (`IHttpRequestFeature`, `IHttpResponseFeature`, `IHttpResponseBodyFeature`, `IHttpConnectionFeature`, `IServerVariablesFeature`...) implemented by the server. `HttpRequest`/`HttpResponse` are facades over features; middleware can replace features (e.g., response buffering/compression wrappers).
- `HttpContext` is **not thread-safe** and is **pooled/reused** after the request completes (Kestrel recycles context objects); never touch it after the request ends or from parallel threads.
- **IHttpContextAccessor** (`AddHttpContextAccessor()`): exposes `HttpContext` via `AsyncLocal`, so it flows with async calls but is `null` where no request exists (hosted services, startup, some background threads after completion).
- Prefer passing needed values (user id, tenant) explicitly or via a scoped service populated by middleware over scattering the accessor.

### 12.1 [Easy] What is HttpContext and how do you pass data between middleware?
**Answer**
`HttpContext` represents the current HTTP request and response with related services and the user. For per-request data across middleware, use `HttpContext.Items`.

**Explanation**
Items is a plain dictionary scoped to the request; keys should be unique (use private `object` keys or type-based keys to avoid collisions). For typed shared data, you could also register a scoped service and fill it in middleware.

**Code**
```csharp
var app = WebApplication.Create(args);

app.Use(async (ctx, next) =>
{
    ctx.Items["start"] = DateTimeOffset.UtcNow;
    await next(ctx);
});
app.MapGet("/", (HttpContext ctx) => $"started: {ctx.Items["start"]}");
app.Run();
```

### 12.2 [Follow-up] How do you use IHttpContextAccessor properly?
**Answer**
Register `AddHttpContextAccessor()`, inject `IHttpContextAccessor`, and read `.HttpContext` lazily at use time (not cached in constructors/fields) while within a request. Always null-check since it may be null outside a request.

**Explanation**
The accessor is a singleton backed by `AsyncLocal<T>`, so the context is per execution flow. Using it in a service layer couples it to web concerns; wrap in a small abstraction like `ICurrentUser`.

**Code**
```csharp
public interface ICurrentUser { string? Id { get; } }

public sealed class CurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    public string? Id => accessor.HttpContext?.User.FindFirst("sub")?.Value;
}

// builder.Services.AddHttpContextAccessor();
// builder.Services.AddScoped<ICurrentUser, CurrentUser>();
```

### 12.3 [Medium] What are Features, and why would you care?
**Answer**
Features are the server-agnostic low-level capabilities (request/response/connection) exposed via `IFeatureCollection`; `HttpRequest`/`HttpResponse` are built on them. Middleware like response compression/buffering replaces or wraps features such as `IHttpResponseBodyFeature` to intercept the output.

**Explanation**
Servers (Kestrel, IIS, TestServer) implement the same features, which is what makes middleware portable. You can read features to access server-specific info (e.g., `IHttpMaxRequestBodySizeFeature` to change the body limit for a request at runtime, `IHttpRequestLifetimeFeature` to abort).

**Code**
```csharp
using Microsoft.AspNetCore.Http.Features;

app.MapPost("/upload", (HttpContext ctx) =>
{
    var f = ctx.Features.Get<IHttpMaxRequestBodySizeFeature>();
    if (f is { IsReadOnly: false }) f.MaxRequestBodySize = 100_000_000; // only before body is read
    return Results.Ok();
});
```

### 12.4 [Hard] What bugs occur when HttpContext is used from background tasks or captured in singletons?
**Answer**
After the request completes, the `HttpContext` object may be recycled for another request, so a captured reference can read or mutate a *different* user's data (or throw `ObjectDisposedException`/`NullReferenceException`). Fire-and-forget tasks started in a request that use `HttpContext` or request-scoped services are unsafe.

**Explanation**
Copy required values (user id, headers, correlation id) into locals/DTOs before starting background work. Don't use `Task.Run` with scoped services resolved from `RequestServices` — they're disposed at request end; create a new scope via `IServiceScopeFactory`. Parallel use (`Task.WhenAll` on the same context) is also unsafe because HttpContext isn't thread-safe.

**Gotchas / follow-ups**
- `IHttpContextAccessor.HttpContext` is null in `Task.Run` started after the request ended or when the execution context was suppressed.
- Using `HttpContext.Response` after `await` in `async void` handlers leads to crashes.

**Code**
```csharp
app.MapPost("/jobs", (HttpContext ctx, IServiceScopeFactory scopes, IBackgroundTaskQueue queue) =>
{
    var userId = ctx.User.Identity?.Name;           // copy, don't capture ctx
    queue.Enqueue(async ct =>
    {
        await using var scope = scopes.CreateAsyncScope();
        var svc = scope.ServiceProvider.GetRequiredService<IJobService>();
        await svc.RunAsync(userId, ct);
    });
    return Results.Accepted();
});

public interface IBackgroundTaskQueue { void Enqueue(Func<CancellationToken, Task> work); }
public interface IJobService { Task RunAsync(string? user, CancellationToken ct); }
```

### 12.5 [Tricky] Why is `Request.Body` empty the second time you read it, and how do you read it twice?
**Answer**
The request body is a forward-only stream; once consumed (e.g., by model binding), it's at the end. Call `Request.EnableBuffering()` before the first read and reset `Position = 0` afterwards (leave the stream open).

**Explanation**
Buffering stores in memory up to 30 KB then spills to disk temp files. Do it in middleware before MVC binding. Don't `using`/dispose the reader's stream (`leaveOpen: true`). For large uploads avoid buffering entirely.

**Gotchas / follow-ups**
- Request logging middleware that reads the body must rewind, otherwise the endpoint sees an empty body.
- Disk-spilled buffers add I/O cost and a DoS vector without size limits.

**Code**
```csharp
app.Use(async (ctx, next) =>
{
    ctx.Request.EnableBuffering();
    using var reader = new StreamReader(ctx.Request.Body, leaveOpen: true);
    string body = await reader.ReadToEndAsync();
    ctx.Request.Body.Position = 0;     // rewind for downstream
    Console.WriteLine($"len={body.Length}");
    await next(ctx);
});
```

**Interview traps**
- `HttpContext.Current` doesn't exist in ASP.NET Core.
- Caching `HttpContext` in a singleton field.
- Using accessor in hosted services (always null).
- Running parallel tasks on one HttpContext.

**Cheat sheet**
- Items = per-request dictionary.
- Features = server capabilities; Request/Response are facades.
- HttpContext: not thread-safe, recycled.
- Accessor = AsyncLocal; null outside requests.
- `EnableBuffering` to re-read body.

---

## Topic 13: Filters vs Middleware; ASP.NET Core vs ASP.NET MVC 5

**Study notes**

- **Middleware**: runs for *every* request in the host pipeline, knows only `HttpContext`; ideal for cross-cutting HTTP concerns (HTTPS redirect, CORS, auth, logging, compression).
- **MVC filters**: run inside the MVC action invocation, know about action context (`ActionArguments`, `ModelState`, `IActionResult`, controller); apply per-action/controller/global.
- Filter order: Authorization → Resource → (model binding) → Action → (action) → Exception (wraps action/action filters/binding) → Result → (result execution). Resource filters wrap the rest (caching/short-circuit before model binding). Exception filters **don't** catch exceptions from resource filters or result filters/result execution.
- Types: `IAuthorizationFilter`, `IResourceFilter`, `IActionFilter`, `IExceptionFilter`, `IResultFilter`, each with async variant; implement either sync or async, not both. DI: `[ServiceFilter]` (resolved from DI), `[TypeFilter]` (instantiated with DI args), or `IFilterFactory`. Minimal APIs have **endpoint filters**.
- **ASP.NET MVC 5 / Web API 2 (.NET Framework)**: System.Web, `Global.asax`, `HttpContext.Current`, `web.config`, IIS-only, separate MVC and Web API stacks (`Controller` vs `ApiController`), OWIN/Katana optional, DI bolted on (Unity/Autofac), `SynchronizationContext` (deadlock-prone with `.Result`). ASP.NET Core: unified MVC/API (`ControllerBase`), built-in DI, middleware pipeline, cross-platform, no `SynchronizationContext`, tag helpers, Razor Pages, configuration/options, much higher perf.

### 13.1 [Easy] What's the difference between middleware and filters?
**Answer**
Middleware operates at the HTTP pipeline level for all requests; filters operate within the MVC action pipeline and know about actions, model state, and results. Use middleware for generic HTTP concerns and filters for MVC-specific logic.

**Explanation**
A request for a static file passes through middleware but never reaches filters. An action filter can inspect `ActionArguments` — middleware can't, as binding hasn't happened yet.

**Code**
```csharp
using Microsoft.AspNetCore.Mvc.Filters;

public sealed class LogActionFilter(ILogger<LogActionFilter> log) : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext c) =>
        log.LogInformation("Executing {Action}", c.ActionDescriptor.DisplayName);
    public void OnActionExecuted(ActionExecutedContext c) { }
}
// builder.Services.AddControllers(o => o.Filters.Add<LogActionFilter>());
```

### 13.2 [Follow-up] Order the filter types and say what each is best for.
**Answer**
Authorization (access checks) → Resource (caching, short-circuit before binding) → Action (pre/post action logic, argument validation) → Exception (handle unhandled exceptions from action) → Result (modify the result before/after execution). Within a type, global → controller → action scope on the way in (reverse on the way out) unless `Order` is set.

**Explanation**
Use `Order` property to control sequence. Resource filters run before model binding so they're good for short-circuiting expensive binding (e.g., response caching, feature gate).

**Code**
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public sealed class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext ctx)
    {
        if (!ctx.ModelState.IsValid)
            ctx.Result = new BadRequestObjectResult(ctx.ModelState); // short-circuit
    }
}
```

### 13.3 [Medium] How do you use DI in filters?
**Answer**
Attributes can't take constructor DI directly; use `[ServiceFilter(typeof(MyFilter))]` (filter must be registered in DI) or `[TypeFilter(typeof(MyFilter), Arguments = ...)]` which creates it with DI plus extra arguments. Or implement `IFilterFactory`.

**Explanation**
`ServiceFilter` respects the registered lifetime (singleton filters must be thread-safe); `TypeFilter` creates a new instance per use (can be cached with `IsReusable`). Global filters added by type (`Filters.Add<T>()`) are also DI-activated.

**Code**
```csharp
public sealed class AuditFilter(IAuditService audit) : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext ctx, ActionExecutionDelegate next)
    {
        var executed = await next();
        await audit.WriteAsync(ctx.ActionDescriptor.DisplayName ?? "?", executed.Exception is null);
    }
}
public interface IAuditService { Task WriteAsync(string action, bool ok); }

// builder.Services.AddScoped<AuditFilter>();
// [ServiceFilter(typeof(AuditFilter))] on a controller/action
```

### 13.4 [Hard] Which exceptions can an exception filter NOT handle, and what's the global alternative?
**Answer**
Exception filters handle unhandled exceptions from controller creation, model binding, action filters and action methods. They don't catch exceptions from authorization/resource filters, result filters, or result execution (formatting), and don't see non-MVC exceptions at all. Global handling is best done via `UseExceptionHandler` or `IExceptionHandler` (.NET 8) plus `AddProblemDetails`.

**Explanation**
`IExceptionHandler` (.NET 8) is invoked by the exception handler middleware, returns `true` when handled, and supports chaining multiple handlers. Middleware catches everything downstream, including endpoints and serialization failures, hence it's the more reliable catch-all. Exception filters are still useful for MVC-specific mapping (e.g., domain exception → `ObjectResult`).

**Gotchas / follow-ups**
- Exceptions thrown after the response started cannot change the response.
- Exception filters aren't invoked for exceptions thrown outside MVC (e.g., in earlier middleware).

**Code**
```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> log) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        log.LogError(ex, "Unhandled");
        ctx.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails { Title = "Server error" }, ct);
        return true;
    }
}
// builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
// builder.Services.AddProblemDetails();
// app.UseExceptionHandler();
```

### 13.5 [Hard] List the key architectural differences between ASP.NET MVC 5 and ASP.NET Core MVC.
**Answer**
MVC 5 sits on System.Web/IIS with `Global.asax`, `web.config`, static `HttpContext.Current`, separate MVC and Web API stacks, and optional OWIN. ASP.NET Core is cross-platform with Kestrel, a middleware pipeline, built-in DI and configuration/options, unified controllers, no `System.Web` and no `SynchronizationContext`, and side-by-side deployment.

**Explanation**
Other differences: `IActionResult` vs `ActionResult`, tag helpers vs HTML helpers (both exist), `appsettings.json` vs `ConfigurationManager.AppSettings`, bundling/minification moved to build tools, `[FromBody]` default behavior changes (Web API 2 vs `[ApiController]` inference), JSON via System.Text.Json instead of Newtonsoft by default, route attribute defaults. Migration: gradual with YARP + System.Web adapters.

**Gotchas / follow-ups**
- `Request.Form` access is async (`ReadFormAsync`) in Core; sync IO is disabled by default on Kestrel (`AllowSynchronousIO` false).
- Model binding of JSON bodies: Core binds a single body parameter; missing `[FromBody]` rules differ.

**Code**
```csharp
// MVC5: var user = HttpContext.Current.User.Identity.Name;   // static, System.Web
// Core: inject IHttpContextAccessor or use ControllerBase.User
using Microsoft.AspNetCore.Mvc;

[ApiController, Route("me")]
public class MeController : ControllerBase
{
    [HttpGet] public IActionResult Get() => Ok(User.Identity?.Name ?? "anonymous");
}
```

### 13.6 [Tricky] Why do `.Result`/`.Wait()` deadlocks happen in MVC 5 but typically not in ASP.NET Core?
**Answer**
ASP.NET (System.Web) installs a `SynchronizationContext` that serializes continuations onto the request context; blocking the context with `.Result` while the continuation waits to resume on it deadlocks. ASP.NET Core has no such context, so continuations run on the thread pool, avoiding the classic deadlock — but blocking still causes thread-pool starvation.

**Explanation**
So sync-over-async is "less deadlock-prone" but still harmful for scalability. `ConfigureAwait(false)` matters mainly in libraries targeting both. Kestrel disallows synchronous IO by default as another nudge to async.

**Code**
```csharp
// Still bad in Core: blocks a pool thread, may starve under load
// var data = SomeAsync().Result;

app.MapGet("/good", async (HttpClient http) => await http.GetStringAsync("https://example.com"));
```

**Interview traps**
- Believing filters run for static files.
- Implementing both sync and async versions of the same filter (only the async one runs).
- Expecting exception filters to catch result-execution exceptions.
- Using `HttpContext.Current` in Core code.

**Cheat sheet**
- Middleware = all requests, HTTP-level; filters = MVC-level with action context.
- Order: Auth → Resource → Action → Exception → Result.
- DI in filters: `ServiceFilter` / `TypeFilter`.
- Global errors: `UseExceptionHandler` + `IExceptionHandler` (.NET 8).
- Core has no `SynchronizationContext`, no System.Web.

---

## Topic 14: Assembly Loading & AssemblyLoadContext

**Study notes**

- .NET Framework had multiple **AppDomains** (isolation + unload). .NET Core+ has a **single AppDomain**; isolation/unload is done with **AssemblyLoadContext (ALC)**. `AppDomain.CreateDomain` throws `PlatformNotSupportedException`.
- `AssemblyLoadContext.Default` holds the app + framework assemblies (TPA list from `deps.json`/`runtimeconfig.json`). `Assembly.Load(name)` resolves by identity; `Assembly.LoadFrom(path)`/`LoadFile` load by path (`LoadFrom` into the default context; use ALC for isolation).
- **Custom ALC**: override `Load(AssemblyName)` to resolve dependencies (typically with `AssemblyDependencyResolver` pointing to plugin's main dll to honor its `.deps.json`), `LoadUnmanagedDll` for native libs. Return `null` from `Load` to fall back to the default context (shared contracts).
- **Collectible ALC** (`isCollectible: true`) can be `Unload()`ed; unload is cooperative — it completes only when no references to its types/assemblies/instances remain (GC-based). Use a `WeakReference` to verify.
- Same assembly loaded in two ALCs = **different type identities**; casting across them throws `InvalidCastException`. Share interface assemblies by letting the default ALC load them.
- Events: `AppDomain.AssemblyResolve` (Framework & Core) and `AssemblyLoadContext.Resolving` for failed resolution.

```
Default ALC: App.dll, Contracts.dll, System.*.dll
   ^ (Load returns null -> fallback for Contracts)
PluginALC(A): PluginA.dll + its deps     PluginALC(B): PluginB.dll + (different Newtonsoft version)
```

### 14.1 [Easy] What is AssemblyLoadContext and why does it exist?
**Answer**
An ALC is a runtime scope in which assemblies are loaded and resolved. It enables plugin isolation, side-by-side dependency versions, and unloading — replacing AppDomains, which .NET Core no longer supports.

**Explanation**
Every assembly belongs to exactly one ALC. The default context handles your app; plugin systems create one per plugin so their dependencies don't conflict.

**Code**
```csharp
using System.Reflection;
using System.Runtime.Loader;

var alc = AssemblyLoadContext.GetLoadContext(typeof(string).Assembly);
Console.WriteLine(alc?.Name);
// Output: Default
```

### 14.2 [Follow-up] How do you build a plugin loader with dependency isolation?
**Answer**
Create a custom ALC with `AssemblyDependencyResolver` for the plugin path, load the plugin assembly into it, find types implementing a shared contract interface, and create instances. The contract assembly must be loaded in the default ALC so both sides share the type.

**Explanation**
`AssemblyDependencyResolver` reads `plugin.deps.json` to resolve managed and native dependencies. Returning `null` from `Load` for the contract assembly keeps it in the default context. Publish the plugin with `<EnableDynamicLoading>true</EnableDynamicLoading>` so its deps and runtimeconfig are emitted.

**Code**
```csharp
using System.Reflection;
using System.Runtime.Loader;

public interface IPlugin { string Run(); }   // lives in shared Contracts assembly

public sealed class PluginContext(string pluginPath) : AssemblyLoadContext(isCollectible: true)
{
    private readonly AssemblyDependencyResolver _resolver = new(pluginPath);

    protected override Assembly? Load(AssemblyName name)
    {
        // Share contracts with the host
        if (name.Name == typeof(IPlugin).Assembly.GetName().Name) return null;
        var path = _resolver.ResolveAssemblyToPath(name);
        return path is null ? null : LoadFromAssemblyPath(path);
    }
}

public static class Loader
{
    public static IPlugin Create(PluginContext ctx, string pluginPath)
    {
        var asm = ctx.LoadFromAssemblyPath(pluginPath);
        var type = asm.GetTypes().First(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsAbstract);
        return (IPlugin)Activator.CreateInstance(type)!;
    }
}
```

### 14.3 [Medium] How do you unload a collectible ALC and verify it?
**Answer**
Call `Unload()`, drop all references to the context, its assemblies, types and instances, then run GC and check a `WeakReference` to the ALC. Keep the usage in a separate non-inlined method so JIT locals don't keep references alive.

**Explanation**
`Unload()` only *starts* the process; actual unload happens when the GC finds the context unreachable. Common leaks: static fields/caches in the host holding plugin types, running threads, timers, event handlers, `ThreadLocal`, and `Marshal.GetFunctionPointerForDelegate` objects.

**Code**
```csharp
using System.Runtime.CompilerServices;
using System.Runtime.Loader;

[MethodImpl(MethodImplOptions.NoInlining)]
static WeakReference LoadAndUnload(string path)
{
    var ctx = new PluginContext(path);
    var plugin = Loader.Create(ctx, path);
    Console.WriteLine(plugin.Run());
    ctx.Unload();
    return new WeakReference(ctx);
}

var weak = LoadAndUnload(args[0]);
for (int i = 0; weak.IsAlive && i < 10; i++)
{
    GC.Collect();
    GC.WaitForPendingFinalizers();
}
Console.WriteLine($"Unloaded: {!weak.IsAlive}");
// Output (when no leaks): Unloaded: True
```

### 14.4 [Hard] Why do you get InvalidCastException casting a plugin object to its interface, even though names match?
**Answer**
Type identity includes the loading context. If the plugin's ALC loaded its own copy of the contract assembly, `IPlugin` from plugin ALC ≠ `IPlugin` in the host, so the cast fails. Fix by making the plugin ALC defer the contract assembly to the default context (return `null` in `Load`) or by ensuring the plugin output doesn't include a private copy.

**Explanation**
Diagnose with `typeof(IPlugin).Assembly.Location` vs the plugin's referenced copy, and `AssemblyLoadContext.GetLoadContext(asm)`. In plugin projects, mark the contract reference `<Private>false</Private>` and `<ExcludeAssets>runtime</ExcludeAssets>` so it isn't copied to the plugin output.

**Gotchas / follow-ups**
- Similar issue with logging/DI abstractions (`Microsoft.Extensions.*`) passed across the boundary: both sides must use the same copy.
- `Assembly.LoadFile` in the default ALC loads another copy of an assembly by path → duplicate types.

**Code**
```csharp
var pluginAsm = ctx.LoadFromAssemblyPath(pluginPath);
var t = pluginAsm.GetTypes().First(x => x.Name == "MyPlugin");
Console.WriteLine(AssemblyLoadContext.GetLoadContext(typeof(IPlugin).Assembly)?.Name);          // Default
Console.WriteLine(AssemblyLoadContext.GetLoadContext(t.GetInterface(nameof(IPlugin))!.Assembly)?.Name);
// If the second line isn't "Default", the plugin carries its own contract copy -> InvalidCastException
```

### 14.5 [Tricky] Can you unload the default ALC or an assembly loaded with `Assembly.LoadFrom`, and what about AppDomain.AssemblyResolve?
**Answer**
No: the default context is never collectible, so assemblies loaded there stay until process exit. Only collectible custom ALCs can unload. `AppDomain.AssemblyResolve` still fires for failures in the default context, but isolation/unload scenarios should use ALC `Resolving`/`Load`.

**Explanation**
This is why hot-reload of arbitrary assemblies leaks memory if you load into default. NativeAOT and trimmed apps can't dynamically load assemblies at all (AOT has no JIT/loader for IL assemblies).

**Gotchas / follow-ups**
- Dynamic `Reflection.Emit` types can be made collectible via `AssemblyBuilderAccess.RunAndCollect`.
- Interview follow-up: "How does `dotnet watch` hot reload work?" — it uses the runtime's EnC (Edit and Continue) apply-update mechanism, not ALC unloading.

**Code**
```csharp
AppDomain.CurrentDomain.AssemblyResolve += (_, e) =>
{
    Console.WriteLine($"Resolving {e.Name}");
    return null;      // let the runtime fail or continue
};
```

**Interview traps**
- Expecting multiple AppDomains in .NET Core.
- Believing `Unload()` is immediate.
- Static references keeping plugin alive.
- Duplicate contract assemblies across contexts.

**Cheat sheet**
- One AppDomain; use ALC for isolation.
- `AssemblyDependencyResolver` + `Load` override.
- `isCollectible: true` + `Unload()` + GC + `WeakReference`.
- Shared contracts must come from the default ALC.
- Default ALC never unloads.

---

## Topic 15: Publishing Options (FDD, Self-contained, Single-file, Trimmed, AOT)

**Study notes**

| Option | Needs runtime on host? | Size | Startup | Notes |
|---|---|---|---|---|
| Framework-dependent (FDD) | Yes (matching major) | Small | Normal | Portable `.dll` runs anywhere with runtime; security patches via shared runtime |
| Self-contained (SCD) | No | Large (~60–70 MB+) | Normal | `-r <RID>`; app carries runtime; you own patching |
| Single-file | Either | One exe | Slight extraction cost for natives | `PublishSingleFile`; native libs may extract unless `IncludeNativeLibrariesForSelfExtract` |
| ReadyToRun (R2R) | Either | Bigger | Faster | `PublishReadyToRun`; needs RID |
| Trimmed | Self-contained only | Smaller | – | `PublishTrimmed`; breaks reflection unless annotated (IL2026 etc.) |
| NativeAOT | No runtime/JIT | Small-ish native binary | Fastest | `PublishAot`; no dynamic loading/Reflection.Emit; limited reflection; RID-specific |

- `dotnet publish -c Release [-r linux-x64] [--self-contained]`. Since .NET 6 SDK/8 SDK, specifying `-r` alone does **not** imply self-contained (.NET 8 SDK default is framework-dependent).
- ASP.NET Core in .NET 8 AOT: minimal APIs, gRPC; not MVC/Razor/SignalR(server). Use `CreateSlimBuilder`/`CreateEmptyBuilder`.
- Containers: `dotnet publish /t:PublishContainer` (SDK container support); official images from .NET 8 run as non-root by default on port 8080; "chiseled" (distroless-style) images are smaller.

### 15.1 [Easy] Framework-dependent vs self-contained deployment?
**Answer**
Framework-dependent apps are small and rely on the installed .NET runtime; self-contained apps bundle the runtime and run without anything installed. FDD benefits from machine-level runtime security patches; SCD gives version isolation.

**Explanation**
SCD is helpful when the host can't be controlled or you need a specific runtime version. Trade-off: bigger deployments and you must redeploy to patch the runtime.

**Code**
```xml
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
  <SelfContained>true</SelfContained>
</PropertyGroup>
<!-- dotnet publish -c Release  OR:  dotnet publish -c Release -r linux-x64 --self-contained -->
```

### 15.2 [Follow-up] What is single-file publishing and what are its caveats?
**Answer**
It bundles the app and dependencies into one executable. Managed assemblies are loaded from the bundle; native libraries may be extracted to a temp location at first run unless you set `IncludeNativeLibrariesForSelfExtract`.

**Explanation**
`Assembly.Location` returns an empty string in single-file apps (use `AppContext.BaseDirectory`/`Environment.ProcessPath`). Compression is optional (`EnableCompressionInSingleFile`, only for self-contained) trading startup time for size. Config files like appsettings.json can still sit next to the exe.

**Code**
```xml
<PropertyGroup>
  <PublishSingleFile>true</PublishSingleFile>
  <SelfContained>true</SelfContained>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
</PropertyGroup>
```
```csharp
Console.WriteLine(AppContext.BaseDirectory);        // use this instead of Assembly.Location
Console.WriteLine(typeof(Program).Assembly.Location.Length == 0); // True in single-file
```

### 15.3 [Medium] What does trimming do and why does it break apps?
**Answer**
The ILLink trimmer removes unreferenced code from a self-contained app to shrink it. It can't see reflection, dynamic loading, or serializers that discover types at runtime, so those may be removed, causing runtime failures.

**Explanation**
Annotate with `[DynamicallyAccessedMembers]`, `[RequiresUnreferencedCode]`, use source generators (System.Text.Json source-gen, `LoggerMessage`, Regex generator, config binder generator `EnableConfigurationBindingGenerator`), and root assemblies with `TrimmerRootAssembly`. Warnings IL2026/IL2087 etc. indicate unsafe patterns. Test the trimmed output — warnings aren't exhaustive.

**Code**
```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>partial</TrimMode>       <!-- trim only assemblies marked trimmable -->
  <SelfContained>true</SelfContained>
</PropertyGroup>
```

### 15.4 [Hard] How does NativeAOT differ from ReadyToRun, and what are its limitations?
**Answer**
R2R precompiles IL to native code but still ships IL and the JIT/runtime (it can re-JIT hot paths). NativeAOT compiles everything ahead-of-time into a standalone native binary with a minimal runtime (GC, type system) and no JIT, giving fast startup and low memory but restricting dynamism.

**Explanation**
Limitations: no `Assembly.LoadFrom`/dynamic plugin loading, no `Reflection.Emit`/expression compilation to IL (interpreter fallback is limited), reflection requires static analysis, platform-specific builds (publish per RID, cross-OS compile not supported), larger compile times, no C++/CLI, and some libraries incompatible (EF Core isn't fully AOT-compatible; heavy reflection libs). Tools: `IsAotCompatible` property for libraries, analyzers (`EnableAotAnalyzer`).

**Gotchas / follow-ups**
- Peak throughput can be lower than JIT with Dynamic PGO for long-running services.

**Code**
```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```
```csharp
// Using AOT-friendly APIs
using System.Text.Json.Serialization;
[JsonSerializable(typeof(string[]))]
internal partial class Ctx : JsonSerializerContext { }
// JsonSerializer.Serialize(new[] { "a" }, Ctx.Default.StringArray);
```

### 15.5 [Tricky] You publish with `-r linux-x64` on .NET 8 SDK and expect a self-contained app. What do you actually get?
**Answer**
A framework-dependent, RID-specific app (with an apphost for that OS) — since .NET 6 SDK warned and .NET 8 SDK changed the default, `-r`/`RuntimeIdentifier` no longer implies self-contained. Add `--self-contained` (or `<SelfContained>true`) explicitly.

**Explanation**
Older SDKs (≤ 5.0) implied self-contained when `-r` was specified, so scripts written then change behavior after upgrading. Check the output for `System.Private.CoreLib.dll` (present only if self-contained). `.runtimeconfig.json` shows `framework` references for FDD, and `includedFrameworks` for SCD.

**Gotchas / follow-ups**
- Trimming, single-file compression, and AOT require self-contained (AOT implies it).
- R2R with FDD is allowed but only the app's code is precompiled; the framework is already R2R.

**Code**
```csharp
Console.WriteLine(Directory.Exists(AppContext.BaseDirectory));
Console.WriteLine(File.Exists(Path.Combine(AppContext.BaseDirectory, "System.Private.CoreLib.dll")));
// Output: True / False (FDD)  — True / True when self-contained (non-single-file)
```

**Interview traps**
- Assuming `-r` implies self-contained on modern SDKs.
- `Assembly.Location` empty in single-file.
- Trimming without testing reflection/serialization paths.
- Expecting EF Core/MVC to just work with NativeAOT.

**Cheat sheet**
- FDD = small, shared runtime; SCD = bundles runtime.
- Single-file ≠ AOT; it's packaging.
- Trim = self-contained only; watch IL warnings.
- R2R = faster startup, still JIT; AOT = no JIT.
- Container: `PublishContainer`, non-root, port 8080 (.NET 8 images).

---

## Topic 16: .NET 8 / .NET 9 Highlights

**Study notes**

**.NET 8 (Nov 2023, LTS)**
- C# 12: primary constructors (all classes/structs), collection expressions `[1, 2]`, `using` aliases for any type, default parameters in lambdas, inline arrays, `ref readonly` parameters.
- Runtime/BCL: Dynamic PGO on by default, `TimeProvider`, `FrozenDictionary`/`FrozenSet`, `SearchValues<T>`, `CompositeFormat`, `Random.GetItems/Shuffle`, DATAS (dynamic adaptation for Server GC, opt-in), better JSON (source-gen improvements, `JsonNamingPolicy.SnakeCaseLower`, `[JsonStringEnumConverter<T>]` for AOT).
- ASP.NET Core: keyed DI services (`AddKeyedSingleton`, `[FromKeyedServices]`), `IExceptionHandler`, `CreateSlimBuilder`/`CreateEmptyBuilder`, Native AOT support for minimal APIs, `ShortCircuit()`, Identity API endpoints (`MapIdentityApi`), Blazor Web App (static SSR + interactive modes), antiforgery for minimal API form binding, `IHostedLifecycleService`.
- EF Core 8: complex types as value objects, primitive collections, `SqlQuery<T>` for unmapped types, `HierarchyId`, `DateOnly`/`TimeOnly` on SQL Server.
- Tooling: `dotnet publish` container support, non-root container images.

**.NET 9 (Nov 2024, STS – 24 months)**
- C# 13: `params` collections (`params ReadOnlySpan<T>`), new `System.Threading.Lock` type (use `lock` with it), `\e` escape, `allows ref struct` anti-constraint, partial properties.
- BCL: `Task.WhenEach`, LINQ `CountBy`/`AggregateBy`/`Index`, `System.Threading.Lock`, `SearchValues` for strings, `JsonSerializerOptions.Web`, `BinaryFormatter` removed (always throws).
- Runtime: DATAS enabled by default for Server GC; further JIT/PGO improvements.
- ASP.NET Core: built-in OpenAPI document generation (`AddOpenApi`/`MapOpenApi`, Microsoft.AspNetCore.OpenApi; Swashbuckle removed from templates), `MapStaticAssets` (build-time fingerprinting/compression, replaces `UseStaticFiles` in templates), `HybridCache` (Microsoft.Extensions.Caching.Hybrid package), improved Blazor.

### 16.1 [Easy] Name key C# 12 features introduced with .NET 8.
**Answer**
Primary constructors for classes and structs, collection expressions, default lambda parameters, `using` aliases for any type, inline arrays, and `ref readonly` parameters.

**Explanation**
Primary constructor parameters are captured by the type (not auto-properties, unlike records); they're mutable captured variables, so don't treat them as readonly.

**Code**
```csharp
using Point = (int X, int Y);          // alias any type

public class Greeter(string name)      // primary constructor
{
    public string Hello() => $"Hello {name}";
}

int[] a = [1, 2, 3];                   // collection expression
List<int> l = [..a, 4];
Point p = (1, 2);
var add = (int x, int y = 10) => x + y;
Console.WriteLine(add(1));             // Output: 11
```

### 16.2 [Follow-up] What are keyed services and when would you use them?
**Answer**
.NET 8 DI lets you register multiple implementations of the same service under a key (`AddKeyedSingleton<T>(key)`) and resolve them via `[FromKeyedServices(key)]` or `GetRequiredKeyedService<T>(key)`. It replaces custom factory/dictionary patterns for choosing implementations such as multiple payment providers or caches.

**Explanation**
Keys are objects (usually strings/enums). Resolve with `KeyedService.AnyKey` for fallbacks. Works with minimal APIs, controllers (`[FromKeyedServices]` in parameters), and constructor injection.

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddKeyedSingleton<ICache, MemoryCache2>("mem");
builder.Services.AddKeyedSingleton<ICache, RedisCache2>("redis");
var app = builder.Build();

app.MapGet("/c", ([FromKeyedServices("redis")] ICache c) => c.Name);
app.Run();

public interface ICache { string Name { get; } }
public sealed class MemoryCache2 : ICache { public string Name => "mem"; }
public sealed class RedisCache2 : ICache { public string Name => "redis"; }
```

### 16.3 [Medium] What do TimeProvider, FrozenDictionary, and SearchValues bring in .NET 8?
**Answer**
`TimeProvider` is an abstraction over time (and timers) enabling deterministic tests (with `FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing`). `FrozenDictionary/FrozenSet` are immutable collections optimized for read-heavy lookups after a costly creation. `SearchValues<T>` precomputes optimized vectorized searching for sets of chars/bytes.

**Explanation**
Use `TimeProvider.System` in production and inject it instead of `DateTime.UtcNow`. Frozen collections suit static lookup tables built at startup. `SearchValues` used with `Span<T>.IndexOfAny`.

**Code**
```csharp
using System.Buffers;
using System.Collections.Frozen;

var map = new Dictionary<string, int> { ["a"] = 1, ["b"] = 2 }.ToFrozenDictionary();
Console.WriteLine(map["b"]);                       // Output: 2

var vowels = SearchValues.Create("aeiou");
Console.WriteLine("hello".AsSpan().IndexOfAny(vowels)); // Output: 1

TimeProvider tp = TimeProvider.System;
Console.WriteLine(tp.GetUtcNow() <= DateTimeOffset.UtcNow); // Output: True
```

### 16.4 [Hard] What changed for APIs and OpenAPI in .NET 9, and what does `MapStaticAssets` do?
**Answer**
.NET 9 ships built-in OpenAPI generation (`builder.Services.AddOpenApi()` + `app.MapOpenApi()`) via Microsoft.AspNetCore.OpenApi and drops Swashbuckle from templates. `MapStaticAssets` is an endpoint-based replacement for `UseStaticFiles` that computes fingerprinted, pre-compressed (gzip/brotli) assets at build time with ETags and caching headers.

**Explanation**
`MapStaticAssets` handles files from the build-time manifest and integrates with endpoint routing (so authorization/metadata apply). `UseStaticFiles` remains for runtime-dynamic files. OpenAPI generation uses transformers (`AddDocumentTransformer`) for customization and is AOT-compatible, unlike reflection-heavy Swashbuckle.

**Gotchas / follow-ups**
- UI (Swagger UI/Scalar) is not built-in; you add a viewer separately.
- `HybridCache` provides L1+L2 caching with stampede protection (in `Microsoft.Extensions.Caching.Hybrid`).

**Code**
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenApi();                 // .NET 9
var app = builder.Build();

if (app.Environment.IsDevelopment())
    app.MapOpenApi();                          // serves /openapi/v1.json

app.MapStaticAssets();                         // .NET 9 (requires build-time manifest)
app.MapGet("/ping", () => "pong");
app.Run();
```

### 16.5 [Tricky] In .NET 9, what is the new `Lock` type and why not just use `lock (object)`?
**Answer**
`System.Threading.Lock` is a dedicated lock type; when the compiler sees `lock` over an instance of this type it emits `Lock.EnterScope()` instead of `Monitor`, giving a faster, clearer implementation. Locking on a `Lock` through other paths (e.g., casting to `object`) falls back to `Monitor` semantics.

**Explanation**
`lock (obj)` uses the object header/sync block (`Monitor`). `Lock` avoids that and supports `EnterScope()` returning a disposable `Lock.Scope`. The compiler warns when a `Lock` is converted to `object` and used in `lock`, because it'd silently use Monitor.

**Gotchas / follow-ups**
- Not async-friendly: you can't `await` inside a `lock`; use `SemaphoreSlim` for async.
- Thread-affine: must be released by the same thread that acquired it.

**Code**
```csharp
using System.Threading;

class Counter
{
    private readonly Lock _gate = new();   // .NET 9 / C# 13
    private int _n;
    public int Next()
    {
        lock (_gate) { return ++_n; }       // compiles to _gate.EnterScope()
    }
}
```

**Interview traps**
- Saying .NET 9 is LTS (it's STS, 24-month support).
- Confusing keyed DI with named options.
- Believing primary-constructor parameters become properties on classes.
- Assuming `MapStaticAssets` replaces `UseStaticFiles` for dynamic content.

**Cheat sheet**
- .NET 8 LTS: C# 12, keyed DI, `IExceptionHandler`, AOT for minimal APIs, `TimeProvider`, frozen collections.
- .NET 9 STS: C# 13, `Lock`, `MapOpenApi`, `MapStaticAssets`, `HybridCache`, `Task.WhenEach`.
- Dynamic PGO default from .NET 8.
- `BinaryFormatter` removed in .NET 9.
- Primary constructors ≠ records.

