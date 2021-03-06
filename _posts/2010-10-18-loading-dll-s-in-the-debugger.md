---
layout: post
title: Loading DLLs in the Debugger
tags: [debugger]
---
In a [recent post]({% post_url 2010-07-22-extension-methods-and-the-debugger %}) I discussed the apparent flakiness of extension methods and the debugger being a result of whether or not the DLL containing the extension methods were loaded into the debugee process.  Several users asked in the comment section why we didn't _fix_ the issue by just loading those DLL's when an extension method was executed.

On the surface this seems like a reasonable request. After all the compiler knows what DLLs were involved in the compilation process and hence which extension methods were available from those DLLs. This information could be added to the PDB and used by the debugger to load the DLL when the extension method was requested. Right?

Unfortunately the answer is maybe which is leaning towards no. Lets ignore the performance and logistics around name lookup and instead focus on the debugger portion of the feature.

The first problem that comes up is which DLL should the debugger be loading?  The PDB can tell us where the DLL's involved in the compilation process were but that has no guaranteed relation to the current running program. Remember the machine the debugee process is running on is potentially (and likely) different than the one it was compiled on. The directory structure of the deployed application can (and will) be very different than the structure of the application when it was compiled. So any information the compiler could embed in the PDB about the path to the DLL is virtually useless.

Even if the paths matched perfectly how could the debugger be certain it was loading the DLL which was referenced in the compilation process over some DLL which just happened to have the same name' True if the DLL was strongly named then a high degree of confidence could be established by doing a strong name check. But now we're limiting the feature to extension methods coming from a strong named DLL. I don't speak for all developers but the number of strong named DLL's I've written is far outweighed by the non-strong named DLLs. With this sort of limitation what good would the feature be?

The next issue is how to reliably load the DLL into the process' The debugger runs in a separate process and can't simply call [LoadLibrary](http://msdn.microsoft.com/en-us/library/ms684175.aspx) to load the DLL. All communication with the debuggee process is done via the CLR and the [debugging interfaces](http://msdn.microsoft.com/en-us/library/ms404484.aspx). They provide no API for directly loading a DLL.  It can be achieved indirectly by executing a function in the process which has the side effect of loading the DLL (say [Assembly::Load](http://msdn.microsoft.com/en-us/library/system.reflection.assembly.load.aspx)). This works but it is limited to situations where function evaluations are available. So scratch broken in native code, many optimized scenarios, etc '

The last issue is whether or not loading the DLL is worth the cost. Loading a DLL into an AppDomain is an operation that cannot be undone short of tearing down the AppDomain. Most applications have a single AppDomain so the effect of this operation lasts for the lifetime of the debugee process. Loading a DLL can produce unwanted side effects in processes which care about DLL load order or simply the set of DLLs in a process (aka any MEF application). In other words evaluating a simple extension method would irreparably change the state of the process! Probably not what the user intended.

Overall this adds up to a limited feature than potentially has a negative impact on user applications and is one the reasons it's not used as a solution here.

Earlier though I said this feature was a 'maybe leaning towards no'. The reason for the 'maybe' part is the feature is possible it's just of questionable value. And it turns out that there are real scenarios where the positive benefits do outweigh the pitfalls. The primary example is the VB expression evaluator. It will force the Visual Basic runtime into the process when certain compiler helper functions are needed. These helper function are needed for a large set of VB expressions (late binding in particular) and hence has a high value to users. The DLL is also strongly named and available in the GAC so there is little fear of loading the wrong DLL. Nor is the introduction of the runtime into the process considered dangerous as it's the default for VB projects.

