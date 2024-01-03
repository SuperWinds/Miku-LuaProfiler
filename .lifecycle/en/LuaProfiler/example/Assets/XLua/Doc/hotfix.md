
## Usage

1. Enable this feature

Add the HOTFIX_ENABLE macro (under Unity3D's File->Build Setting->Scripting Define Symbols). This macro must be set separately for the editor and each mobile platform! If using automated packaging, note that macros set via API in the code will not take effect; they must be set in the editor.

(It is recommended to develop business code with HOTFIX_ENABLE turned off, and only enable HOTFIX_ENABLE when building the mobile version or developing patches in the editor)

2. Execute the XLua/Generate Code menu.

3. Injection. This step will be performed automatically during mobile package build. For patch development in the editor, manually execute the "XLua/Hotfix Inject In Editor" menu. Only when "hotfix inject finish!" or "had injected!" is printed is it considered successful; otherwise, an error message will be printed.

If "hotfix inject finish!" or "had injected!" has been printed, but executing xlua.hotfix still reports an error like "xlua.access, no field __Hitfix0_Update", it means either the class is not configured in the Hotfix list, or the injection was successful but a subsequent compilation overwrote the injection result.

## Constraints

Static constructors are not supported.

Currently, only hot patches for code under Assets are supported, not for engines or C# system libraries.

## API
xlua.hotfix(class, [method_name], fix)

* Description: Inject a Lua patch
* class: C# class, represented in two ways: CS.Namespace.TypeName or as a string "Namespace.TypeName". The string format must be consistent with C#'s Type.GetType requirements. If it's a non-Public nested type (Nested Type), it can only be represented as a string "Namespace.TypeName+NestedTypeName";
* method_name: Optional method name;
* fix: If method_name is provided, fix will be a function; otherwise, a set of functions is provided through a table, organized with key as method_name and value as function.

base(csobj)

* Description: Subclass override functions call the parent class implementation via base.
* csobj: The object
* Return value: A new object, which can invoke methods on its base

Example (located in HotfixTest2.cs):

```lua
xlua.hotfix(CS.BaseTest, 'Foo', function(self, p)
    print('BaseTest', p)
    base(self):Foo(p)
end)
```

util.hotfix_ex(class, method_name, fix)

* Description: An enhanced version of xlua.hotfix, allowing the original function to be executed within the fix function. The downside is that fix execution will be slightly slower.
* method_name: Method name;
* fix: Lua function to replace the C# method.

## Identifying Types for Hot Update

There are two methods, as with other configurations:

Method one: Directly tag the class with Hotfix (not recommended, this example is for demonstration convenience);

Method two: Configure a list within a static field or property of a static class. Properties can be used for more complex configurations, such as creating a whitelist based on Namespace.

```csharp
// For other dlls outside of Assembly-CSharp.dll, the following code needs to be placed in the Editor directory
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

The Hotfix tag can set flags for customizing generated code and instrumentation.

* Stateless, Stateful

Legacy settings; the Stateful method has been removed in newer versions, as similar effects can be achieved with the xlua.util.state interface. See the example code in HotfixTest2.cs for usage.

Since Stateful is absent, the default is Stateless, so setting this flag is unnecessary.

* ValueTypeBoxing

Value type adaptation for delegates converges to object. The benefit is less code, but the downside is boxing and GC for value types, suitable for businesses sensitive to text segment size.

* IgnoreProperty

Do not inject into properties or generate adaptation code. Most properties are simple and less likely to err, so it is advised not to inject.

* IgnoreNotPublic

Do not inject or generate adaptation code for non-public methods. Non-public methods only called by their class, except for ones like MonoBehaviour's private methods that require reflection, do not need injection. However, this makes fixing more laborious as all public methods referencing the non-injected function must be rewritten.

* Inline

Do not generate adaptation delegates; instead, inject processing code directly into the function body.

* IntKey

Do not generate static fields; manage all injection points in an array.

Benefits: Minimal impact on the text segment.

Drawbacks: Less convenient than the default method; you need to specify which function to hotfix by ID, assigned during code injection. The function-to-ID map is saved in Gen/Resources/hotfix_id_map.lua.txt, timestamped, and backed up to the same directory level. Keep this file safe after mobile version release.

The file format is roughly as follows (note: this file is only for IntKey mode; without specifying IntKey mode injection, the file returns an empty table):

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

To replace HotfixTest's Update function, you must:

```lua
CS.XLua.HotfixDelegateBridge.Set(7, func)
```

For overloaded functions, one function name corresponds to multiple IDs, as with the Add function above.

Can automation be achieved? Yes, xlua.util provides the auto_id_map function, which allows you to specify the function to patch directly using the class and method name after one execution.

```lua
(require 'xlua.util').auto_id_map()
xlua.hotfix(CS.HotfixTest, 'Update', function(self)
        self.tick = self.tick + 1
        if (self.tick % 50) == 0 then
            print('<<<<<<<<Update in lua, tick = ' .. self.tick)
        end
    end)
```

The prerequisite is that hotfix_id_map.lua.txt is placed where it can be referenced by require 'hotfix_id_map'.

## Recommendations

* Tag all types likely to change with Hotfix;
* It's recommended to reflectively identify all delegate types involved in function parameters, fields, properties, and events, and tag them with CSharpCallLua;
* For business code, engine APIs, and system APIs that need high-performance access in Lua patches, tag them with LuaCallCSharp;
* Engine and system APIs may be trimmed by code (any C#-unreferenced areas will be trimmed). If you anticipate new API calls beyond C# code, add LuaCallCSharp or ReflectionUse to those API types;

## Patching

xlua can replace C#'s constructors, functions, properties, and events with Lua functions. Lua implementations are functions, e.g., attributes correspond to a getter and a setter function, events to an add and a remove function.

* Function

Pass the function name to method_name; it supports overloading, with different overloads forwarded to the same Lua function.

For example:

```csharp

// C# class to fix
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
｝

```

```lua

xlua.hotfix(CS.HotfixCalc, 'Add', function(self, a, b)
    return a + b
end)

```

The difference between static and member functions is that member functions add a self parameter, which is the C# object itself (corresponding to C#'s this) in Stateless mode.

Ordinary parameters correspond to Lua's parameters, ref parameters to a Lua parameter and return value, and out parameters to a Lua return value.

Generalized function patching follows the same rules as ordinary functions.

* Constructor

The constructor's method_name is ".ctor".

Unlike ordinary functions, the constructor's hot patch is not a replacement but a call to Lua after executing the original logic.

* Property

For a property named "AProp", the getter's method_name is get_AProp, and the setter's method_name is set_AProp.

* [] Operator

Assignment corresponds to set_Item, retrieval to get_Item. The first parameter is self, followed by key and value for assignment, and only key for retrieval, with the return value being the retrieved value.

* Other Operators

C#'s operators have internal representations, such as op_Addition for the + operator. Overriding this function overrides C#'s + operator.

* Event

For an event named "AEvent", the += operator is add_AEvent, and -= corresponds to remove_AEvent. Both functions have self as the first parameter and the delegate following the operator as the second.

After direct access to the private delegate corresponding to the event through xlua.private_accessible (no need to call xlua.private_accessible for versions greater than 2.1.11), the event can be triggered directly through the object's "&event name" field, such as self\['&MyEvent'\](), where MyEvent is the event name.

* Destructor

The method_name is "Finalize", with a self parameter.

Unlike ordinary functions, the destructor's hot patch is not a replacement but a call to the Lua function at the beginning, followed by the original logic.

* Generalized Type

The same rules apply, but it should be noted that each generalized type is an independent type after instantiation and can only be patched for the instantiated type. For example:

```csharp
public class GenericClass<T>
{
｝
```

You can only patch GenericClass\<double\>, GenericClass\<int\>, not GenericClass.

An example of patching GenericClass<double> is as follows:

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

* Unity Coroutine

Through util.cs_generator, a function can simulate an IEnumerator, using coroutine.yield similar to C#'s yield return. The following C# code and corresponding hotfix code have the same effect:

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

```csharp
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

* Entire Class

If you want to replace the entire class, you don't need to call xlua.hotfix repeatedly; it can be done all at once. Just provide a table organized by method_name = function.

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
