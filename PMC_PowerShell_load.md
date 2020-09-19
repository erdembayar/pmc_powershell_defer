# PackageManager Console in VS

* Status: **Discussion**
* Author(s): [Erick Yondon](https://github.com/erdembayar)

## Issue

[9988](https://github.com/NuGet/Home/issues/9988) - NuGet PMC memory usage - investigate decreasing the amount of dlls loaded initially

## Background

Child of https://devdiv.visualstudio.com/DevDiv/_workitems/edit/1167790.

To goal is to figure out there is a quick win as far as assembly loading goes.
If the PMC is visible when VS shuts down, when VS starts, it's loaded even with no solution loaded.
This 75 M memory penalty is incurred even if user never uses the PMC in that session.

Please lazily initialize and DontForceCreate. This will reduce the penalty for many sessions with very little change.
- We investigated last request with [Issue#9987](https://github.com/NuGet/Home/issues/9987): We [can implemenent it (PR#3657)](https://github.com/NuGet/NuGet.Client/pull/3657) but since it's customer user experience breaking change we `decided not to do` it for now.

## Who might be affected

All VS customers, because anyone can open PMC from menu.

## Investigation
Here is break down of loaded DLLs when I open PMC, looks Powershell related Dlls are most of them.

Please note size of DLLs don't have exact 1:1 relation how much memory it occupy, and there are many other overheads, so actual memory footprint could be be more than below.
On my PC with VS 2019 I saw about `50MB` difference (after GCed) once I open PMC for both MVC and OrchardCore, not `75MB`.

```xml
NuGet dlls:
NuGet.Common.dll - 113k
NuGet.Configuration.dll - 154k
NuGet.Console.dll - 137k
NuGet.PackageManagement.dll - 387k
NuGet.PackageManagement.VisualStudio.dll - 580k
NuGet.Packaging.dll - 651k
NuGet.ProjectModel.dll - 208k
NuGet.SolutionRestoreManager.dll - 1.674k
NuGet.Versioning.dll - 51k
NuGet.VisualStudio.Common.dll- 141
NuGet.VisualStudio.Implementation.dll - 213k
Total:4.309Kb
```

``` Nomination Solution open:
NuGet.Commands.dll - 550k
NuGet.DependencyResolver.Core.ni.dll - 90k
NuGet.Frameworks.ni.dll- 119k
Nuget.LibraryModel.ni.dll - 56k
Nuget.VisualStudio.ni.dll - 35k
NuGet Nomination only total: 850Kb
Nuget + Nomination total: 4309Kb + 850Kb = 5.159Kb
```

```xml
PowerShell dlls:
Microsoft.PowerShell.Commands.Diagnostics.dll -> 89k
Microsoft.PowerShell.Commands.Management.dll -> 573k
Microsoft.PowerShell.Commands.Utility.dll -> 6.609k
Microsoft.PowerShell.ConsoleHost.dll -> 199k
Microsoft.PowerShell.Security.dll -> 91k
NuGet.PackageManagement.PowerShellCmdlets.dll ->135k
NuGet.PackageManagement.UI.dll -> 184
NuGet.Tools.dll -> 170k ?
Microsoft.WSMan.Management.dll -> 274k
Microsoft.MSMan.Runtime.ni.dll -> 14Kb
System.Management.Automation.ni.dll -> 6.282kb
NuGetConsole.Host.PowerShell.dll -> 111k (default domain)
Powershell only: 14.731k
```

## Questions and answers

#### How does PMC(+Powershell) get loaded currently? In general when it gets focus.
1. `Open PMC through View -> Other Windows -> Package Manager Console.`
   
	It gets loaded because now it's focused.

2. `Have PMC available in the window toolbar, and loads only when you click on it to focus.`
 
   If you don't change focus into it then it doesn't get loaded.
   One exception: It could happen automatically if PMC windows was focused by the time user close the VS instance and then VS re-opens next time then 
   PMC(+Powershell) is still get loaded.
   But still above is not very common because once user start debugging or build solution then focus automaticly change to `Output` window, so it's very unlikely
   `PMC` window would be last focused window before user close solution. When you're testing then it creates false sense that PMC always open if you don't do other popular actions like 
   'Build', `Debug` etc. Once you do any of those actions then window focus switches to `Error List`, or `Output`, or `Breakpoints`, or `Find Results`....

`So in short most of likely PMC(+Powershell) won't get loaded unless user is initiated.`

#### How init.ps1 and Install.ps1 fit into this picture?
1. `Install.ps1` : 

   If you install EF6 package from PM UI then this action automatically loads Powershell related dlls even though PMC never opened. 
   Of course you can do install it from PMC too.
2. `init.ps1` : 

   For normal build, run, debug doesn't require user to PMC open nor Powershell to load, it works normally.
   Most common scenario for `init.ps1` need is if user change model of your entity in EF6 then user need do migration which requires user to open PMC.

#### How often do users open PMC? 
Fernando created opened this [query](onenote:https://microsoft.sharepoint.com/teams/NuGet/Team/NuGet/Data.one#NuGet%20PMC%20Count%20raking%20over%20VS%20Windows&section-id=%7B9B2B943B-C74F-4D7B-814C-98E23759A79B%7D&page-id=%7B27C1DD1F-D935-466C-9F7B-130A0CB69D4E%7D&end)
According to this PMC tanks `86-th` most opened window during `last week`.

#### How often do users open PMC?
Fernando created this [query](https://dataexplorer.azure.com/clusters/ddtelvsraw/databases/VS?query=H4sIAAAAAAAAA7VWwW4TMRC99yusXJKK1KRVG6hQkKoKKg4g1EI5IFTN2pNdq1578dibBPHxjDeB0kjZthIcs3nz5s3zzNiXsHjToot0fbX3UywqDCjOdAtOob7C0GL4ZGqkCHXzOSrxeiag9KNDvf8HfV6Bc2jfaTGbieG1oQT2KiZtvDycyku0CITDP/Au2weoMcMHLT1vLMS5D/XzhXHaL2pwUGLNoM2HeWDw4C5+idcYyHgnWFWItDCxEgNO9SKDsAWbIKIooLxJrgF1O/oYfIMhGqSxMKXzAfXdp5leOaiNGn3dGyjvIi6j1DiHZKNsSSpGy8hF1BjDChojm+B1UrFdixiMd4clwiANMXnw5OfRMCw4sI+I8U1EbXrZa1CgdUCiCqjqQ+ISH6GW+SrjUBrdh/IkH8FlvQKLv61a/3L5GB+qPK6aXlAFQS8gYL9ILvhR2fpZOgh3CXk+M/MDIhfd0jqGbc/DoaUGHg2v0eaYgLZDPZ23P0KtR4xHw8x5FndK4KExcdXjIRS4bOTcmrKK1N8MD6Qin4LqdXgjur8yQ9PjwkRuFMWN3Iek2/TQmVKF1mZ59zRvdLDrvUqI8/NBBG/7q+JFCLxEtozZRG99xbzr+Nu3/f0dq+k9L8cUHlpMw95WyuOVe0iW6I0ejveGO1uDVIU1bAY4I3cRF8lY7VJdYOiDbc5tK2vMN4bhG2TjSrend+iivJpY2b2/Cb8n5PiNgLV9lOoagvmBe0KcqWhazDfTVYQoZoI9U7lEiyP+WwjlbardDffw0lCkEV808vdFI7cvGul4PdvUeahZeekkdAlyIYOxmOx3nM+eyPqfOYtUblOOxUnHO/7kI9j/aA9VfuH+rTv/ljKbc59x7U2xEm/ze+IiGX6vPJH/r/eILFOedTEcdq3pQxRMvdWWXJkSLllLwgLFvAKWEZ3uirsEd2tcySKCX9ysO320rnv8jj6+P/+QLjB+6VIy6C/V/MqagJ68mJxOD4oCTg+OTyeTA5ieHB5MTk5f6qPpfKqPju+eXNt0r34B6awOw/UJAAA=).
According to this PMC ranks `841-th most active time spent` window among all windows by the time query ran.
On a typical session, users `spend 30 sec working on` the PMC windows and `it stays open around 4.6 mins`. 
I ran same query for last [3 days timeframe](https://microsoft.sharepoint.com/teams/NuGet/_layouts/OneNote.aspx?id=%2Fteams%2FNuGet%2FTeam%2FNuGet&wd=target%28Data.one%7C9B2B943B-C74F-4D7B-814C-98E23759A79B%2FNumber%20of%20Window%20frames%20from%20RawEvents%203%20days%7CD1F07042-9ED2-43EE-A89F-69F22FE790D3%2F%29) instead of 1 day then PMC was ranking `1278-th` from `3265 different` windows. On a typical session, users spend `33 sec working active on` the PMC windows and it stays `open around 4.58 mins`. 

## Open questions
Below questions came from github and teams's threads.
1. I'm also curious about the error scenarios, when the powershell host can't be initialised. Now, the user never gets a prompt, just an error 
message in the PMC window. With a change like this, they'll get a (fake) prompt, and then as soon as they start to use it, then they get 
an error. Does that seem acceptable?
2. How often users execute commands on PMC once it's open? See below `Proposed soliution #2`.
3. Do we need PMC without solution? If yes how often it happens?

## Proposed Solutions

1. Do nothing.

   Since it's not really high ranking. It's just 86-th most opened and 841-th most active time spent window. Even `Have PMC available in the window toolbar` it loads only when get focused and it's ranking very low time user actively spent. 
 Here I believe `focused` and `user actively spend` are very 
correlated concepts.

2. Add telemetry track how many times users execute commands.
	
	I have [PR#3674](https://github.com/NuGet/NuGet.Client/pull/3674) for tracking how often users actually execute commands on once PMC is opened. Because it could be open
and user doesn't execute anything whole duration. It could happen inadvertently because if PMC windows was focused by the time user close the solution and then VS re-opens it next time and powershell is still loaded.
But still above is not common scenario because once user start debugging or build solution then focus automaticly change to `Output` window.

3. Defer loading of Powershell load even though PMC is opened, then only start loading it when user start typing or paste command on it.
 
   I have [PR#3667](https://github.com/NuGet/NuGet.Client/pull/3667) which deferres load of Powershell load even though PMC is still open, 
and it only starts loading Powershell dlls when user start typing or paste something.

	Some measurements after this change.
	
	1. MVC+EF6 solution loaded: `719Mb`

	2. PMC open: `731MB` (increase by `12MB`)

		Nuget.Packagement.UI

		NuGetConsole.Host.PowerShell.dll

		System.Management.Automation.ni.dll

	3. Powershell loaded when user start typing:`754MB` (increase by `23MB`, which is deferred Powershell load.)
   
		Microsoft.PowerShell.Commands.Diagnostics.dll -> 89k

		Microsoft.PowerShell.Commands.Management.dll -> 573k

		Microsoft.PowerShell.Commands.Utility.dll -> 6.609k

		Microsoft.PowerShell.ConsoleHost.dll -> 199k

		Microsoft.PowerShell.Security.dll -> 91k

		NuGet.PackageManagement.PowerShellCmdlets.dll ->135k

		Microsoft.WSMan.Management.dll -> 274k

		Microsoft.MSMan.Runtime.ni.dll -> 14Kb
