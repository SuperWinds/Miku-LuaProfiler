
## GC Issues with Complex Value Types

The default passing mechanism for xLua complex value types (structs) is by reference, which requires boxing the value type before passing it to Lua. After Lua has finished using it, the reference is released. Since each boxing of a value type creates a new object, releasing the reference on the Lua side after use results in a GC operation. To address this, xLua has implemented a GC optimization strategy for structs. With a simple configuration, you can ensure structs that meet certain criteria are passed to Lua without incurring GC.

## What Conditions Must a Struct Meet?

1. A struct may nest other structs, but both the struct and its nested structs must only contain the following basic types: byte, sbyte, short, ushort, int, uint, long, ulong, float, double. For example, most value types defined by UnityEngine, such as the Vector series, Quaternion, Color, etc., meet these criteria, as do some user-defined structs.
2. The struct must be configured with the GCOptimize attribute (for several commonly used structs in UnityEngine, such as the Vector series, Quaternion, Color, etc., this attribute is already configured). This attribute can be set via a configuration file or a C# Attribute.
3. Any usage of this struct must be included in the list for code generation.

## How to Configure?

Refer to the GCOptimize section in `XLua's configuration.md`.
