# Journey to the Center of the NativeAOT

_Looking back to all that has occurred to me since that eventful day, I am scarcely able to believe in the reality of my adventures. They were truly so wonderful that even now I am bewildered when I think of them._

We are not going to go to the center of the Earth as the advanturers in the Jules Verne novel, but we are going on a journey of our own that also goes layer by layer until we reach the core.

In this write up we will describe how we took a rather large .NET application and carefully modified it to compile into native code with the NativeAOT compiler. We will also look deeper into the NativeAOT tooling itself to show how it behaves on such a large application, where are the shortcomings, and what lessons can be learned to improve the compiler itself.

## Use case

We started with a rather large application called [eM Client](http://www.emclient.com). In order to understand the rest of the story we first need to describe what it is, how is the application composed, what technologies it relies on, and how all this fits the overall picture.

The application itself is an email, calendaring and chat client. If you familiar with Mozilla Thunderbird or Microsoft Outlook, it may give you an idea of the scope of features that such application presents.

At the forefront, there is a desktop frontend written in the WinForms toolkit with many custom UI controls. The desktop version is available on Windows (win-x86) and macOS (osx-arm64, osx-x64). For macOS we use a custom implementation of the WinForms toolkit that is built on top of the macOS AppKit / Cocoa API and wraps the native UI controls. Since we need to display HTML content we also rely on Chromium Embedded Framework (on Windows) or WebKit (on macOS).

There's also a mobile frontend written in Xamarin.Forms / .NET MAUI (at the time of this writing the upgrade is still in progress; both builds run on .NET 8 though). This frontend currently only comes with the email related funcionality.

Rest of the code base is shared between the desktop and mobile versions. There's 150+ projects covering different areas, such as data storage, data synchronization, various protocols (ranging from TCP-based SMTP, IMAP and POP3 to JSON/XML-over-HTTP based ones), and also many file format parsers (EML, PST, iCalendar, dozen of other proprietary formats for import).

The code base also depends on many 3rd-party SDKs, such as those for [Microsoft Graph](https://github.com/microsoftgraph/msgraph-sdk-dotnet), [Google Workspace](https://github.com/googleapis/google-api-dotnet-client), and others.

## First steps

In order to prepare the application to compile with NativeAOT the good first step is to always start with analyzers. They can be collectively enabled on per-project basis with the `<IsAotCompatible>true</IsAotCompatible>` project switch.

We started the journed by marking 130+ of our assemblies and fixing all the warnings reported by the analyzers. The most common culprits and solutions are mentioned in [this section](https://github.com/filipnavara/nativeaot-notes/blob/main/third-party-libraries.md#updating-libraries-to-be-aot-compatible).

Next up, we devised strategies on how to deal with [third-party libraries](https://github.com/filipnavara/nativeaot-notes/blob/main/third-party-libraries.md).

Fundamentally, this work is benefiting not just NativeAOT but also the MonoVM AOT we already use in the mobile app. The analyzers successfully caught a code that was problematic and caused exceptions at runtime.

## UI frameworks

### WinForms

WinForms and NativeAOT are not an officially supported scenario yet. There is an [ongoing effort to remove the blockers](https://github.com/dotnet/winforms/issues/4649). The scope of the work is rather wide though and it's been spaning multiple releases.

[One of the issues](https://github.com/dotnet/winforms/issues/10813) is that the framework relies on non-string resources. These are either deserialized using the binary formatter or the `TypeConverter`s (part of System.ComponentModel). Both approaches are problematic for different reasons. Binary formatter is going through deprecation cycle over last few .NET releases. In .NET 9 it's finally removed from the core class libraries. A work has been done to replace it with a [NrbfDecoder](https://github.com/dotnet/runtime/pull/103232) to handle reading of pre-existing data. As part of the prototyping for the `NrbfDecoder` API shape the usage has been validated on WinForms use cases. `TypeConverter`s depend on reflection and the NativeAOT compiler doesn't have intrinsic understanding of the WinForms framework necessary to figure out which types are going to be accessed by the reflection-based logic. The solution to this problem is described in detail in the [TypeDescriptor-related trimming support issue](https://github.com/dotnet/runtime/issues/101202).

Second issue is the dependency on COM interop for features like accessibility or open/save dialogs. NativeAOT doesn't support the traditional COM interop with class attributes that was present in .NET since the inception. Instead, it depends on a more modern [ComWrappers API](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/tutorial-comwrappers) introduced in .NET 5. Early pioneering work on the new WinForms COM interop was done by [Andrii Kurdiumov](https://github.com/kant2002) and he has written [informative blog posts](https://codevision.medium.com/using-com-in-nativeaot-131dbc0d559e) about it. This is currently an area of focus in .NET 9.

It seems that most of the efforts may culminated in the .NET 9 release. Current previews (Preview 5, as of this writing) still suffer from some issues but they already got the basic scenarios up and running.

As part of some earlier work we did in effort to speed up the application startup, we also tried to move some of the serialized resources back into code. WinForms uses an all-or-nothing localization model. One either has all the layout and strings in the generated `.Designer.cs` file, or the layout parameters and strings are serialized in the `.resx` resource file with most of the code in `.Designer.cs` just building the UI control tree and calling `ApplyResources` on each control. If the application is designed with flexible layouts, like ours, then the layouts are identical across all localizations and only the strings are different. C# 12 added a feature called [interceptors](https://devblogs.microsoft.com/dotnet/new-csharp-12-preview-features/) to the source generator toolbox. Thus, an idea was born - what if we made a source generator that reads the .resx files, generates the designer code in C# (like a non-localizable app would do), and only peeks into the resources for strings. We did just that with the [Intercepforms experiment](https://github.com/emclient/Intercepforms). You just add a NuGet and the source generator intercepts every `ApplyResources` call in the `InitializeComponent` method in the `.Designer.cs` file with a compiled method. While it didn't necessarily make a huge difference in startup time in CoreCLR, it inadvertently has the side effect of removing some of the TypeConverter reflection.

### Xamarin.Forms / .NET MAUI

Both of these mobile frameworks had a pre-existing support for use with ILLink to trim the code to some 
extent. This was gradually improved with every .NET release and particularly with .NET 8 which got the 
experimental NativeAOT/iOS support.

## Platform support

For the desktop part of the application, we publish for the osx-arm64, osx-x64, and win-x86 runtime identifiers. The macOS support was added in .NET 8, while the Windows x86 support only landed in usable form in .NET 9 Preview 4.

### macOS

Early in the .NET 8 product cycle we resurrected the existing bits of the macOS support in NativeAOT. This work was soon followed by [Austin Wise](https://github.com/AustinWise) adding support for Objective-C interop API. This was soon picked up by the .NET and Xamarin.iOS teams and extended to support the platform including the full iOS/macOS API bindings which were previously only available in CoreCLR and MonoVM.

As part of the NativeAOT/iOS bring-up few important technologies like [Managed Static Registrar](https://github.com/xamarin/xamarin-macios/blob/main/docs/managed-static-registrar.md) were developed that were crucial to get an application of this size running. Notably, these improvements also significantly helped performance of interop scenarios on other runtimes.

One peculiar thing about the macOS support in NativeAOT is that if you target the `net8.0-macos` framework the build process includes an extra ILLink step. The responsibilities of ILLink and NativeAOT overlap in many ways. They both do trimming but they slight behavioral differences that may be observable in some scenarios. We hit one such difference on our macOS version of the System.Drawing library. The library itself is an automatically generated type forwarder. All the implementations of the drawing primitives live in System.Drawing.Common or System.Drawing.Primitives libraries. Why is the System.Drawing library necessary at all then? To answer that we need to go back to the WinForms section above. The resource files reference .NET types by name. For example, you may encounter something like this in the .resx file:

```xml
  <data name="user_gray" type="System.Resources.ResXFileRef, System.Windows.Forms">
    <value>../Resources/user_gray.png;System.Drawing.Bitmap, System.Drawing, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a</value>
  </data>
```

The referenced type is in the System.Drawing assembly. The assembly is auto-generated and it contains a code similar to:

```csharp
[assembly: TypeForwardedTo(typeof(System.Drawing.Bitmap))]
```

That in turns forwards to the actual implementation in System.Drawing.Common:

```csharp
	[System.Runtime.CompilerServices.TypeForwardedFrom("System.Drawing, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a")]
	public sealed class Bitmap : Image { ... }

	[TypeConverter(typeof(ImageConverter))]
	[System.Runtime.CompilerServices.TypeForwardedFrom("System.Drawing, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a")]
	public abstract class Image : MarshalByRefObject, IDisposable , ICloneable, ISerializable { ... }
```

If you didn't skip the WinForms section, you likely see what is going on. At runtime, the resource deserialization needs to resolve the type name into a type. It then examines the attributes on the type, and finds the `TypeConverter` attribute. This tells it to instantiate the `ImageConverter` class and finally use that to interpret the data in the resources.

Now, let's get back to the ILLink and NativeAOT little difference. ILLink strips the `TypeForwardedTo` annotations from the trimmed assemblies. The intention is good. As far as ILLink is concerned, all the references to `Bitmap` in the code are rewritten to the real implementation assembly, and so the attribute is no longer needed. It doesn't have the intrinsic knowledge of the resource resolution logic, and hence doesn't see that it will still be required at runtime. NativeAOT seems to keep the information about forwarded types though.

The build process for `net8.0-macos` runs the ILLink first and then pipes the output into NativeAOT compiler. This has the unfortunate side effect of losing the the `TypeForwardedTo` attributes. The fix is adding a simple `<TrimmerRootAssembly Include="System.Drawing" />` item into the main project and we can move on.

### Windows

32-bit support for ILC targets was only added in .NET 9 Preview 5, as previously stated. We produce the x86 binaries for variety of reason but the biggest one being the burden of distributing the Chromium Embedded Framework which we use for displaying HTML content. Other apps may have similar requirements to produce 32-bit binaries to consume external libraries, or to build components that run inside 32-bit contexts.

Unlike CoreCLR (as of .NET 9 Preview 5), the NativeAOT port for win-x86 uses funclet based exception handling instead of the legacy SEH code. The funclet exception handling was easier to implement in the NativeAOT runtime because it shares code with most of the other targets. 
As exception handling is largely rewritten in CoreCLR for .NET 9 to bring it closer to NativeAOT and to improve performance, it's likely that in the future this topic will be revisited and the approaches unified. CoreCLR historically offers interoperabilty with SEH exceptions thrown in interop scenarios while NativeAOT never supported this. This may prove to be a challenge in any future unification endeavours.

### Undocumented limits

One thing of note is that non-X86/X64 platforms often come with some limits imposed by the instruction set. While in most cases this is handled transparently to the developer there are cases where one needs to be aware of those limits. Due to tooling bugs these limits may result in user-visible errors, degraded performance or larger executable file size.

A good example of such limit are the relative branches in the generated code. Notably, ARM64 has +/- 128 MiB limit for those branches and ARM32 has +- 16 MiB limit in the Thumb2 instruction encoding. Any longer branch in the code needs to be encoded with different set of instructions, or handled through a thunk. Thunk, in this case, is a piece of code that's within the relative code location limit and that contains the longer jump sequence. These thunks are typically produced by the platform linker but in some cases the compiler can generate them too. ILC, the NativeAOT compiler, currently doesn't generate them at the expense of generating more pessimistic code.

Another set of limits is imposed by the object file format - eg. section sizes or file offset represented using 32-bit data types - limiting the output executable size to roughly 2 GiB. These limits differ by platform and the 2 GiB figure is not perfectly accurate but it's a good ballpark figure.

Lastly, one undocumented limit is the size of the unwinding section on Apple platforms (macOS/iOS/tvOS). The unwinding section is used for exception handling, producing stack traces, and for garbage collection. We will talk about this particular limit later in the section dedicated to [Object Writer](ilc-resource-usage.md#object-file-emitting).

## Main project

TODO: PublishAot, feature switches

## Compilation

The first thing you will notice is that the compilation is taking rather long time. It uses a lot of memory, and the output may not be small either. People of curious mind may wonder what is going on, and we will try to answer at least part of that.

Let's talk about [ILC CPU and memory usage](https://github.com/filipnavara/nativeaot-notes/blob/main/ilc-resource-usage.md).

Interested in reducing the size of the compiled app? Check out [Sizoscope](https://github.com/MichalStrehovsky/sizoscope).

TODO: Size overall, 700+MB -> 250-270MB
TODO: Size variation between macOS/Windows - libs, but also prolog+epilog on ARM leaf functions