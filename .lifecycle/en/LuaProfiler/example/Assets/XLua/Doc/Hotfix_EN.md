
## Usage

1. Add the `HOTFIX_ENABLE` macro to enable this feature (navigate to File->Build Settings->Scripting Define Symbols in Unity3D). This macro should be set separately for the editor and each mobile platform! For automated builds, note that macros set in the API code are not valid and must be set in the editor.

(It is recommended to use `HOTFIX_ENABLE` only for mobile build versions or when developing a patch in the editor, not for regular development code.)

2. Execute the `XLua/Generate Code` menu option.

3. This step will be automatically performed during the injection and building of mobile packages. Manually execute the `XLua/Hotfix Inject In Editor` menu when developing a hotfix in the editor. Upon successful injection, "hotfix inject finish!" or "had injected!" will be printed.

## Constraints

Static constructors are not supported.

Currently, only code hotfixes for Assets are supported, not for the engine or the C# system library.

## API

`xlua.hotfix(class, [method_name], fix)`

* Description: Inject a Lua patch.
* `class`: C# class, represented in two ways: `CS.Namespace.Typename` or the string "Namespace.Typename". The string format is consistent with C#'s `Type.GetType`; if the nested type is non-public, you must use the string method "Namespace.TypeName+NestedTypeName".
* `method_name`: Optional method name.
* `fix`: If `method_name` is provided, `fix` will be a function; otherwise, it will be a table providing a set of functions. The table is organized with `method_name` as the key and the function as the value.

`base(csobj)`

* Description: Implement child type override functions by calling the parent type via `base`.
* `csobj`: The object.
* Return value: A new object, which can be used to call the method via `base`.

Example (in HotfixTest2.cs):

```lua
xlua.hotfix(CS.BaseTest, 'Foo', function(self, p)
    print('BaseTest', p)
    base(self):Foo(p)
end)
```

`util.hotfix_ex(class, method_name, fix)`

* Description: An enhanced version of `xlua.hotfix`, which allows the original function to be performed within the `fix` function. Its disadvantage is that the implementation of `fix` will be slightly slower.
* `method_name`: The method name.
* `fix`: The Lua function used to replace the C# method.

## Tag the type of hotfix

There are two methods, similar to other configurations:

1. Directly add the `Hotfix` attribute in the type.

2. Configure a list in a static field or property of a static type. Properties can be used for more complex configurations, such as Namespace-based whitelisting.

```csharp
public static class HotfixCfg
{
    [Hotfix]
    public static List<Type> by_field = new List<Type>()
    {
        typeof(HotFixSubClass),
        typeof(GenericClass<>),
    };

    [Hotfix]
    public static List<Type> by_property
    {
        get
        {
            return (from type in Assembly.Load("Assembly-CSharp").GetTypes()
                    where type.Namespace == "XXXX"
                    select type).ToList();
        }
    }
}
```

## Hotfix Flag

The `Hotfix` attribute can set flags to customize the generated code and instrumentation.

* Stateless and Stateful

This is a legacy setting. The Stateful method has been removed in newer versions because similar effects can be achieved with the `xlua.util.state` interface. For usage, see the sample code in HotfixTest2.cs.

Without Stateful, the default is Stateless, so there is no need to set this flag.

* ValueTypeBoxing

Adapts value types. The delegate will be cast to an object. Its advantage is less code; its disadvantage is that it generates boxing and garbage collection. Suitable for services sensitive to text segment size.

* IgnoreProperty

No property injection or adaptation code generation. Generally, most properties are simple and less likely to have errors. Injection is not recommended.

* IgnoreNotPublic

No injection or adaptation code generation for non-public methods. Only private methods (such as those in MonoBehaviour) called by reflection will be injected. Other non-public methods called only within their own type may not be injected, increasing the repair workload. All public methods referencing this function must be rewritten.

* Inline

No generation of adaptation delegates. The process code is directly injected into the function body.

* IntKey

Instead of generating static fields, all injection points are managed in an array.

Advantages: Minimal impact on the text segment.

Disadvantages: Less convenient than the default method; it requires using an ID to indicate which function the hotfix applies to. This ID is assigned during code injection. The function-ID mapping is saved in `Gen/Resources/hotfix_id_map.lua.txt`. Automatic timestamp backups are made to the same directory level as `hotfix_id_map.lua.txt`. After releasing the mobile version, save the file properly.

The format of this file is as follows (Note: This file is only used in IntKey mode. If IntKey mode is not specified for injection, the file will return only an empty table):

```lua
return {
    ["HotfixTest"] = {
        [".ctor"] = {
            5
        },
        ["Start"] = {
            6
        },
        ["Update"] = {
            7
        },
        ["FixedUpdate"] = {
            8
        },
        ["Add"] = {
            9,10
        },
        ["OnGUI"] = {
            11
        },
    },
}
```

To replace the `Update` function of `HotfixTest`, you must:

```lua
CS.XLua.HotfixDelegateBridge.Set(7, func)
```

If it is an overloaded function, a function name will correspond to multiple IDs, such as the `Add` function above.

Can it be automated? Yes. `xlua.util` provides the `auto_id_map` function. After executing it once, you can use the type and method name to indicate the function to be fixed.

```lua
(require 'xlua.util').auto_id_map()
xlua.hotfix(CS.HotfixTest, 'Update', function(self)
        self.tick = self.tick + 1
        if (self.tick % 50) == 0 then
            print('<<<<<<<<Update in lua, tick = ' .. self.tick)
        end
    end)
```

The prerequisite is that `hotfix_id_map.lua.txt` is in a directory that can be referenced by `require 'hotfix_id_map'`.

## Usage suggestions

* Add the `Hotfix` attribute to all types that are likely to be modified.
* Use reflection to find all delegate types involved in function parameters, fields, properties, and events, and add them to `CSharpCallLua`.
* Add `LuaCallCSharp` to business code, engine APIs, system APIs, and types that require high-performance access from Lua hotfixes.
* Engine APIs and system APIs may be stripped by the linker (those not referenced by C# will be stripped). If you anticipate adding API calls not present in C# code, add these APIs to either `LuaCallCSharp` or `ReflectionUse`.

## Patching

Xlua uses Lua functions to replace C#'s constructors, functions, properties, and events. Lua implementations are function-based. For example, a property corresponds to a getter and a setter function, and an event corresponds to an add and a remove function.

* Functions

`method_name` specifies the function name and supports overloading. Different overloads are directed to the same Lua function.

For example:

```csharp
// C# class to be fixed
[Hotfix]
public class HotfixCalc
{
    public int Add(int a, int b)
    {
        return a - b;
    }

    public Vector3 Add(Vector3 a, Vector3 b)
    {
        return a - b;
    }
}
```

```lua
xlua.hotfix(CS.HotfixCalc, 'Add', function(self, a, b)
    return a + b
end)
```

The difference between a static function and a member function is that a member function includes a `self` parameter. In Stateless mode, `self` is the C# object itself (equivalent to C#'s `this`).

Regular parameters correspond to Lua parameters. The `ref` parameter corresponds to a Lua parameter and a return value, and the `out` parameter corresponds to a return value in Lua.

Generic functions follow the same hotfix rules as regular functions.

* Constructors

The `method_name` for constructors is ".ctor".

Unlike regular functions, the constructor hotfix is not a replacement but calls Lua after executing the original logic.

* Properties

A property named "AProp" will have a getter with `method_name` "get_AProp" and a setter with `method_name` "set_AProp".

* Indexers

The setter corresponds to "set_Item", and the getter corresponds to "get_Item". The first parameter is `self`, followed by the key and value for the setter. The getter has only the key parameter, and the return value is the retrieved value.

* Other operators

C# operators have a set of internal representations. For example, the `+` operator function name is "op_Addition" (for other operators' internal representations, see the relevant documentation). Overriding this function will override the `+` operator in C#.

* Events

For an event named "AEvent", the `+=` operator is "add_AEvent", and the `-=` operator is "remove_AEvent". The first parameter for both functions is `self`, and the second parameter is the delegate following the operator.

Then, directly access the private delegate corresponding to the event via `xlua.private_accessible`. (For versions newer than 2.1.11, calling `xlua.private_accessible` is not necessary.) The event can be triggered directly through the object's "&event name" field, such as `self['&MyEvent']()`, where "MyEvent" is the event name.

* Destructors

The `method_name` is "Finalize" and it takes a `self` parameter.

Unlike regular functions, the destructor hotfix is not a replacement but continues the original logic after calling the Lua function.

* Generic types

The rules are the same as for non-generic types. Note that each instantiated generic type is a separate type. You can only patch each instantiated type. For example:

```csharp
public class GenericClass<T>
{
}
```

You can only patch `GenericClass<double>` and `GenericClass<int>`, not `GenericClass` itself.

Here is an example of patching `GenericClass<double>`:

```csharp
luaenv.DoString(@"
    xlua.hotfix(CS.GenericClass(CS.System.Double), {
        ['.ctor'] = function(obj, a)
            print('GenericClass<double>', obj, a)
        end;
        Func1 = function(obj)
            print('GenericClass<double>.Func1', obj)
        end;
        Func2 = function(obj)
            print('GenericClass<double>.Func2', obj)
            return 1314
        end
    })
");
```

* Unity coroutines

Using `util.cs_generator`, you can simulate an `IEnumerator` with a function. The `coroutine.yield` here is similar to C#'s `yield return`. For example, the following C# code is equivalent to the corresponding hotfix code.

```csharp
[XLua.Hotfix]
public class HotFixSubClass : MonoBehaviour {
    IEnumerator Start()
    {
        while (true)
        {
            yield return new WaitForSeconds(3);
            Debug.Log("Wait for 3 seconds");
        }
    }
}
```

```lua
luaenv.DoString(@"
    local util = require 'xlua.util'
    xlua.hotfix(CS.HotFixSubClass,{
        Start = function(self)
            return util.cs_generator(function()
                while true do
                    coroutine.yield(CS.UnityEngine.WaitForSeconds(3))
                    print('Wait for 3 seconds')
                end
            end)
        end;
    })
");
```

* Entire type

To replace an entire type, you don't need to call `xlua.hotfix` repeatedly; you can do it all at once. Provide a table with `method_name = function`.

```lua
xlua.hotfix(CS.StatefullTest, {
    ['.ctor'] = function(csobj)
        return util.state(csobj, {evt = {}, start = 0, prop = 0})
    end;
    set_AProp = function(self, v)
        print('set_AProp', v)
        self.prop = v
    end;
    get_AProp = function(self)
        return self.prop
    end;
    get_Item = function(self, k)
        print('get_Item', k)
        return 1024
    end;
    set_Item = function(self, k, v)
        print('set_Item', k, v)
    end;
    add_AEvent = function(self, cb)
        print('add_AEvent', cb)
        table.insert(self.evt, cb)
    end;
    remove_AEvent = function(self, cb)
       print('remove_AEvent', cb)
       for i, v in ipairs(self.evt) do
           if v == cb then
               table.remove(self.evt, i)
               break
           end
       end
    end;
    Start = function(self)
        print('Start')
        for _, cb in ipairs(self.evt) do
            cb(self.start, 2)
        end
        self.start = self.start + 1
    end;
    StaticFunc = function(a, b, c)
       print(a, b, c)
    end;
    GenericTest = function(self, a)
       print(self, a)
    end;
    Finalize = function(self)
       print('Finalize', self)
    end
})
```
