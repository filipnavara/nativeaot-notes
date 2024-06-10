# Journey to the Center of the NativeAOT

Looking back to all that has occurred to me since that eventful day, I am scarcely able to 
believe in the reality of my adventures. They were truly so wonderful that even now I am 
bewildered when I think of them.

We are not going to go to the center of the Earth as the advanturers in the Jules Verne 
novel, but we are going on a journey of our own that also goes layer by layer until we 
reach the core.

In this write up we will describe how we took a rather large .NET application and carefully 
modified it to compile into native code with the NativeAOT compiler. We will also look 
deeper into the NativeAOT tooling itself to show how it behaves on such a large application, 
where are the shortcomings, and what lessons can be learned to improve the compiler itself.

## Use case

We started with a rather large application called [eM Client](http://www.emclient.com). In 
order to understand the rest of the story we first need to describe what it is, how is the 
application composed, what technologies it relies on, and how all this fits the overall 
picture.

The application itself is an email, calendaring and chat client. If you familiar with 
Mozilla Thunderbird or Microsoft Outlook, it may give you an idea of the scope of features 
that such application presents.

At the forefront, there is a desktop frontend written in the WinForms toolkit with many 
custom UI controls. The desktop version is available on Windows (win-x86) and macOS 
(osx-arm64, osx-x64). For macOS we use a custom implementation of the WinForms toolkit 
that is built on top of the macOS AppKit / Cocoa API and wraps the native UI controls. 
Since we need to display HTML content we also rely on Chromium Embedded Framework (on 
Windows) or WebKit (on macOS).

There's also a mobile frontend written in Xamarin.Forms / .NET MAUI (at the time of this 
writing the upgrade is still in progress; both builds run on .NET 8 though). This frontend 
currently only comes with the email related funcionality.

Rest of the code base is shared between the desktop and mobile versions. There's 150+ 
projects covering different areas, such as data storage, data synchronization, various 
protocols (ranging from TCP-based SMTP, IMAP and POP3 to JSON/XML-over-HTTP based ones), 
and also many file format parsers (EML, PST, iCalendar, dozen of other proprietary 
formats for import).

The code base also depends on many 3rd-party SDKs, such as those for [Microsoft Graph](https://github.com/microsoftgraph/msgraph-sdk-dotnet),
[Google Workspace](https://github.com/googleapis/google-api-dotnet-client), and others.

## First steps

In order to prepare the application to compile with NativeAOT the good first step is 
to always start with analyzers. They can be collectively enabled on per-project basis 
with the `<IsAotCompatible>true</IsAotCompatible>` project switch.

We started the journed by marking 130+ of our assemblies and fixing all the 
warnings reported by the analyzers. The most common culprits and solutions are mentioned 
in [this section](https://github.com/filipnavara/nativeaot-notes/blob/main/third-party-libraries.md#updating-libraries-to-be-aot-compatible).

Next up, we devised strategies on how to deal with [third-party libraries](https://github.com/filipnavara/nativeaot-notes/blob/main/third-party-libraries.md).

## UI frameworks


## Main project

TODO: PublishAot, feature switches

## Compilation

The first thing you will notice is that the compilation is taking rather long time. It uses a 
lot of memory, and the output may not be small either. People of curious mind may wonder what 
is going on, and we will try to answer at least part of that.

Let's talk about [ILC CPU and memory usage](https://github.com/filipnavara/nativeaot-notes/blob/main/ilc-resource-usage.md).

TODO: Sizoscope
