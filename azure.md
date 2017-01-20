# Profiling Azure Web Apps

The general assumptions for a normal CLR byte-code-instrumenting profiler is
that you have full, administrative control over the , and therefore, full
control of the Global Assembly Cache (GAC).  As a corollary, it is usually
assumed that you can load custom assemblies, which is usually added to the GAC,
from any injected or modified function.

In Azure Web Apps (or any shared environment), these assumptions are not valid.
However, a profiler can function properly in Azure Web Apps once the GAC problem
is solved.

## Access to GAC

Unlike standalone instances, Web Apps are run in a shared environment.  The app
and you, the user, have limited access to the underlying system.  Since the GAC
is a global resource, you will not be able to modify it in any way.

What does that mean for the profiler?

Typically, a profiler will inject wrapper code in an assembly that calls out to
functions in a custom assembly.  I'll illustrate with a simplified example of
byte-code injection.

```csharp
// TargetAssembly.dll
class TargetClass {
  public void TargetMethod() {
    // Injected header
    [TracerAssembly.dll]TracerClass.BeginTracerMethod();

    // ... Original method...
  }
}
```

`TargetAssembly.dll` could be in the GAC, which indicate that it's either a system DLL
(e.g. System.dll) or system-wide installed library DLL.  It could also be in the app's
library path, which indicate that it's either app's code or library installed on the
app.

The GAC DLLs are loaded into Shared (App) Domain.  Inversely, Shared Domains can only
contain GAC DLLs.  It is the latter fact that is often un-acknowledged.  Since Shared
Domain is the CLR's representation of the GAC, the user cannot, in any way or fashion,
load non-GAC custom DLLs into the Shared Domain.

Another little known fact: while all code has access to Shared Domain, Shared Domain
has no knowledge or access of other App Domains.  In other words, code loaded into
Shared Domain can only call other code loaded in Shared Domain.

Again, what does this mean for the profiler?

In standalone instances, `TracerAssembly.dll` can be added to the GAC.  Since GAC
assemblies are loaded into the Shared Domain and all code has access to the Shared
Domain, the injection example above works almost everywhere.  The only exception is
`mscorlib`, which I will explain later.

In Azure Web Apps, since the user has no access to modify the GAC,
`TracerAssembly.dll` must be loaded into the app's own app-domain.  Immediately,
the example breaks when `TargetAssembly.dll` is a GAC DLL.  Remember, GAC DLLs are
loaded into the Shared Domain, only have access to other GAC DLLs which are also
loaded into the Shared Domain, and have no knowledge of code/DLLs loaded into in
other app-domains.

To see this effect in action (but by no mean is this a proper profiler or
`TracerAssembly.dll` deployment),
- Create a simple Web App.
- Add your version of `TracerAssembly.dll` into the app's `bin` directory.
- Add an assembly reference to the `TracerAssembly.dll`.
- Deploy your app to Azure Web Site.
- Setup your profiler on Azure Web Site to inject references to `TracerAssembly.dll`.
- Try to inject a function in `System.dll`.  It will not load `TracerAssembly.dll`.
An error in the log should indicate something along the line of "Assembly not found".
- Try to inject a function in your app's DLL, e.g. the app's ProcessRequest override.
You should see `TracerAssembly.dll` loaded in the process manager.

In summary, without any change to your profiler and `TracerAssembly.dll`, you can
still instrument non-GAC assemblies and functions by deploying DLLs in the app's
`bin` directory, but that is missing a significant piece of the pie.  Next, I'll
outline a way for instrumenting GAC assemblies.

## Enclosed Injection
## DLL Loading