# Third-party libraries

An big enough application is likely going to depend on third-party libraries that 
may not have been audited for AOT safety. Some of these libraries may work out of 
the box while others may be problematic. There are generally few options that can 
be employed to attach the issue:

- Update the library to be AOT safe and run it through AOT analyzer. This may
  be the preferred option if the library is open source. It will likely help
  other users of the library as well.
- Use alternative library that is AOT safe. Many popular libraries have forks
  or alternatives that are built with AOT in mind. For example, there's
  [DapperAOT](https://github.com/DapperLib/DapperAOT) for [Dapper](https://github.com/DapperLib/Dapper),
  [NextGenMapper](https://github.com/DedAnton/NextGenMapper) for [Automapper](https://github.com/AutoMapper/AutoMapper)
  (not source compatible), etc.
- Mark the whole library as rooted and not trimmed. This may help alleviate some
  issues but certainly not all of them.

## Updating libraries to be AOT compatible

The general rule of thumb is to start with marking assemblies with `<IsAotCompatible>true</IsAotCompatible>`
in the project files. This will enable the AOT and single-file analyzers and start
warning on problematic APIs.

Typically you will encounter usage of some APIs like `Enum.GetValues(Type)` that 
come with newer generic-shaped alternatives like `Enum.GetValues<T>()`. This can 
be trivially changed as long as the library targets the latest `TargetFramework` 
version. If a down-level compatibility is necessary, wrapping inside `#if NET80_OR_GREATER` 
conditional preprocessor directive is advised.

There's a class of problemes, like the JSON serialization, that is solved with
source generators. These generators run at compile time and generate data and 
methods necessary to avoid reflection at runtime, or at least to scope the 
reflection to types and methods that can be statically analyzed by the AOT 
compiler.

### JSON serialization and deserialization

Updating JSON serialization to use source generators requires to essentially do 
two steps:
- Identify which classes/structs participate in the JSON (de)serialization and
  create `JsonSerializerContext` derived class for them with the `JsonSerializable`
  attribute for each type.
- When doing the serialization itself, use the method overloads that take the
  `JsonTypeInfo<T>` extra parameter. Don't worry, the AOT analyzer will flag
  this for you with appropriate warnings.

And example of the serialization context class would look like this:
```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using MyServiceClient.Data;

namespace MyServiceClient
{
	[JsonSourceGenerationOptions(JsonSerializerDefaults.Web)]
	[JsonSerializable(typeof(TransformRequest))]
	[JsonSerializable(typeof(TransformReply))]
	partial class JsonSourceGenerationContext : JsonSerializerContext
	{
	}
}
```

You may notice the `JsonSourceGenerationOptions` attribute in the example. If you 
used custom `JsonSerializerOptions` in calls to `JsonSerializer.[De]Serialize[Async]`  
then in many of the cases the options can be moved to the `JsonSourceGenerationOptions`
attribute.

Then you just modify your calls to `JsonSerializer.Serialize(transformRequest)` to
`JsonSerializer.Serialize(transformRequest, JsonSourceGenerationContext.Default.TransformRequest)`.
Common JSON abstractions, such as `JsonContent` for HTTP request, or the System.Net.Http.Json 
extension methods also come with overloads that include the `JsonTypeInfo<T>` parameter.

### XML serialization and deserialization

There's currently no source generator for XML serializers like there's one for JSON. The 
options to handle the sitation are limited. We employed <strike>two</strike>three different approaches to handle 
the problem. In some cases the easiest way is to hand write the serialization using the 
`XmlReader`/`XmlWriter` classes. In other cases we crafted a way to use the SGen tools to 
generate the (de)serialization code automatically with minimal refactoring.

For the hand-written parts we took the approach of implementing the `IXmlSerializable` on 
each class. The interface comes with three methods - `ReadXml(XmlReader)`, `WriteXml(XmlWriter)`, 
and `XmlSchema GetSchema()`. For our purposes the last method is irrelevant and can just 
return `null`. The other two methods then implement strongly typed (de)serialization that 
can be called instead of `XmlSerializer.[De]Serialize`.

<strike>
In cases where the amount of XML was too big for the hand-written approach we devised a 
clever trick to use SGen to generate the code and embed it. For those of you not familiar 
with SGen, it is a tool that was available since .NET Framework to pregenerate serialization 
code for `XmlSerializer`. In modern .NET it's now available in the [Microsoft.XmlSerializer.Generator](https://www.nuget.org/packages/Microsoft.XmlSerializer.Generator) NuGet package. Unfortunately, 
the way the generator works and the output is consumed is not AOT friendly. The generator 
takes a managed .dll as an input, produces a .cs file with the serialization code, and then 
compiles the .cs file into another auxiliary .dll that ships alongside the application. At 
runtime the `XmlSerializer` looks if this auxiliary .dll is available and loads it and calls 
into it to perform the serialization. Since NativeAOT doesn't allow dynamic assembly loading 
this is a non-starter.

In order to use SGen we need to restructure the code a bit and then use a couple of MSBuild 
tricks to automate everything. The first step is to split the classes that we want to 
(de)serialize into separate .cs files. In case there's some attached code logic the recommened 
approach is to use `partial` classes and split the logic into its own file. Once we are done 
with that, add a new `<ItemGroup>` into the project and list all the files with the pure 
serialization classes inside as `<SgenCompile Include="MyDataClass.cs">`. Finally, add the 
[MSBuild magic](XmlSerializerGenerator.targets) with `<Import Project="XmlSerializerGenerator.targets" />`  
and make sure that you have the copy of [XmlSerializerGenerator.targets](XmlSerializerGenerator.targets) 
alongside the project.

What does the MSBuild magic do? Good question, it builds a small .dll file just from the 
source files listed in the `SgenCompile` item group. It then proceeds to run SGen on it and 
takes the .cs file from the output. The .cs file gets fixed up to address some warnings and 
then it gets included into a compile item for the main project. For example, if you had a 
`MyDataClass` class then you will get a `MyDataClassSerializer` generated class. The new 
`MyDataClassSerializer` class can now be used in place of `XmlSerializer`. This will 
still generate code that produces AOT warnings but those can be ignored with a local suppression.
</strike>

It turns out that the `XmlSerializer` still depends on reflection even for the source generated 
mapping due to how it internally handles `Mode`. ILLink annotations may be the only way to 
workaround it for now, and at that point there's very little benefit to using the pregenerated 
serializers.

## Alternative AOT-safe libraries

###  HTTP REST API SDKs

A special mention is deserved for libraries that wrap HTTP REST APIs. These APIs often 
come with machine readable description in the OpenAPI format or another JSON Schema 
based specification.

It's common to generate the client SDK for those APIs through an automated generator. 
There are multiple such generators ([NSwag](https://github.com/RicoSuter/NSwag), 
[AutoRest](https://github.com/Azure/autorest), [Refit](https://github.com/reactiveui/refit),
[Microsoft Kiota](https://github.com/microsoft/kiota), and several others). Notably,
these generators differ in the additional dependencies (eg. Newtonsoft.Json vs
System.Text.Json) and various implementation details. Some generators are more suitable 
for AOT than others.

Microsoft Kiota seems to work quite well in the AOT scenarios since serialization and 
deserialization code is generated as strongly typed source code. It doesn't depend on 
the reflection-based serialization in the underlying JSON library. It is essential to 
use the latest version of Kiota libraries though, since AOT specific issues were fixed 
there.

As a proof of concept, we built a [subset of Google API Client SDK](https://github.com/emclient/Google.Apis.Kiota) 
using Microsoft Kiota. The Google Discovery API definitions are first converted into 
OpenAPI. Then the `yq` tool is used to patch up any problematic spot in the OpenAPI
definition file. Lastly, Kiota is used to generate the client API libraries that are 
published as NuGet. All of this is scripted using GitHub Actions to ensure an updated 
SDK can be produced seamlessly when the Google API is updated. We tested replacing 
the Google provided SDK for Gmail, Google Calendar, Google Tasks, Google Drive APIs 
in [eM Client](https://www.emclient.com/) and it was mostly seamless. The only 
problematic part were the resumable upload APIs for Google Drive which are also 
hand-written in the [Google provided client libraries](https://github.com/googleapis/google-api-dotnet-client).

## Consuming AOT-unsafe libraries

There are multiple workarounds that you can use for AOT unsafe libraries, but most of 
them are out of scope for this document.

For cases where excessive code trimming is a problem or fields/methods are accessed by 
reflection, you can instruct the compiler to root additional assemblies, classes, fields, 
or methods. The way to do so is to add them as trimming roots to the main project with
`TrimmerRootAssembly` or `TrimmerRootDescriptor`. This is described in the [ILLink documentation](https://github.com/dotnet/runtime/blob/7ffb9a44ab32cd96d5684c1560671695535ae2cb/docs/tools/illink/illink-tasks.md).

Some technologies, like COM interop on Windows, can be augmented for existing libraries 
without modifying them. For an example of this approach, see the [WinFormsComInterop project](https://github.com/kant2002/WinFormsComInterop).

Libraries that require dynamic assembly loading or rely on dynamic code generarion are 
fundamentally incompatible with NativeAOT. If you heavily rely on those and have no 
alternative, then technologies like CoreCLR ReadyToRun or MonoVM may be a better fit. 
Both of them offer partial AOT augmented by JIT or interpreter to handle the dynamic code.

