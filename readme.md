<!--
GENERATED FILE - DO NOT EDIT
This file was generated by [MarkdownSnippets](https://github.com/SimonCropp/MarkdownSnippets).
Source File: /readme.source.md
To change this file edit the source file and then run MarkdownSnippets.
-->

# <img src="/src/icon.png" height="25px"> Newtonsoft.Json.Encryption

[![Build status](https://ci.appveyor.com/api/projects/status/qciwf7sdysdeu693/branch/master?svg=true)](https://ci.appveyor.com/project/SimonCropp/newtonsoft-json-encryption)
[![NuGet Status](https://img.shields.io/nuget/v/Newtonsoft.Json.Encryption.svg?label=Newtonsoft.Json.Encryption)](https://www.nuget.org/packages/Newtonsoft.Json.Encryption/)
[![NuGet Status](https://img.shields.io/nuget/v/Rebus.Newtonsoft.Encryption.svg?label=Rebus.Newtonsoft.Encryption)](https://www.nuget.org/packages/Rebus.Newtonsoft.Encryption/)
[![NuGet Status](https://img.shields.io/nuget/v/NServiceBus.Newtonsoft.Encryption.svg?label=NServiceBus.Newtonsoft.Encryption)](https://www.nuget.org/packages/NServiceBus.Newtonsoft.Encryption/)


Leverages the Newtonsoft extension API to encrypt/decrypt specific nodes at serialization time. So only the nodes that require encryption are touched, the remaining content is still human readable. This approach provides an compromise between readability/debugabaility and security.

<!--- StartOpenCollectiveBackers -->

[Already a Patron? skip past this section](#endofbacking)


## Community backed

**It is expected that all developers either [become a Patron](https://opencollective.com/nservicebusextensions/contribute/patron-6976) or have a [Tidelift Subscription](#support-via-tidelift) to use NServiceBusExtensions. [Go to licensing FAQ](https://github.com/NServiceBusExtensions/Home/#licensingpatron-faq)**


### Sponsors

Support this project by [becoming a Sponsor](https://opencollective.com/nservicebusextensions/contribute/sponsor-6972). The company avatar will show up here with a website link. The avatar will also be added to all GitHub repositories under the [NServiceBusExtensions organization](https://github.com/NServiceBusExtensions).


### Patrons

Thanks to all the backing developers. Support this project by [becoming a patron](https://opencollective.com/nservicebusextensions/contribute/patron-6976).

<img src="https://opencollective.com/nservicebusextensions/tiers/patron.svg?width=890&avatarHeight=60&button=false">

<a href="#" id="endofbacking"></a>

<!--- EndOpenCollectiveBackers -->


## Support via TideLift

Support is available via a [Tidelift Subscription](https://tidelift.com/subscription/pkg/nuget-nservicebus.json.encryption?utm_source=nuget-nservicebus.json.encryption&utm_medium=referral&utm_campaign=enterprise).


<!-- toc -->
## Contents

  * [Encryption Algorithms](#encryption-algorithms)
  * [Decorating properties](#decorating-properties)
  * [Serialized](#serialized)
  * [Supported property types](#supported-property-types)
  * [Usage](#usage)
  * [Breakdown](#breakdown)
    * [Key](#key)
    * [EncryptionFactory and JsonSerializer](#encryptionfactory-and-jsonserializer)
    * [Serialization](#serialization)
    * [Deserialization](#deserialization)
  * [Rebus](#rebus)
  * [NServiceBus](#nservicebus)
  * [Security contact information](#security-contact-information)<!-- endtoc -->


## NuGet packages

  * https://nuget.org/packages/Newtonsoft.Json.Encryption/
  * https://nuget.org/packages/Rebus.Newtonsoft.Encryption/
  * https://nuget.org/packages/NServiceBus.Newtonsoft.Encryption/


## Encryption Algorithms

Any implementation of [SymmetricAlgorithm](https://msdn.microsoft.com/en-us/library/system.security.cryptography.symmetricalgorithm.aspx) is supported.


## Decorating properties

```C#
public class ClassToSerialize
{
    [Encrypt]
    public string Property { get; set; }
}
```


## Serialized

```C#
{
    "Property":"wSayABpFI3g7a/D6gGTq5g=="
}
```


## Supported property types

 * `string`
 * `byte[]`
 * `Guid`
 * `IDictionary<T, string>`
 * `IDictionary<T, byte[]>`
 * `IDictionary<T, Guid>`
 * `IEnumerable<string>`
 * `IEnumerable<byte[]>`
 * `IEnumerable<Guid>`

Note that only the values in a `IDictionary` are encrypted.


## Usage

The full serialize and deserialization workflow:
<!-- snippet: Workflow -->
<a id='snippet-workflow'/></a>
```cs
// per system (periodically rotated)
var key = Encoding.UTF8.GetBytes("gdDbqRpqdRbTs3mhdZh9qCaDaxJXl+e6");

// per app domain
using var factory = new EncryptionFactory();
var serializer = new JsonSerializer
{
    ContractResolver = factory.GetContractResolver()
};

// transferred as meta data with the serialized payload
byte[] initVector;

string serialized;

// per serialize session
using (var algorithm = new RijndaelManaged
{
    Key = key
})
{
    //TODO: store initVector for use in deserialization
    initVector = algorithm.IV;
    using (factory.GetEncryptSession(algorithm))
    {
        var instance = new ClassToSerialize
        {
            Property = "PropertyValue",
        };
        var builder = new StringBuilder();
        using (var writer = new StringWriter(builder))
        {
            serializer.Serialize(writer, instance);
        }

        serialized = builder.ToString();
    }
}

// per deserialize session
using (var algorithm = new RijndaelManaged
{
    IV = initVector,
    Key = key
})
{
    using (factory.GetDecryptSession(algorithm))
    {
        using var stringReader = new StringReader(serialized);
        using var jsonReader = new JsonTextReader(stringReader);
        var deserialized = serializer.Deserialize<ClassToSerialize>(jsonReader);
        Console.WriteLine(deserialized!.Property);
    }
}
```
<sup><a href='/src/Newtonsoft.Json.Encryption.Tests/Snippets/Snippets.cs#L12-L73' title='File snippet `workflow` was extracted from'>snippet source</a> | <a href='#snippet-workflow' title='Navigate to start of snippet `workflow`'>anchor</a></sup>
<!-- endsnippet -->


## Breakdown


### Key

See [SymmetricAlgorithm.Key](https://msdn.microsoft.com/en-us/library/system.security.cryptography.symmetricalgorithm.key.aspx).

Example Key used for [RijndaelManaged algorithm](https://msdn.microsoft.com/en-us/library/system.security.cryptography.rijndaelmanaged.aspx) in the below sample code:

```C#
var key = Encoding.UTF8.GetBytes("gdDbqRpqdRbTs3mhdZh9qCaDaxJXl+e6");
```

A new valid key can be generated by instanitiating a [SymmetricAlgorithm](https://msdn.microsoft.com/en-us/library/system.security.cryptography.symmetricalgorithm.aspx) and accessing [SymmetricAlgorithm.Key](https://msdn.microsoft.com/en-us/library/system.security.cryptography.symmetricalgorithm.key.aspx).


### EncryptionFactory and JsonSerializer

Generally a single instance of `EncryptionFactory` will exist per AppDomain.

A single instance of `EncryptionFactory` is safe to be used for multiple instances of `JsonSerializer`. 

```C#
var factory = new EncryptionFactory();

var serializer = new JsonSerializer
{
    ContractResolver = factory.GetContractResolver()
};
```


### Serialization

A single encrypt session is used per serialization instance.

On instantiation the `SymmetricAlgorithm` will generate a valid [IV](https://msdn.microsoft.com/en-us/library/system.security.cryptography.symmetricalgorithm.iv.aspx). This is generally a good value to use for serialization and then stored for deserialization.

<!-- snippet: serialize -->
<a id='snippet-serialize'/></a>
```cs
// per serialize session
using (var algorithm = new RijndaelManaged
{
    Key = key
})
{
    //TODO: store initVector for use in deserialization
    initVector = algorithm.IV;
    using (factory.GetEncryptSession(algorithm))
    {
        var instance = new ClassToSerialize
        {
            Property = "PropertyValue",
        };
        var builder = new StringBuilder();
        using (var writer = new StringWriter(builder))
        {
            serializer.Serialize(writer, instance);
        }

        serialized = builder.ToString();
    }
}
```
<sup><a href='/src/Newtonsoft.Json.Encryption.Tests/Snippets/Snippets.cs#L29-L53' title='File snippet `serialize` was extracted from'>snippet source</a> | <a href='#snippet-serialize' title='Navigate to start of snippet `serialize`'>anchor</a></sup>
<!-- endsnippet -->


### Deserialization

A single decrypt session is used per serialization instance.

 * `key` must be the same as the one use for serialization.
 * `initVector` must be the same as the one use for serialization. It is safe to be transferred with the serialized text. 


<!-- snippet: deserialize -->
<a id='snippet-deserialize'/></a>
```cs
// per deserialize session
using (var algorithm = new RijndaelManaged
{
    IV = initVector,
    Key = key
})
{
    using (factory.GetDecryptSession(algorithm))
    {
        using var stringReader = new StringReader(serialized);
        using var jsonReader = new JsonTextReader(stringReader);
        var deserialized = serializer.Deserialize<ClassToSerialize>(jsonReader);
        Console.WriteLine(deserialized!.Property);
    }
}
```
<sup><a href='/src/Newtonsoft.Json.Encryption.Tests/Snippets/Snippets.cs#L55-L72' title='File snippet `deserialize` was extracted from'>snippet source</a> | <a href='#snippet-deserialize' title='Navigate to start of snippet `deserialize`'>anchor</a></sup>
<!-- endsnippet -->


## Rebus

<!-- snippet: RebugsUsage -->
<a id='snippet-rebugsusage'/></a>
```cs
var activator = new BuiltinHandlerActivator();

activator.Register(() => new Handler());
var configurer = Configure.With(activator);

var encryptionFactory = new EncryptionFactory();
var settings = new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.All,
    ContractResolver = encryptionFactory.GetContractResolver()
};
configurer.Serialization(s => { s.UseNewtonsoftJson(settings); });
configurer.EnableJsonEncryption(
    encryptionFactory: encryptionFactory,
    encryptStateBuilder: () =>
    (
        algorithm: new RijndaelManaged
        {
            Key = key
        },
        keyId: "1"
    ),
    decryptStateBuilder: (keyId, initVector) =>
        new RijndaelManaged
        {
            Key = key,
            IV = initVector
        });
```
<sup><a href='/src/Rebus.Newtonsoft.Encryption.Tests/Snippets/Snippets.cs#L14-L43' title='File snippet `rebugsusage` was extracted from'>snippet source</a> | <a href='#snippet-rebugsusage' title='Navigate to start of snippet `rebugsusage`'>anchor</a></sup>
<!-- endsnippet -->


## NServiceBus

<!-- snippet: NsbUsage -->
<a id='snippet-nsbusage'/></a>
```cs
var configuration = new EndpointConfiguration("NServiceBusSample");
var serialization = configuration.UseSerialization<NewtonsoftSerializer>();
var encryptionFactory = new EncryptionFactory();
serialization.Settings(
    new JsonSerializerSettings
    {
        ContractResolver = encryptionFactory.GetContractResolver()
    });

configuration.EnableJsonEncryption(
    encryptionFactory: encryptionFactory,
    encryptStateBuilder: () =>
    (
        algorithm: new RijndaelManaged
        {
            Key = key
        },
        keyId: "1"
    ),
    decryptStateBuilder: (keyId, initVector) =>
        new RijndaelManaged
        {
            Key = key,
            IV = initVector
        });
```
<sup><a href='/src/NServiceBus.Newtonsoft.Encryption.Tests/Snippets/Snippets.cs#L11-L37' title='File snippet `nsbusage` was extracted from'>snippet source</a> | <a href='#snippet-nsbusage' title='Navigate to start of snippet `nsbusage`'>anchor</a></sup>
<!-- endsnippet -->


## Security contact information

To report a security vulnerability, use the [Tidelift security contact](https://tidelift.com/security). Tidelift will coordinate the fix and disclosure.


## Icon

[Lock](https://thenounproject.com/term/lock/59296/) designed by [Mourad Mokrane](https://thenounproject.com/molumen/) from [The Noun Project](https://thenounproject.com).
