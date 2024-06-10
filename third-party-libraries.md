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

TODO: Write about hand-written serialization and how to use SGen

## Consuming AOT-unsafe libraries

TODO: Trimming rooting, descriptors, dynamic code, COM, etc.