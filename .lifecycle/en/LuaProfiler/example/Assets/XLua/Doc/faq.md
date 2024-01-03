
It is not recommended to generate code during development, as this can avoid many compilation failures due to inconsistencies, as well as the wait time for code generation.

Code generation must be executed before building the mobile version, and it is recommended to automate this process.

Performance tuning and testing must be performed after code generation, as there is a significant performance difference between generating and not generating code.

## Does having all C# APIs in the CS namespace consume a lot of memory?

Due to lazy loading, this "having" is just a virtual concept. For example, for UnityEngine.GameObject, the methods, properties, etc., of this type are only loaded when CS.UnityEngine.GameObject is accessed for the first time or when the first instance is passed to Lua.

## In what scenarios are LuaCallSharp and CSharpCallLua used?

Consider the caller and callee. For instance, if you want to call C#'s GameObject.Find function in Lua, or invoke instance methods, properties, etc., of a GameObject, you should add LuaCallSharp to the GameObject class. Conversely, if you want to attach a Lua function to a UI callback, where the caller is C# and the callee is a Lua function, the delegate declared for the callback should be added to CSharpCallLua.

Sometimes it can be confusing, such as when calling List<int>.Find(Predicate<int> match). List<int> should have LuaCallSharp, but Predicate<int> should have CSharpCallLua because the caller of match is in C#, and the callee is a Lua function.

A simpler approach is to add XXX to CSharpCallLua whenever you see the message "This delegate/interface must add to CSharpCallLua: XXX".

## Will there be gc alloc when passing value types?

If you use a delegate to call a Lua function, or use the GC-free interfaces of LuaTable and LuaFunction, or arrays, the following value types will not incur GC allocation:

1. All basic value types (all integers, all floating-point numbers, decimal).
2. All enumeration types.
3. Structs that only contain value types, which can be nested with other structs that only contain value types.

Types 2 and 3 need to be added to GCOptimize.

## Is reflection available on iOS?

iOS has two limitations: 1. No JIT; 2. Code stripping.

For C# calling Lua through a delegate or interface, if no code is generated, it uses reflection emit, which depends on JIT, so this is currently only available in the editor.

For Lua calling C#, it is mainly affected by code stripping. In this case, you can configure ReflectionUse (do not configure LuaCallSharp) and execute "Generate Code". This will not generate wrapper code for the class but will generate link.xml to configure the class to avoid stripping.

In short, except for CSharpCallLua, which is essential (and usually doesn't generate much code), LuaCallSharp generation can be replaced with reflection.

## Does it support calling generic methods?

1. Support for generic constraints to a certain base class is demonstrated in the example: (../Examples/09_GenericMethod/).
2. If there are no generic constraints, it is recommended to encapsulate the method as non-generic. For static methods, you can write an encapsulation to instantiate the generic method. For member methods, xLua supports extension methods, allowing you to add an extension method to instantiate the generic method. This extension method is used just like a regular member method.

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

3. If the xlua version is greater than 2.1.12, new support for reflection calling of generic methods is added (with certain restrictions, see the instructions below). For example, for this C# type:

```csharp
public class GetGenericMethodTest
{
    int a = 100;
    public int Foo<T1, T2>(T1 p1, T2 p2)
    {
        Debug.Log(typeof(T1));
        Debug.Log(typeof(T2));
        Debug.Log(p1);
        Debug.Log(p2);
        return a;
    }

    public static void Bar<T1, T2>(T1 p1, T2 p2)
    {
        Debug.Log(typeof(T1));
        Debug.Log(typeof(T2));
        Debug.Log(p1);
        Debug.Log(p2);
    }
}
```

In Lua, call it like this:

```lua
local foo_generic = xlua.get_generic_method(CS.GetGenericMethodTest, 'Foo')
local bar_generic = xlua.get_generic_method(CS.GetGenericMethodTest, 'Bar')

local foo = foo_generic(CS.System.Int32, CS.System.Double)
local bar = bar_generic(CS.System.Double, CS.UnityEngine.GameObject)

-- call instance method
local o = CS.GetGenericMethodTest()
local ret = foo(o, 1, 2)
print(ret)

-- call static method
bar(2, nil)
```

Usage restrictions are as follows:

* It works under mono.
* Under il2cpp, if the generic parameter is a reference type, it can be used.
* Under il2cpp, if the generic parameter is a value type, it can be used if the same generic parameter has been called in C# (usually the case in hotfix scenarios, so it is generally available under the hotfix feature).

## Does Lua support calling C# overloaded functions?

Supported, but not as fully as C# does. For example, the overloaded methods void Foo(int a) and void Foo(short a) cannot be distinguished because both int and short correspond to Lua's number. In such cases, you can use extension methods to give one of the methods an alias.

## It runs normally in the editor, but when packaging, generating code reports "No definition for certain method/property/field". What should I do?

This is often because the method/property/field is within conditional compilation and is only valid under UNITY_EDITOR. This can be resolved by adding this method/property/field to the blacklist. After adding it, wait for the compilation to complete and re-execute code generation.

## Why can't this[string field] or this[object field] operator overloads be accessed in Lua? (For example, Dictionary\<string, xxx\>, Dictionary\<object, xxx\> cannot retrieve values through dic['abc'] or dic.abc in Lua)

Because: 1. This feature would prevent access to methods, properties, fields, etc., defined by the base class (e.g., Animation cannot access GetComponent method); 2. Data with a key that is the name of a method, property, or field of the current class cannot be retrieved, such as with the Dictionary type, dic['TryGetValue'] returns a function pointing to the Dictionary's TryGetValue method.

If your version is greater than 2.1.11, you can use get_Item to get values and set_Item to set values. Note that only this[string field] or this[object field] have these alternative APIs; other types of keys do not.

```lua
dic:set_Item('a', 1)
dic:set_Item('b', 2)
print(dic:get_Item('a'))
print(dic:get_Item('b'))
```

If your version is less than or equal to 2.1.11, it is recommended to use the equivalent method for this operator, such as Dictionary's TryGetValue. If this method is not provided, you can encapsulate one using an Extension method in C#.

## Why are some Unity objects null in C#, but not nil in Lua? For example, a GameObject that has been Destroyed

Actually, the C# object is not null; it is the == operator overloaded by UnityEngine.Object. When an object is Destroyed, not initialized, etc., obj == null returns true, but the C# object itself is not null. You can verify this with System.Object.ReferenceEquals(null, obj).

For this situation, you can write an extension method for UnityEngine.Object:

```csharp
[LuaCallCSharp]
[ReflectionUse]
public static class UnityEngineObjectExtention
{
    public static bool IsNull(this UnityEngine.Object o) // or named IsDestroyed, etc.
    {
        return o == null;
    }
}
```

Then in Lua, you use IsNull to judge all UnityEngine.Object instances.

```lua
print(go:GetComponent('Animator'):IsNull())
```

## How to construct a generic instance

If the types involved are all in the mscorlib and Assembly-CSharp assemblies, the construction of generic instances is the same as for ordinary types, both are CS.namespace.typename(). The expression of typename for generic instances contains identifier-illegal symbols, and the last part must be replaced with ["typename"], for example, List<string>:

```lua
local lst = CS.System.Collections.Generic["List`1[System.String]"]()
```

If the typename of a generic instance is uncertain, you can print typeof(uncertain type).ToString() in C#.

If types from outside mscorlib or Assembly-CSharp are involved, you can use C# reflection:

```lua
local dic = CS.System.Activator.CreateInstance(CS.System.Type.GetType('System.Collections.Generic.Dictionary`2[[System.String, mscorlib],[UnityEngine.Vector3, UnityEngine]],mscorlib'))
dic:Add('a', CS.UnityEngine.Vector3(1, 2, 3))
print(dic:TryGetValue('a'))
```

If your xLua version is greater than v2.1.12, there will be more elegant expressions:

```lua
-- local List_String = CS.System.Collections.Generic['List<>'](CS.System.String) -- another way
local List_String = CS.System.Collections.Generic.List(CS.System.String)
local lst = List_String()

local Dictionary_String_Vector3 = CS.System.Collections.Generic.Dictionary(CS.System.String, CS.UnityEngine.Vector3)
local dic = Dictionary_String_Vector3()
dic:Add('a', CS.UnityEngine.Vector3(1, 2, 3))
print(dic:TryGetValue('a'))
```
local List_String = CS.System.Collections.Generic.List(CS.System.String)
local lst = List_String()

local Dictionary_String_Vector3 = CS.System.Collections.Generic.Dictionary(CS.System.String, CS.UnityEngine.Vector3)
local dic = Dictionary_String_Vector3()
dic:Add('a', CS.UnityEngine.Vector3(1, 2, 3))
print(dic:TryGetValue('a'))
```

## Why do I get the error "try to dispose a LuaEnv with C# callback!" when calling LuaEnv.Dispose?

This error occurs when you try to dispose a LuaEnv instance that still has C# callbacks registered. To fix this, you need to release all C# callbacks before disposing the LuaEnv instance. You can release the callbacks by setting the references to null or unregistering them from the corresponding events.

## How can I execute the original C# logic first and then execute the patch?

You can use the `util.hotfix_ex` function to call the original C# logic before applying the patch. Here's an example:

```lua
local util = require 'xlua.util'
util.hotfix_ex(CS.HotfixTest, 'Add', function(self, a, b)
   local org_sum = self:Add(a, b)
   print('org_sum', org_sum)
   return a + b
end)
```

## How can I assign a C# function to a delegate field?

In versions prior to 2.1.9, you can treat the C# function as a Lua function and assign it to the delegate field. However, this approach has lower performance because it involves calling Lua functions through the Birdage adaptation code and then calling back to C#.

Starting from version 2.1.9, you can use the `util.createdelegate` function to create a delegate from a C# function. Here's an example:

```lua
local util = require 'xlua.util'

local d = util.createdelegate(CS.TestDelegate, obj, CS.TestClass, 'Foo', {typeof(CS.System.Int32)})
obj_has_TestDelegate.field = d
```

## Why do I sometimes get a Lua error without an error message?

There are two common reasons for this issue:

1. If you are using coroutines, Lua errors in coroutines are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the upper layer catches the error and does not print it, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I resolve overload ambiguity?

If you encounter overload ambiguity issues, you can use the `xlua.tofunction` function to specify the exact overload to call. Here's an example:

```lua
local m1 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int16), typeof(CS.System.Int16)})
local m2 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int32), typeof(CS.System.Int32)})
local f1 = xlua.tofunction(m1)
local f2 = xlua.tofunction(m2)

local obj = CS.TestOverload()

f1(obj, 1, 2) -- Calls the short version
f2(obj, 1, 2) -- Calls the int version
```

## How can I integrate xLua's Wrap generation into my project's automated build process?

You can use the command line to call Unity's custom class methods for building the project. You can refer to [Example 13](../Examples/13_BuildFromCLI/) for more details.

## How can I handle the error "DelegatesGensBridge.cs references a non-existent class" when building the mobile version?

This error occurs when the Hotfix list references types that are not available in the mobile version, such as types that are only used by the editor. To fix this, you need to exclude the types that are not available in the mobile version from the Hotfix list.

## How can I load bytecode?

You can load bytecode directly after compiling it with luac. However, please note that Lua bytecode is platform-specific. You need to use the appropriate bytecode for the target platform.

## How can I release C# objects held by Lua?

To release C# objects held by Lua, you need to ensure that:

1. All Lua references to the C# objects are released.
2. A garbage collection cycle is completed in Lua.

By following these steps, the C# objects will be eligible for garbage collection. However, please note that Lua's garbage collector works differently from C#'s garbage collector. Lua's garbage collector is not automatically triggered in the background but is instead invoked when memory is allocated. Therefore, you may need to manually trigger a garbage collection cycle in Lua using `collectgarbage('collect')` or `LuaEnv.FullGc()`.

## How can I prevent crashes in multi-threaded environments?

To prevent crashes in multi-threaded environments, you need to define the `THREAD_SAFE` macro in the Player Settings. This ensures that xLua is thread-safe and can be used in multi-threaded scenarios.

## Why do I sometimes get Lua errors without error messages?

There are two common reasons for this issue:

1. If your error code is running in a coroutine, Lua errors are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the error is caught by an upper layer and not printed, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I execute the original C# logic first and then execute the patch?

You can use the `util.hotfix_ex` function to execute the original C# logic before applying the patch. Here's an example:

```lua
local util = require 'xlua.util'
util.hotfix_ex(CS.HotfixTest, 'Add', function(self, a, b)
   local org_sum = self:Add(a, b)
   print('org_sum', org_sum)
   return a + b
end)
```

## How can I assign a C# function to a delegate field?

In versions prior to 2.1.9, you can treat the C# function as a Lua function and assign it to the delegate field. However, this approach has lower performance because it involves calling Lua functions through the Birdage adaptation code and then calling back to C#.

Starting from version 2.1.9, you can use the `util.createdelegate` function to create a delegate from a C# function. Here's an example:

```lua
local util = require 'xlua.util'

local d = util.createdelegate(CS.TestDelegate, obj, CS.TestClass, 'Foo', {typeof(CS.System.Int32)})
obj_has_TestDelegate.field = d
```

## Why do I sometimes get a Lua error without an error message?

There are two common reasons for this issue:

1. If you are using coroutines, Lua errors in coroutines are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the upper layer catches the error and does not print it, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I resolve overload ambiguity?

If you encounter overload ambiguity issues, you can use the `xlua.tofunction` function to specify the exact overload to call. Here's an example:

```lua
local m1 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int16), typeof(CS.System.Int16)})
local m2 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int32), typeof(CS.System.Int32)})
local f1 = xlua.tofunction(m1)
local f2 = xlua.tofunction(m2)

local obj = CS.TestOverload()

f1(obj, 1, 2) -- Calls the short version
f2(obj, 1, 2) -- Calls the int version
```

## How can I integrate xLua's Wrap generation into my project's automated build process?

You can use the command line to call Unity's custom class methods for building the project. You can refer to [Example 13](../Examples/13_BuildFromCLI/) for more details.

## How can I handle the error "DelegatesGensBridge.cs references a non-existent class" when building the mobile version?

This error occurs when the Hotfix list references types that are not available in the mobile version, such as types that are only used by the editor. To fix this, you need to exclude the types that are not available in the mobile version from the Hotfix list.

## How can I load bytecode?

You can load bytecode directly after compiling it with luac. However, please note that Lua bytecode is platform-specific. You need to use the appropriate bytecode for the target platform.

## How can I release C# objects held by Lua?

To release C# objects held by Lua, you need to ensure that:

1. All Lua references to the C# objects are released.
2. A garbage collection cycle is completed in Lua.

By following these steps, the C# objects will be eligible for garbage collection. However, please note that Lua's garbage collector works differently from C#'s garbage collector. Lua's garbage collector is not automatically triggered in the background but is instead invoked when memory is allocated. Therefore, you may need to manually trigger a garbage collection cycle in Lua using `collectgarbage('collect')` or `LuaEnv.FullGc()`.

## How can I prevent crashes in multi-threaded environments?

To prevent crashes in multi-threaded environments, you need to define the `THREAD_SAFE` macro in the Player Settings. This ensures that xLua is thread-safe and can be used in multi-threaded scenarios.

## Why do I sometimes get Lua errors without error messages?

There are two common reasons for this issue:

1. If your error code is running in a coroutine, Lua errors are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the error is caught by an upper layer and not printed, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I execute the original C# logic first and then execute the patch?

You can use the `util.hotfix_ex` function to execute the original C# logic before applying the patch. Here's an example:

```lua
local util = require 'xlua.util'
util.hotfix_ex(CS.HotfixTest, 'Add', function(self, a, b)
   local org_sum = self:Add(a, b)
   print('org_sum', org_sum)
   return a + b
end)
```

## How can I assign a C# function to a delegate field?

In versions prior to 2.1.9, you can treat the C# function as a Lua function and assign it to the delegate field. However, this approach has lower performance because it involves calling Lua functions through the Birdage adaptation code and then calling back to C#.

Starting from version 2.1.9, you can use the `util.createdelegate` function to create a delegate from a C# function. Here's an example:

```lua
local util = require 'xlua.util'

local d = util.createdelegate(CS.TestDelegate, obj, CS.TestClass, 'Foo', {typeof(CS.System.Int32)})
obj_has_TestDelegate.field = d
```

## Why do I sometimes get a Lua error without an error message?

There are two common reasons for this issue:

1. If you are using coroutines, Lua errors in coroutines are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the upper layer catches the error and does not print it, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I resolve overload ambiguity?

If you encounter overload ambiguity issues, you can use the `xlua.tofunction` function to specify the exact overload to call. Here's an example:

```lua
local m1 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int16), typeof(CS.System.Int16)})
local m2 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int32), typeof(CS.System.Int32)})
local f1 = xlua.tofunction(m1)
local f2 = xlua.tofunction(m2)

local obj = CS.TestOverload()

f1(obj, 1, 2) -- Calls the short version
f2(obj, 1, 2) -- Calls the int version
```

## How can I integrate xLua's Wrap generation into my project's automated build process?

You can use the command line to call Unity's custom class methods for building the project. You can refer to [Example 13](../Examples/13_BuildFromCLI/) for more details.

## How can I handle the error "DelegatesGensBridge.cs references a non-existent class" when building the mobile version?

This error occurs when the Hotfix list references types that are not available in the mobile version, such as types that are only used by the editor. To fix this, you need to exclude the types that are not available in the mobile version from the Hotfix list.

## How can I load bytecode?

You can load bytecode directly after compiling it with luac. However, please note that Lua bytecode is platform-specific. You need to use the appropriate bytecode for the target platform.

## How can I release C# objects held by Lua?

To release C# objects held by Lua, you need to ensure that:

1. All Lua references to the C# objects are released.
2. A garbage collection cycle is completed in Lua.

By following these steps, the C# objects will be eligible for garbage collection. However, please note that Lua's garbage collector works differently from C#'s garbage collector. Lua's garbage collector is not automatically triggered in the background but is instead invoked when memory is allocated. Therefore, you may need to manually trigger a garbage collection cycle in Lua using `collectgarbage('collect')` or `LuaEnv.FullGc()`.

## How can I prevent crashes in multi-threaded environments?

To prevent crashes in multi-threaded environments, you need to define the `THREAD_SAFE` macro in the Player Settings. This ensures that xLua is thread-safe and can be used in multi-threaded scenarios.

## Why do I sometimes get Lua errors without error messages?

There are two common reasons for this issue:

1. If your error code is running in a coroutine, Lua errors are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the error is caught by an upper layer and not printed, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I execute the original C# logic first and then execute the patch?

You can use the `util.hotfix_ex` function to execute the original C# logic before applying the patch. Here's an example:

```lua
local util = require 'xlua.util'
util.hotfix_ex(CS.HotfixTest, 'Add', function(self, a, b)
   local org_sum = self:Add(a, b)
   print('org_sum', org_sum)
   return a + b
end)
```

## How can I assign a C# function to a delegate field?

In versions prior to 2.1.9, you can treat the C# function as a Lua function and assign it to the delegate field. However, this approach has lower performance because it involves calling Lua functions through the Birdage adaptation code and then calling back to C#.

Starting from version 2.1.9, you can use the `util.createdelegate` function to create a delegate from a C# function. Here's an example:

```lua
local util = require 'xlua.util'

local d = util.createdelegate(CS.TestDelegate, obj, CS.TestClass, 'Foo', {typeof(CS.System.Int32)})
obj_has_TestDelegate.field = d
```

## Why do I sometimes get a Lua error without an error message?

There are two common reasons for this issue:

1. If you are using coroutines, Lua errors in coroutines are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the upper layer catches the error and does not print it, you won't see the error message. Make sure to print or log the error message when catching Lua errors.

## How can I resolve overload ambiguity?

If you encounter overload ambiguity issues, you can use the `xlua.tofunction` function to specify the exact overload to call. Here's an example:

```lua
local m1 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int16), typeof(CS.System.Int16)})
local m2 = typeof(CS.TestOverload):GetMethod('Add', {typeof(CS.System.Int32), typeof(CS.System.Int32)})
local f1 = xlua.tofunction(m1)
local f2 = xlua.tofunction(m2)

local obj = CS.TestOverload()

f1(obj, 1, 2) -- Calls the short version
f2(obj, 1, 2) -- Calls the int version
```

## How can I integrate xLua's Wrap generation into my project's automated build process?

You can use the command line to call Unity's custom class methods for building the project. You can refer to [Example 13](../Examples/13_BuildFromCLI/) for more details.

## How can I handle the error "DelegatesGensBridge.cs references a non-existent class" when building the mobile version?

This error occurs when the Hotfix list references types that are not available in the mobile version, such as types that are only used by the editor. To fix this, you need to exclude the types that are not available in the mobile version from the Hotfix list.

## How can I load bytecode?

You can load bytecode directly after compiling it with luac. However, please note that Lua bytecode is platform-specific. You need to use the appropriate bytecode for the target platform.

## How can I release C# objects held by Lua?

To release C# objects held by Lua, you need to ensure that:

1. All Lua references to the C# objects are released.
2. A garbage collection cycle is completed in Lua.

By following these steps, the C# objects will be eligible for garbage collection. However, please note that Lua's garbage collector works differently from C#'s garbage collector. Lua's garbage collector is not automatically triggered in the background but is instead invoked when memory is allocated. Therefore, you may need to manually trigger a garbage collection cycle in Lua using `collectgarbage('collect')` or `LuaEnv.FullGc()`.

## How can I prevent crashes in multi-threaded environments?

To prevent crashes in multi-threaded environments, you need to define the `THREAD_SAFE` macro in the Player Settings. This ensures that xLua is thread-safe and can be used in multi-threaded scenarios.

## Why do I sometimes get Lua errors without error messages?

There are two common reasons for this issue:

1. If your error code is running in a coroutine, Lua errors are indicated by the return value of the `coroutine.resume` function. To throw an exception when an error occurs in a coroutine, you can add an `assert` statement around the `coroutine.resume` call.

2. If the error is caught by an upper layer and not printed, you won't see the error message. Make sure to print or log the error message when catching Lua errors.
* 1. Adjust GcPause to make gc start more promptly. By default, 200 indicates gc will begin after memory usage doubles from the last collection. Coco2dx sets it to 100, meaning gc starts immediately after one cycle completes. You can also modify GcStepmul to accelerate the gc process. The standard setting of 200 suggests gc runs twice as fast as memory allocation, while coco2dx sets GcStepmul to 5000.
* 2. In scenarios with lower performance demands, such as switching scenes, you can invoke a full gc cycle (using LuaEnv.FullGc or by executing collectgarbage('collect') in Lua).

## How to resolve unexplained crashes in multi-threaded environments

Multi-threading usage necessitates defining the THREAD_SAFE macro in (Player Setting/Scripting Define Symbols).

Typical subtle multi-threaded contexts include asynchronous sockets in C# and object finalizers.