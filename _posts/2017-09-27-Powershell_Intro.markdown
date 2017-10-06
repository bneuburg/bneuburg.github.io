---
layout: post
title:  "Introduction to Powershell for Linux admins"
date:   2017-09-27 11:03:22 +0200
categories: powershell terminal
---

Even though I am rather a Linux power user my dayjob as an IT Security guy requires that I also deal with
Windows clients and servers on a regular basis.
Before I switched to Linux full time, I was using Windows XP with the dreaded `cmd.exe` as the only built-in terminal.
Luckily things have changed quite a bit:
1. Microsoft provides a modern command line interface and object-oriented scripting language with Powershell
1. We got bash on Windows (Windows Subsystem for Linux)
1. We have Powershell for Linux
1. We could run Powershell on bash on Windows

So these are quite confusing times. The focus of this post will be on Powershell and to give Linux powerusers an introduction about similarities and differences between commonly used POSIX shells like
bash and zsh and Windows Powershell. Note that even though you can install Powershell on your Linux system the information provided will refer to usage from Windows. Moreover I will not cover installation of Powershell at all and the example code provided was executed on Powershell 5.0.

## Getting started with Powershell


The Powershell developers and product managers made sure that Linux admins will feel welcome. This is highlighted by the fact, that many internal commands are aliased to their counterpart binaries on your standard Linux system. You can e.g. simply run `ls` instead of `DIR` and even `curl` will do what you would expect. To take that welcoming atmosphere even further, they created man-pages and provide a Powerful Tab-completion (even for parameters)!

So fire up Powershell and run `ls`,  `ps > file.txt`, `cp file.txt file.txt.copy`, `cat file.txt` and `rm file.txt.copy`. If you want to learn about options or arguments for a specific command just ask `man`:

{% highlight powershell %}
PS C:\Users\bneuburg> man Get-Help
NAME
    Get-Help

SYNTAX
    Get-Help [[-Name] <string>] [-Path <string>] [-Category <string[]> {Alias | Cmdlet | Provider | General | FAQ | Glossary | HelpFile | ScriptCommand | Function | Filter | ExternalScript | All | DefaultHelp
    | Workflow | DscResource | Class | Configuration}] [-Component <string[]>] [-Functionality <string[]>] [-Role <string[]>] [-Full]  [<CommonParameters>]
...
[5 more parameter sets skipped]
...
{% endhighlight %}

To get a list of currently defined aliases, just run `alias`:

{% highlight powershell %}
CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Alias           % -> ForEach-Object
Alias           ? -> Where-Object
Alias           cat -> Get-Content
Alias           cd -> Set-Location
Alias           copy -> Copy-Item
Alias           cp -> Copy-Item
Alias           cpi -> Copy-Item
Alias           curl -> Invoke-WebRequest
Alias           echo -> Write-Output
Alias           history -> Get-History
Alias           kill -> Stop-Process
Alias           ls -> Get-ChildItem
Alias           man -> help
Alias           mv -> Move-Item
Alias           pwd -> Get-Location
Alias           rm -> Remove-Item
Alias           rmdir -> Remove-Item
Alias           sort -> Sort-Object
Alias           wget -> Invoke-WebRequest
{% endhighlight %}

I truncated the output a little bit to highlight 2 facts:
1. Even though you used `ls` etc. before like you did on Unix, these well known commands are actually aliases for the actual commands (actually not commands but cmdlets, more on that in the next section)
1. Some aliases point to the same command (e.g. `curl` and `wget` are aliases for `Invoke-WebRequest`, `rm` and `rmdir` point to `Remove-Item`)

Before digging a little deeper, be aware that even though many things appear quite familiar, the functionality (e.g. output style or option names) might differ severely from a standard binary on Unix.

## Terminology

To make sure we are on the same page here is some terminology that will pop up when you are dealing with Powershell. Common language is important but be aware that some terms used between Linux users might mean something different when consulting Powershell documentation.

- Cmdlet: Cmdlets are lightweight commands for execution in Powershell. This means that those are not full blown PE/ELF executables but .NET programs designed and compiled for Powershell. Some of them are included in the core of Powershell, so you could almost consider them built-ins; others are included in modules. Most Cmdlets have names matching a Verb-Noun pattern.
- Cmdlet parameters: Parameters of a cmdlet can be required, named, positional and switch (you could say optional) parameters. In Unix those are often called options.
- Cmdlet Parameter sets: Some parameters are mutually exclusive with others, some require additional parameters. The possible combinations of these are called parameter sets. `man` will tell you all parameter sets in the `SYNTAX` section.
- Commands: Commands in Powershell are not only Cmdlets, but also of the following types: Function, Application, Configuration, ExternalScript, Filter, Script, Workflow. More on these types in an upcoming post.
- Host: This is how the interface between the user and the backend interpreter is provided.
- Member: See next section
- Modules: As in other programming languages modules are used to package and namespace a certain functionality. In Powershell installing and loading additional modules means that you have more Cmdlets available. Modules can either be appropriately assembled binaries or scripts with a `.psm1` suffix. In older versions of Powershells before the module style was en vogue the term for module like packages was SnapIn.
- Objects: Powershell Cmdlets work on .NET objects, thus they consume and produce objects (as in object oriented programming).
- Package management: This term describes a symbiosis of Linux distribution package management (e.g. apt/dpkg) and library management for scripting languages (rubygems, CPAN, pip) and can be used to either install Powershell modules or use Powershell code to install applications or setup services.
- Providers: To accommodate the Unix philosophy of "everything is a file" Powershell providers are used to present arbitrary data sources as files or directories (e.g. local certificate store, registry). Besides the built in providers you can create your own. The resulting "directories" presenting the data are called drives.
- Script: A script is a plaintext file - usually with a `.ps1` suffix - that contains Powershell instructions

## Fundamental concepts


### Object Orientation


As noted above Powershell as a .NET product is very reliant on object orientation. This not only manifests in how Cmdlets are named (Verb-Noun, e.g. `Get-Item`, `Remove-Item`) but also in how output can be handled. If you run a command in bash your output will usually be a string, whether it is multiline or not. Thus if you are only interested in a certain part of the output you will usually use string filtering and manipulation tools, such as `grep`, `cut` or `awk`. Let's demonstrate this by looking at the `arp` command on Debian:

{% highlight bash %}
$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.122.244                  (incomplete)                              virbr0
192.168.122.196          ether   12:14:00:d1:b0:10   C                     virbr0
{% endhighlight %}

So even though the output format is clearly defined (c.f. the heading), `arp` won't hand you each arp cache entry as an object, that you can query for certain characteristics. This poses no problem if you are a human looking at it or if you are only interested in the Address field, but once you want to use only the Iface field for subsequent processing, you're gonna have a lot of fun with string manipulation.
In Powershell on the other hand, all Cmdlets return objects. Each of these objects have _Members_ which are most of the time either _Properties_ or _Methods_. You can use the `Get-Member` Cmdlet in order to retrieve information about a returned object, e.g.

{% highlight powershell %}
PS C:\Users\bneuburg> ls file.txt | Get-Member

   TypeName: System.IO.FileInfo

Name                      MemberType     Definition
----                      ----------     ----------
LinkType                  CodeProperty   System.String LinkType{get=GetLinkType;}                                                               
Mode                      CodeProperty   System.String Mode{get=Mode;}
Target                    CodeProperty   System.Collections.Generic.IEnumerable`1[[System.String, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]] Target{get=GetTarget;}
AppendText                Method         System.IO.StreamWriter AppendText()
CopyTo                    Method         System.IO.FileInfo CopyTo(string destFileName), System.IO.FileInfo CopyTo(string destFileName, bool overwrite)
Create                    Method         System.IO.FileStream Create()
CreateObjRef              Method         System.Runtime.Remoting.ObjRef CreateObjRef(type requestedType)
CreateText                Method         System.IO.StreamWriter CreateText()
Decrypt                   Method         void Decrypt()
Delete                    Method         void Delete()
Encrypt                   Method         void Encrypt()
Equals                    Method         bool Equals(System.Object obj)
GetAccessControl          Method         System.Security.AccessControl.FileSecurity GetAccessControl(), System.Security.AccessControl.FileSecurity GetAccessControl(System.Security.AccessControl.AccessControl...
GetHashCode               Method         int GetHashCode()
GetLifetimeService        Method         System.Object GetLifetimeService()
GetObjectData             Method         void GetObjectData(System.Runtime.Serialization.SerializationInfo info, System.Runtime.Serialization.StreamingContext context), void ISerializable.GetObjectData(Syste...
GetType                   Method         type GetType()
InitializeLifetimeService Method         System.Object InitializeLifetimeService()
MoveTo                    Method         void MoveTo(string destFileName)
Open                      Method         System.IO.FileStream Open(System.IO.FileMode mode), System.IO.FileStream Open(System.IO.FileMode mode, System.IO.FileAccess access), System.IO.FileStream Open(System....
OpenRead                  Method         System.IO.FileStream OpenRead()
OpenText                  Method         System.IO.StreamReader OpenText()
OpenWrite                 Method         System.IO.FileStream OpenWrite()
Refresh                   Method         void Refresh()
Replace                   Method         System.IO.FileInfo Replace(string destinationFileName, string destinationBackupFileName), System.IO.FileInfo Replace(string destinationFileName, string destinationBac...
SetAccessControl          Method         void SetAccessControl(System.Security.AccessControl.FileSecurity fileSecurity)
ToString                  Method         string ToString()
PSChildName               NoteProperty   string PSChildName=_timer
PSDrive                   NoteProperty   PSDriveInfo PSDrive=C
PSIsContainer             NoteProperty   bool PSIsContainer=False
PSParentPath              NoteProperty   string PSParentPath=Microsoft.PowerShell.Core\FileSystem::C:\Users\cyborg5\Desktop
PSPath                    NoteProperty   string PSPath=Microsoft.PowerShell.Core\FileSystem::C:\Users\cyborg5\Desktop\_timer
PSProvider                NoteProperty   ProviderInfo PSProvider=Microsoft.PowerShell.Core\FileSystem
Attributes                Property       System.IO.FileAttributes Attributes {get;set;}
CreationTime              Property       datetime CreationTime {get;set;}
CreationTimeUtc           Property       datetime CreationTimeUtc {get;set;}
Directory                 Property       System.IO.DirectoryInfo Directory {get;}
DirectoryName             Property       string DirectoryName {get;}
Exists                    Property       bool Exists {get;}
Extension                 Property       string Extension {get;}
FullName                  Property       string FullName {get;}
IsReadOnly                Property       bool IsReadOnly {get;set;}
LastAccessTime            Property       datetime LastAccessTime {get;set;}
LastAccessTimeUtc         Property       datetime LastAccessTimeUtc {get;set;}
LastWriteTime             Property       datetime LastWriteTime {get;set;}
LastWriteTimeUtc          Property       datetime LastWriteTimeUtc {get;set;}
Length                    Property       long Length {get;}
Name                      Property       string Name {get;}
BaseName                  ScriptProperty System.Object BaseName {get=if ($this.Extension.Length -gt 0){$this.Name.Remove($this.Name.Length - $this.Extension.Length)}else{$this.Name};}
VersionInfo               ScriptProperty System.Object VersionInfo {get=[System.Diagnostics.FileVersionInfo]::GetVersionInfo($this.FullName);}
{% endhighlight %}

Woah! Quite a lot of output. So we see that a `Get-ChildItem` on a single file will return an object of type `System.IO.FileInfo`.
This type has among others a property called `CreationTime` of type `datetime`. We can now simply retrieve it in an object oriented way:
{% highlight powershell %}
PS C:\Users\bneuburg> ls file.txt | Select-Object -ExpandProperty CreationTime

Wednesday, Sep 27, 2017 11:22:33 AM

{% endhighlight %}

Now lets get some details about the `datetime` type:
{% highlight powershell %}
PS C:\Users\bneuburg> ls file.txt | Select-Object -ExpandProperty CreationTime | Get-Member



   TypeName: System.DateTime

Name                 MemberType     Definition
----                 ----------     ----------
Add                  Method         datetime Add(timespan value)
AddDays              Method         datetime AddDays(double value)
AddHours             Method         datetime AddHours(double value)
ToType               Method         System.Object IConvertible.ToType(type conversionType, System.IFormatProvider provider)
ToUniversalTime      Method         datetime ToUniversalTime()
Date                 Property       datetime Date {get;}
Day                  Property       int Day {get;}
DayOfWeek            Property       System.DayOfWeek DayOfWeek {get;}
DayOfYear            Property       int DayOfYear {get;}
Hour                 Property       int Hour {get;}
Kind                 Property       System.DateTimeKind Kind {get;}
Millisecond          Property       int Millisecond {get;}
Minute               Property       int Minute {get;}
Month                Property       int Month {get;}
Second               Property       int Second {get;}
Ticks                Property       long Ticks {get;}
DateTime             ScriptProperty System.Object DateTime {get=if ((& { Set-StrictMode -Version 1; $this.DisplayHint }) -ieq  "Date")...

{% endhighlight %}

Here I truncated the output to maintain minor readability, but again we have an object with Methods and Properties. Let's add some hours:

{% highlight powershell %}
PS C:\Users\bneuburg> $myctime = ls file.txt | Select-Object -ExpandProperty CreationTime
PS C:\Users\bneuburg> ($myctime).Hour
11
PS C:\Users\bneuburg> ($myctime).AddHour(5)

Wednesday, Sep 27, 2017 16:22:33 AM

{% endhighlight %}

It is important to note that even though the host outputs a string that can be copy pasted, this string is a mere representation of the underlying object. So if the representation does not fit your needs, pipe the output to a `Format-*` Cmdlet to tune it accordingly. Of course simple string manipulation, even though in an object oriented way, is not the main use case for this. You can script complex workflows by relying on the objects and their properties and thus avoiding the dangers of using `grep`, `awk` etc. when handling strings, since these tools have no context information on the processed data.

### The Pipeline

The pipe character `|` can be used to create complex pipelines, i.e. sequences of cmdlets that transform or filter the output of the first cmdlet of the pipeline to derive the desired output.

We already saw examples of pipelines in the previous section where we e.g. used the `Get-Member` cmdlet to gain information on the type of object that `Get-ChildItem` (or in short: `ls`) returns.

Even though the basic concept of piping is the same for POSIX shells and Powershell, one major advantage is the ability to use the object orientation in each step. I might dig into this a little deeper in an upcoming post, for now just have a look at this example from the `Restart-Service` man page and guess what it does.

{% highlight powershell %}
PS C:\> Get-Service -Name "net*" | 
  Where-Object {$_.Status -eq "Stopped"} | Restart-Service
{% endhighlight %}

## More tips and tricks

- You can use wildcards with many cmdlets (e.g. `Get-Service -Name "net*"`)
- Powershell does not care about case sensitivity for the most part
- Declare variables with `$variablename =`
- Call methods or retrieve properties on variables with `($variablename).method_or_property`
- To install all man pages, run `Update-Help`, but be aware that you need to be be an administrator in order to do this
- Use the help system and tab completion excessively
- Get a list of available modules with `Get-Module -ListAvailable`
- Environment variables are stored in `$env` and can be accessed by a colon, e.g. `$env:PSModulePath`, or you can do `cd env:; ls`
- Output redirection is similar to `bash`, c.f. `man about_Redirection`
- If you have long strings as output, they will get truncated by Powershell, even when you write them to a file. Use `| Out-String -Width 256` (or an appropriate value)
- If output is still truncated, we are probably talking about arrays or hashes, in these cases run `$FormatEnumerationLimit = -1`
- You can distribute longer pipelines over multiple lines by just adding a new line after the `|` without indicating that the command will continue on the next line, since Powershell syntax always expects more cmdlets after a `|`. The same holds true for opening parentheses
- In other cases you can use the backtick to signal Powershell that the command should continue on the next line
- To see what cmdlets matching a certain pattern (e.g. anything with firewall in its name) available, run `Get-Command *firewall*`

I guess that's a lot to digest, hopefully I'll be able to follow this up soon and walk you through some levels of [Under The Wire][utw] (which by the way provides a sandboxed playground for Powershell that can be accessed via ssh).

[utw]: http://www.underthewire.tech