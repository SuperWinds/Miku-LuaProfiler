
# FAQs

## How to use xLua distribution package?

xLua is currently released as a zip package and can be extracted to the project directory.

## Can xLua be placed in another directory?

Yes, but the generated code directory needs to be configured (by default, it is in the Assets\XLua\Gen directory). For details, see the GenPath configuration in XLua Configuration.doc.

It's important to note that when changing directories, the generated code and xLua core code must be in the same assembly. If you want to use the hotfix function, the xLua core code must be in the Assembly-CSharp assembly.

## Does Lua source code only use the txt extension?

It can use any extension.

If you want to add TextAsset to an installation package (for example, to the Resources directory), Unity does not recognize the Lua extension. This is a limitation imposed by Unity.

If you do not add it to the installation package, there is no limit to the extension. For example, you can download it to a directory (this is also feasible in hotfix mode), and then read this directory with CustomLoader or by setting package.path.

The Lua source code (including examples) of xLua uses the txt extension because xLua itself is a library and doesn't provide download functionality. It's inconvenient to download code from elsewhere at runtime. Using TextAsset is a simpler solution.

## The editor (or non-il2cpp for Android) runs normally, but when iOS calls a function, "attempt to call a nil value" is reported.

By default, il2cpp will strip code, such as engine code, C# system APIs, and third-party dlls. Simply put, functions in these areas will not be compiled into your final release package if your C# code does not access them.

Solution: Add a reference (for example, configuring it to LuaCallCSharp, or adding access to that function in your C# code), or use the link.xml configuration (When ReflectionUse is configured, xLua will automatically configure it for you in link.xml) to instruct il2cpp not to strip certain types of code.

## Where can I find Plugins source code and how can I use it?

Plugins source code is located in the xLua_Project_Root/build directory.

The source code compilation depends on CMake. After installing CMake, execute make_xxxx_yyyy.zz. Here, xxxx represents the platform, such as iOS or Android; yyyy is the virtual machine to be integrated, including Lua 5.3 and LuaJIT. The file extension zz indicates the script type: bat for Windows and sh for other platforms.

Windows compilation requires Visual Studio 2015.

Android compilation is done on Linux, depends on NDK, and requires setting the ANDROID_NDK environment variable in the script to the NDK installation directory.

iOS and OS X compilations must be performed on a Mac.

## How do I solve the "xlua.access, no field __Hitfix0_Update" error?

Follow the [Hotfix Operation Guide](hotfix.md).

## How do I solve the "please install the Tools" error?

Do not install Tools in the same level directory as Assets. You can find Tools in the installation package or in the master directory.

## How do I solve the "This delegate/interface must add to CSharpCallLua: XXX" error?

In the editor, xLua can run even without generating code. This message appears either because CSharpCallLua was not added to the type, or because the code was generated before adding, but no generation was executed again.

Solution: After confirming that CSharpCallLua has been added to XXX (type name), clear the code and run it again.

If there is no problem with the editor, but the error is reported on the mobile device, it means that you did not generate the code (execute “XLua/Generate Code”) before release.

## What do I do if executing "XLua/Hotfix Inject In Editor" menu on Unity 5.5 or later versions produces the following prompt: "WARNING: The runtime version supported by this application is unavailable."

This is because the injection tool was compiled with .NET 3.5. The Unity 5.5 warning indicates that MonoBleedingEdge's mono environment does not support .NET 3.5. However, due to backward compatibility, no real problems related to this warning have been found so far.

You may find that defining INJECT_WITHOUT_TOOL in nested mode will not produce this warning. However, this mode is used for debugging and is not recommended as it may cause library conflicts.

## How do I trigger an event in hotfix?

First, enable private member access using xlua.private_accessible.

Then, call delegates using the "&event name" field of the object, for example self\['&MyEvent'\](), where MyEvent is the event name.

## How do I patch Unity Coroutine's implementation function?

Refer to the corresponding section of the [Hotfix Operation Guide](hotfix.md).

## Is NGUI (or UGUI/DOTween, etc...) supported?

Yes. The key feature of xLua is that anything you can write with C# can be replaced with Lua, and the plugins available in C# will remain available.

## If debugging is needed, how do I deal with the filepath parameter of CustomLoader?

When Lua calls require 'a.b', CustomLoader will be invoked and the string "a.b" will be passed. You need to interpret this string, load the Lua file (from file/memory/network, etc.), and return two things. The first is a path that the debugger can understand, such as a/b.lua, which is returned by setting the filepath parameter of the ref type. The second is the bytes[] of the source code in UTF8 format, which is returned as the function's return value.

## What is generated code?

XLua supports a Lua-C# interaction technique that involves generating adaptation code between the two for interaction. It offers better performance and is therefore recommended.

Another interaction technique is reflection, which has a smaller impact on the installation package and can be used in scenarios with lower performance requirements and installation package size constraints.

## How do I solve errors with the code generated before and after changing the interface?

Clear the generated code (execute the "Clear Generated Code" menu, which may disappear after a restart. If so, manually delete the entire generated code directory), and then regenerate the code once compilation is complete.

## When should the code be generated?

During the development period, generating code is not recommended to avoid compilation failures due to inconsistency and the waiting time for compiling the generated code.

The generated code must be executed before building the version for mobile devices. Automatic execution is recommended.

To optimize performance, the generated code must be executed before the performance test because there are significant differences between the generated code and the code not generated.

## Do all C# APIs in CS namespaces occupy high memory?

Due to LazyLoad, their existence is merely a virtual concept. For example, for UnityEngine.GameObject, its methods and properties are loaded only when accessing the first CS.UnityEngine.GameObject or transferring the first instance to Lua.

## In what scenarios are LuaCallSharp and CSharpCallLua used?

It depends on the caller and callee. For instance, if you want to call C#'s GameObject, find a function in Lua, or call GameObject's instance methods or properties, the GameObject type needs to be added to LuaCallSharp. If you want to add a Lua function to a UI callback (where C# is the caller and the Lua function is the callee), the delegate declared by the callback needs to be added to CSharpCallLua.

Sometimes it can be confusing, like when calling List<int>, for example. Find(Predicate<int> match) and List<int> will of course be added to LuaCallSharp. However, Predicate<int> needs to be added to CSharpCallLua, because the caller of match is C#, and a Lua function is being called.

A more straightforward approach: When you see "This delegate/interface must add to CSharpCallLua: XXX", simply add XXX to CSharpCallLua.

## Will gc alloc appear in value type transfer?

If you are using a delegate to call a Lua function, if the LuaTable and LuaFunction you use have no gc interface, or if there is an array, the following value types will not incur gc alloc:

1. All basic value types (all integers, all floating-point numbers, decimals)

2. All enumerated types

3. Structs containing only value types, which can nest other structs.

For 2 and 3, please add those types to GCOptimize.

## Is reflection available on iOS?

There are two restrictions on iOS: 1. no JIT; 2. code stripping;

When C# calls Lua via delegates or interfaces, using reflection emit instead of the generated code relies on JIT, so this is only available in editor mode.

If Lua calls C#, it will mainly be affected by code stripping. In this case, you can configure ReflectionUse (but not LuaCallSharp), and execute "Generate Code". No package code except for link.xml will be generated for the type this time. Set this type to 'not to be stripped'.

In short, only CSharpCallLua is necessary (a little code of this type is generated), and reflection can be used in LuaCallSharp generation.

## Is calling generic methods supported?

This is partially supported. See [Example 9 for the degree of support.](../Examples/09_GenericMethod/)

There are other ways to call generic methods. If it is a static method, you can write a wrapper to instantiate the generic method.

If it is a member method, xLua supports extension methods. You can add an extension method to instantiate a generic method. This extension method functions just like a regular member method.

```csharp
// C#
public static Button GetButton(this GameObject go)
{
    return go.GetComponent<Button>();
}
```

```lua
-- lua
local go = CS.UnityEngine.GameObject.Find("button")
go:GetButton().onClick:AddListener(function()
    print('onClick')
end)
```

## Can Lua call C# overloaded functions?

Yes, but this is not as well supported as in C#. For example, in overloaded methods void Foo(int a) and void Foo(short a), because int and short both correspond to the number in Lua, it is impossible to use the parameters to determine which overloaded method is being called. In this situation, you can use an extension method to give one of them an alias.

Yes, but this is not as well supported as in C#. For example, in the overloaded methods `void Foo(int a)` and `void Foo(short a)`, because both `int` and `short` correspond to the number type in Lua, it is impossible to use the parameters to determine which overloaded method is being called. In this situation, you can use an extension method to create an alias for one of them.

## What should I do if "A method/property/field is not defined" is reported when the code is generated during packaging, but it runs normally in the editor?

This often occurs because the method/property/field is conditionally compiled and only available in `UNITY_EDITOR`. This can be resolved by adding the method/property/field to the blacklist and then re-executing code generation after the compilation is complete.

## Why is the `this[string field]` or `this[object field]` operator overload inaccessible in Lua? (For example, in `Dictionary<string, xxx>`, I cannot retrieve values in `Dictionary<object, xxx>` using `dic['abc']` or `dic.abc` in Lua)

Because: 1. This feature will make methods, properties, and fields defined in the base type inaccessible. (For example, `Animation` cannot access the `GetComponent` method.); 2. Data cannot be retrieved if the key is the name of a method, property, or field of the current type. For instance, in the `Dictionary` type, `dic['TryGetValue']` returns a function that points to the `TryGetValue` method of the Dictionary.

If your version is newer than 2.1.11, you can use `get_Item` to get the value and `set_Item` to set the value. Note that only the `this[string field]` or `this[object field]` operator has these two alternative APIs. Keys of other types do not.

```lua
dic:set_Item('a', 1)
dic:set_Item('b', 2)
print(dic:get_Item('a'))
print(dic:get_Item('b'))
```

If your version is 2.1.11 or earlier, it is recommended to directly use the equivalent method of the operator, such as `TryGetValue` of the Dictionary. If this method is not available, you can wrap a method through an extension method in C# and then use it.

## Why are some Unity objects null in C# but not nil in Lua, for example, a GameObject that has been destroyed?

In fact, the C# object is not null, but `UnityEngine.Object` overloads the `==` operator. When an object is destroyed, or in the case of failed initialization, `obj == null` returns true, but the C# object is not actually null. You can verify this using `System.Object.ReferenceEquals(null, obj)`.

In this case, you can write an extension method for `UnityEngine.Object`.

```csharp
[LuaCallCSharp]
[ReflectionUse]
public static class UnityEngineObjectExtension
{
    public static bool IsNull(this UnityEngine.Object o) // or a name like IsDestroyed, etc.
    {
        return o == null;
    }
}
```

Then, in Lua, use `IsNull` for all `UnityEngine.Object` instances.

```lua
print(go:GetComponent('Animator'):IsNull())
```

## How do I construct a generic instance?

If the types involved are all in the `mscorlib` and `Assembly-CSharp` assemblies, constructing a generic instance is the same as for ordinary types. Both use `CS.namespace.typename()`. The typename expression for a generic instance contains the identifier illegal symbol, so the last part should be replaced with `["typename"]`. Use `List<string>` as an example.

```lua
local lst = CS.System.Collections.Generic["List`1[System.String]"]()
```

If the typename of a generic instance is undefined, you can print `typeof(undefined type).ToString()` in C#.

If the types involved are not in the `mscorlib` and `Assembly-CSharp` assemblies, you can use C# reflection:

```lua
local dic = CS.System.Activator.CreateInstance(CS.System.Type.GetType('System.Collections.Generic.Dictionary`2[[System.String, mscorlib],[UnityEngine.Vector3, UnityEngine]],mscorlib'))
dic:Add('a', CS.UnityEngine.Vector3(1, 2, 3))
print(dic:TryGetValue('a'))
```

If your xLua version is greater than v2.1.12, you can:

```lua
-- local List_String = CS.System.Collections.Generic['List<>'](CS.System.String) -- another way
local List_String = CS.System.Collections.Generic.List(CS.System.String)
local lst = List_String()

local Dictionary_String_Vector3 = CS.System.Collections.Generic.Dictionary(CS.System.String, CS.UnityEngine.Vector3)
local dic = Dictionary_String_Vector3()
dic:Add('a', CS.UnityEngine.Vector3(1, 2, 3))
print(dic:TryGetValue('a'))
```

## Why is the "try to dispose a LuaEnv with C# callback!" error reported when LuaEnv.Dispose is called?

This occurs because C# still has a delegate pointing to a function in the Lua virtual machine. This check prevents calling these invalid delegates after the virtual machine is released (they are invalid because the Lua function they reference has been released), which could cause exceptions or even crashes.

How do I resolve this? Release these delegates, i.e., the delegates that will not be referenced in C#:

If you retrieve and save the object member through `LuaTable.Get` in C#, then assign `null` to the member.

If you register a Lua function in Lua to some event callbacks, then deregister these callbacks.

If you perform an injection to C# via `xlua.hotfix(class, method, func)`, then remove it via `xlua.hotfix(type, method, nil)`.

Note that this should be done before `Dispose`.

## Crashes occur when calling LuaEnv.Dispose.

It is very likely that this `Dispose` operation is executed by Lua, which is equivalent to releasing the Lua virtual machine during its execution. Therefore, this can only be executed by C#.

## When the C# parameter (or field) type is `object`, the integer is transferred as the `long` type by default. How do I specify other types like `int`, for example?

See [Example 11](../Examples/11_RawObject/RawObjectTest.cs)

## How do I execute the original C# logic before executing hotfix?

With `util.hotfix_ex`, you can call the original C# logic.

```lua
local util = require 'xlua.util'
util.hotfix_ex(CS.HotfixTest, 'Add', function(self, a, b)
   local org_sum = self:Add(a, b)
   print('org_sum', org_sum)
   return a + b
end)
```

## How do I assign a C# function to a delegate field?

In versions 2.1.8 or earlier, you can simply use the C# function as a Lua function. The performance is lower because the delegate calls Lua through the bridge adaptation code first, and then Lua calls C#.

In version 2.1.9, the `createdelegate` function was added in `xlua.util`.

For example, consider the following C# code:

```csharp
public class TestClass
{
    public void Foo(int a)
    { 
    }
	
    public static void SFoo(int a)
    {
    }
}
public delegate void TestDelegate(int a);
```

You can use the `Foo` function to create a `TestDelegate` instance.

```lua
local util = require 'xlua.util'

local d1 = util.createdelegate(CS.TestDelegate, obj, CS.TestClass, 'Foo', {typeof(CS.System.Int32)}) -- Since Foo is an instance method, the second parameter must pass a TestClass instance
local d2 = util.createdelegate(CS.TestDelegate, nil, CS.TestClass, 'SFoo', {typeof(CS.System.Int32)})

obj_has_TestDelegate.field = d1 + d2 -- When calling field, Foo and SFoo will be triggered, bypassing Lua adaptation
```

## Why is Lua sometimes interrupted without an error message?

Generally, this occurs in two situations:

1. You used a coroutine instead of standard Lua to run code that encountered an error. Coroutine errors are expressed via the returned resume value. Please refer to the relevant Lua official documentation for more details. If you want to directly throw an exception for a coroutine error, you can add an `assert` to your resume call.

Change the following code:

```lua
coroutine.resume(co, ...)
```

to:

```lua
assert(coroutine.resume(co, ...))
```

2. `print` doesn't work after an upper layer catch.

For example, in some SDKs, the exception disappears after a try-catch during a callback.

## How do I deal with overload ambiguity?

For example, due to ignoring the `out` parameter, one of the overloads of `Physics.Raycast` cannot be called. (For instance, `short` and `int` cannot be distinguished.)

First, it's rare for the `out` parameter to cause overload ambiguity. As of September 22, 2017, we have only received feedback about `Physics.Raycast`. We recommend solving this by self-packaging (this is also applicable to `short` and `int`): Directly package static functions with different names. For a member method, package it with an extension method.

In the hotfix scenario, if we do not package it in advance, how do I call the specified overload?

Use `xlua.tofunction` and reflection. `xlua.tofunction` takes a `MethodBase` object and returns a Lua function. For example, consider the following C# code:

```csharp
class TestOverload
{
    public int Add(int a, int b)
    {
        Debug.Log("int version");
        return a + b;
    }

    public short Add(short a, short b)
    {
        Debug.Log("short version");
        return (short)(a + b);
    }
}
```

We can call the specified overload in this way:

```lua
local m1 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int16), typeof(CS.System.Int16)})
local m2 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int32), typeof(CS.System.Int32)})
local f1 = xlua.tofunction(m1) -- Remember, for the same MethodBase, only call tofunction once, then reuse it
local f2 = xlua.tofunction(m2)

local obj = CS.TestOverload()

f1(obj, 1, 2) -- Calls the short version, since it's a member method, the object must be passed, static methods do not require it
f2(obj, 1, 2) -- Calls the int version
```

Note: Since `xlua.tofunction` is not easy to use and reflection can also be utilized, it is recommended to use this only as a temporary solution. Instead, try to solve the issue with a packaging method.
local f1 = xlua.tofunction(m1) -- Remember, for the same MethodBase, only use tofunction once, then reuse it
local f2 = xlua.tofunction(m2)

local obj = CS.TestOverload()

f1(obj, 1, 2) -- Calls the short version, it's a member method, so the object must be passed, static methods do not require this
f2(obj, 1, 2) -- Calls the int version
```

Note: Since xlua.tofunction is not user-friendly and reflection can also be used, this approach is recommended only as a temporary solution. Instead, consider using a wrapper method.

## Is the extension method for interfaces supported?

Due to the amount of generated code, calling with obj:ExtensionMethod() is not supported. You can only call CS.ExtensionClass.ExtensionMethod(obj) through the static method.

