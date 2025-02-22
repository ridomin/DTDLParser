﻿# Inspect object model using advanced technques asynchronously

The `ModelParser` class is used to determine whether one or more DTDL models are valid, to identify specific modeling errors, and to enable inspection of model contents.
This tutorial walks through the last of these uses: how to access elements and properties in the object model.
This is an advanced addendum to the [Inspect object model asynchronously](./Tutorial04_InspectObjectModelAsync.md) tutorial.
A [synchronous version](./Tutorial05_InspectObjectModelAdvanced.md) of this tutorial is also available.

## Create a ModelParser

To parse a DTDL model, you need to instantiate a `ModelParser`.
No arguments are required.

```C# Snippet:DtdlParserTutorial05Async_CreateModelParser
var modelParser = new ModelParser();
```

## Obtain the JSON text of a DTDL model

The DTDL language is syntactically JSON.
The `ModelParser` expects a single string or an asynchronous enumeration of strings.
The single string or each value in the enumeration is JSON text of a DTDL model.

```C# Snippet:DtdlParserTutorial05Async_ObtainDtdlText
string jsonText =
@"{
  ""@context"": ""dtmi:dtdl:context;3"",
  ""@id"": ""dtmi:example:anInterface;1"",
  ""@type"": ""Interface"",
  ""contents"": [
    {
      ""@type"": ""Property"",
      ""name"": ""expectedDistance"",
      ""schema"": ""double""
    },
    {
      ""@type"": ""Telemetry"",
      ""name"": ""currentDistance"",
      ""schema"": ""double""
    }
  ]
}";
```

## Submit the JSON text to the ModelParser

The main asynchronous method on the `ModelParser` is `ParseAsync()`.
One argument is required, which can be either a string or an asynchronous enumeration of strings containing the JSON text to parse as DTDL.

```C# Snippet:DtdlParserTutorial05Async_CallParseAsync
var parseTask = modelParser.ParseAsync(jsonText);
```

The return value is a `Task`, whose completion must be awaited before proceeding.
If the submitted model is complete and valid, no exception will be thrown.
Proper code should catch and process exceptions as shown in other tutorials such as [this one](Tutorial02_FixInvalidDtdlModelAsync.md), but for simplicity the present tutorial omits exception handling.

```C# Snippet:DtdlParserTutorial05Async_CallWait
parseTask.Wait();
IReadOnlyDictionary<Dtmi, DTEntityInfo> objectModel = parseTask.Result;
```

## Display elements in object model

The object model is a collection of objects in a class hierarchy rooted at `DTEntityInfo`.
All DTDL elements derive from the DTDL abstract type Entity, and each DTDL type has a corresponding C# class whose name has a prefix of "DT" (for Digital Twins) and a suffix of "Info".
The elements in the object model are indexed by their identifiers, which have type `Dtmi`.  The following snippet displays the identifiers of all elements in the object model:

```C# Snippet:DtdlParserTutorial05Async_DisplayElements
Console.WriteLine($"{objectModel.Count} elements in model:");
foreach (KeyValuePair<Dtmi, DTEntityInfo> modelElement in objectModel)
{
    Console.WriteLine(modelElement.Key);
}
```

For the JSON text above, this snippet displays:

```Console
4 elements in model:
dtmi:example:anInterface:_contents:__expectedDistance;1
dtmi:example:anInterface:_contents:__currentDistance;1
dtmi:example:anInterface;1
dtmi:dtdl:instance:Schema:double;2
```

Of these four identifiers, only dtmi:example:anInterface;1 is present in the DTDL source model.
The identifiers for the contents named "expectedDistance" and "currentDistance" are auto-generated by the `ModelParser` following rules that guarantee their uniqueness.
The identifier dtmi:dtdl:instance:Schema:double;2 represents an element in the DTDL language model for the schema 'double', as can be seen by using the `ModelParser.GetTermOrUri()` static method:

```C# Snippet:DtdlParserTutorial05Async_DisplayDoubleTerm
Console.WriteLine(ModelParser.GetTermOrUri(new Dtmi("dtmi:dtdl:instance:Schema:double;2")));
```

This snippet displays:

```Console
double
```

## Drill down on one element and its property values by subtype

An individual element can be looked up in the object model by its identifier:

```C# Snippet:DtdlParserTutorial05Async_GetInterfaceById
var anInterfaceId = new Dtmi("dtmi:example:anInterface;1");
var anInterface = (DTInterfaceInfo)objectModel[anInterfaceId];
```

The DTDL type of each element is expressed via the property `EntityKind` on the `DTEntityInfo` base class, which has type `enum DTEntityKind`.
This can be used to specialize accesses for particular subtypes of DTDL Entities.
For example, there are five subtypes of Content, so the following snippet will display the values of appropriate properties for each `DTContentInfo` subclass:

```C# Snippet:DtdlParserTutorial05Async_DisplayInterfaceContentPropertiesByKind
foreach (KeyValuePair<string, DTContentInfo> contentElement in anInterface.Contents)
{
    switch (contentElement.Value.EntityKind)
    {
        case DTEntityKind.Property:
            var propertyElement = (DTPropertyInfo)contentElement.Value;
            Console.WriteLine($"Property '{propertyElement.Name}'");
            Console.WriteLine($"  schema: {propertyElement.Schema.Id?.ToString() ?? "(none)"}");
            Console.WriteLine($"  writable: {(propertyElement.Writable ? "true" : "false")}");
            break;
        case DTEntityKind.Telemetry:
            var telemetryElement = (DTTelemetryInfo)contentElement.Value;
            Console.WriteLine($"Telemetry '{telemetryElement.Name}'");
            Console.WriteLine($"  schema: {telemetryElement.Schema.Id?.ToString() ?? "(none)"}");
            break;
        case DTEntityKind.Command:
            var commandElement = (DTCommandInfo)contentElement.Value;
            Console.WriteLine($"Command '{commandElement.Name}'");
            Console.WriteLine($"  request schema: {commandElement.Request.Schema.Id?.ToString() ?? "(none)"}");
            Console.WriteLine($"  response schema: {commandElement.Response.Schema.Id?.ToString() ?? "(none)"}");
            break;
        case DTEntityKind.Relationship:
            var relationshipElement = (DTRelationshipInfo)contentElement.Value;
            Console.WriteLine($"Relationship '{relationshipElement.Name}'");
            Console.WriteLine($"  target: {relationshipElement.Target?.ToString() ?? "(none)"}");
            Console.WriteLine($"  writable: {(relationshipElement.Writable ? "true" : "false")}");
            break;
        case DTEntityKind.Component:
            var componentElement = (DTComponentInfo)contentElement.Value;
            Console.WriteLine($"Component '{componentElement.Name}'");
            Console.WriteLine($"  schema: {componentElement.Schema.Id}");
            break;
    }
}
```

For the JSON text above, this snippet displays:

```Console
Property 'expectedDistance'
  schema: dtmi:dtdl:instance:Schema:double;2
  writable: false
Telemetry 'currentDistance'
  schema: dtmi:dtdl:instance:Schema:double;2
```

## Use reflection to inspect property values of elements in the object model

Properties can also be accessed via the `System.Reflection` framework.
The following snippet scans through all elements in the object model, finds all property values that are subclasses of `DTEntityInfo`, and displays the identifier of each referenced element:

```C# Snippet:DtdlParserTutorial05Async_DisplayObjectModelEntityProperties
foreach (KeyValuePair<Dtmi, DTEntityInfo> modelElement in objectModel)
{
    Console.WriteLine($"{modelElement.Key} refers to:");

    TypeInfo typeInfo = modelElement.Value.GetType().GetTypeInfo();
    foreach (MemberInfo memberInfo in typeInfo.DeclaredMembers)
    {
        if (memberInfo is PropertyInfo propertyInfo)
        {
            object propertyValue = propertyInfo.GetValue(modelElement.Value);
            if (propertyValue is DTEntityInfo refSingle)
            {
                Console.WriteLine($"  {refSingle.Id}");
            }
            else if (propertyValue is IList refList)
            {
                foreach (object refObj in refList)
                {
                    if (refObj is DTEntityInfo refElement)
                    {
                        Console.WriteLine($"  {refElement.Id}");
                    }
                }
            }
            else if (propertyValue is IDictionary refDict)
            {
                foreach (object refObj in refDict.Values)
                {
                    if (refObj is DTEntityInfo refElement)
                    {
                        Console.WriteLine($"  {refElement.Id}");
                    }
                }
            }
        }
    }
}
```

For the JSON text above, this snippet displays:

```Console
dtmi:example:anInterface:_contents:__expectedDistance;1 refers to:
  dtmi:dtdl:instance:Schema:double;2
dtmi:example:anInterface:_contents:__currentDistance;1 refers to:
  dtmi:dtdl:instance:Schema:double;2
dtmi:example:anInterface;1 refers to:
  dtmi:example:anInterface:_contents:__expectedDistance;1
  dtmi:example:anInterface:_contents:__currentDistance;1
  dtmi:example:anInterface:_contents:__expectedDistance;1
  dtmi:example:anInterface:_contents:__currentDistance;1
dtmi:dtdl:instance:Schema:double;2 refers to:
```
