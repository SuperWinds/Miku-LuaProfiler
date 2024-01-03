
## xLua Tutorial

### Lua File Loading

1. Execute String

   The most basic method is to use `LuaEnv.DoString` to execute a string, which must conform to Lua syntax. For example:

   ```
    luaenv.DoString("print('hello world')")
   ```

   The complete code can be found in the XLua\Tutorial\LoadLuaScript\ByString directory.
      > However, this method is not recommended. The method introduced below is preferable.

2. Load Lua File

   Simply use Lua's `require` function, for example:

   ```
    DoString("require 'byfile'")
   ```

   The complete code can be found in the XLua\Tutorial\LoadLuaScript\ByFile directory.

   `require` actually calls a series of loaders to load the file. If one succeeds, no further attempts are made. If all fail, a file-not-found error is reported. In addition to the native loader, xLua also adds a loader that loads from Resources. Note that because Resources only supports limited suffixes, Lua files placed under Resources must have a `.txt` suffix (see the included example).

   The recommended way to load Lua scripts is: the entire program has only one `DoString("require 'main'")`, and then other scripts are loaded in `main.lua` (similar to the command line execution of Lua scripts: `lua main.lua`).

   Some may ask: What if my Lua file is downloaded, or extracted from a custom file format, or needs to be decrypted, etc.? Good question, xLua's custom Loader can meet these needs.

3. Custom Loader

   Adding a custom loader in xLua is straightforward and involves only one interface:

   ```
    public delegate byte[] CustomLoader(ref string filepath);
    public void LuaEnv.AddLoader(CustomLoader loader)
   ```

   You can register a callback with `AddLoader`. The callback parameter is a string, which is passed transparently when `require` is called in Lua code. The callback can then load the specified file based on this parameter. If debugging support is needed, the `filepath` should be modified to the real path. The callback's return value is a byte array; if null, the loader cannot find the file; otherwise, it is the content of the Lua file. With this, it’s simple. Use IIPS’s IFS? No problem. Write a loader to call the IIPS interface to read file content. File already encrypted? No problem, write your own loader to read and decrypt the file before returning it. For a complete example, see XLua\Tutorial\LoadLuaScript\Loader.

### C# Access to Lua

This refers to C# actively initiating access to Lua data structures. The examples involved in this chapter can be found under XLua\Tutorial\CSharpCallLua.

1. To obtain a global basic data type, access `LuaEnv.Global`. It has a templated `Get` method that can specify the return type.

   ```
    luaenv.Global.Get<int>("a")
    luaenv.Global.Get<string>("b")
    luaenv.Global.Get<bool>("c")
   ```

2. Access a Global Table

   Use the `Get` method mentioned above. What type should be specified?
1.     Map to an ordinary class or struct
       Define a class with public properties corresponding to the table's fields and a parameterless constructor. For example, for `{f1 = 100, f2 = 100}`, you can define a class with `public int f1; public int f2;`. xLua will instantiate an object and assign the corresponding fields.
       A table's properties can be more or less than a class's properties. It can nest other complex types. Note that this process is a value copy; if the class is complex, the cost can be significant. Also, modifying the class's field values will not sync to the table and vice versa.
       This functionality can be optimized by adding types to GCOptimize during generation. For details, refer to the configuration introduction document. Is there a reference mapping? Yes, as follows:

2.     Map to an Interface

       This method depends on generated code (if no code is generated, an `InvalidCastException` exception will be thrown). The code generator will create an instance of this interface. Getting a property will access the corresponding table field, and setting a property will update the corresponding field. You can even access Lua functions through interface methods.

3.     A more lightweight by-value method: Map to `Dictionary<>`, `List<>`

       If you don't want to define a class or interface, consider this method, provided the keys and values in the table are consistent.

4.     Another by-ref method: Map to the `LuaTable` class

       This method does not require code generation, but it is slower and lacks type checking compared to method 2.

3. Access a Global Function

   Still use the `Get` method, but the difference is in the type mapping.
1.     Map to a Delegate
       This is the recommended approach, as it offers better performance and type safety. The downside is that it requires code generation (if no code is generated, an `InvalidCastException` exception will be thrown).
       Declare a delegate with a parameter for each function argument. For multiple return values, map them from left to right to C#'s output parameters, which include return values, `out` parameters, and `ref` parameters.
       All parameter and return value types are supported, including various complex types, `out`, and `ref` modifiers, and even returning another delegate.
       Using a delegate is straightforward, just use it like a regular function.

2.     Map to `LuaFunction`

       The pros and cons of this method are the opposite of the first. It's also simple to use. `LuaFunction` has a variadic `Call` function that can pass any type and number of parameters. The return value is an object array, corresponding to Lua's multiple return values.

4. Usage Recommendations

1.     Accessing Lua global data, especially tables and functions, is costly. It is recommended to minimize such access. For example, during initialization, obtain the Lua function to be called once (mapped to a delegate), save it, and then directly call that delegate. The same applies to tables.

2.     If the Lua side's implementation is provided in the form of delegates and interfaces, users can be completely decoupled from xLua: a dedicated module can handle xLua initialization and the mapping of delegates and interfaces, then set these delegates and interfaces where they are needed.

### Lua Calls C#

> The examples involved in this chapter can be found under XLua\Tutorial\LuaCallCSharp

#### New C# Object

You create a new object in C# like this:

```
var newGameObj = new UnityEngine.GameObject();
```

In Lua, it corresponds to:

```
local newGameObj = CS.UnityEngine.GameObject()
```

It's basically similar, except:

```
1. There is no `new` keyword in Lua;
2. All C# related items are placed under `CS`, including constructors, static member properties, and methods.
```

If there are multiple constructors, xlua supports overloading. For instance, to call GameObject's constructor with a string parameter, you would write:

```
local newGameObj2 = CS.UnityEngine.GameObject('helloworld')
```

#### Access C# Static Properties and Methods

##### Read Static Properties

```
CS.UnityEngine.Time.deltaTime
```

##### Write Static Properties

```
CS.UnityEngine.Time.timeScale = 0.5
```

##### Call Static Method

```
CS.UnityEngine.GameObject.Find('helloworld')
```

Tip: If you frequently access a class, you can first reference it with a local variable and then access it, which reduces typing time and improves performance:

```
local GameObject = CS.UnityEngine.GameObject
GameObject.Find('helloworld')
```

#### Access C# Member Properties and Methods

##### Read Member Properties

```
testobj.DMF
```

##### Write Member Properties

```
testobj.DMF = 1024
```

##### Call Member Method

Note: When calling a member method, the object must be passed as the first parameter. It is recommended to use the colon syntax sugar, as follows:

```
testobj:DMFunc()
```

##### Parent Class Properties and Methods

xlua supports accessing the base class's static properties and methods (through derived classes), and accessing the base class's member properties and member methods (through instances of derived classes).

##### Input and Output Attributes of Parameters (out, ref)

Lua's parameter processing rules: C#'s ordinary parameters are considered input parameters, `ref`-modified ones are input parameters, `out` is not counted, and then they correspond to the actual parameter list of the Lua call from left to right.

Lua's return value processing rules: C#'s return value (if any) is considered a return value, `out` is a return value, `ref` is a return value, and then they correspond to Lua's multiple return values from left to right.

##### Overloaded Methods
Directly access overloaded functions through different parameter types, for example:

```
testobj:TestFunc(100)
testobj:TestFunc('hello')
```

The integer parameter `TestFunc` and the string parameter `TestFunc` will be accessed separately.

Note: xlua supports the calling of overloaded functions to a certain extent because Lua's types are not as rich as C#'s, and there are one-to-many situations. For example, C#'s `int`, `float`, and `double` all correspond to Lua's `number`. In the example above, if `TestFunc` has these overloaded parameters, the first line will not be able to distinguish between them and can only call one (the one that appears first in the generated code).

##### Operators

Supported operators are: `+`, `-`, `*`, `/`, `==`, unary `-`, `<`, `<=`, `%`, `[]`.

##### Methods with Default Parameter Values

Just like calling a function with default value parameters in C#, if the actual parameters given are fewer than the formal parameters, default values will be used.

##### Variadic Methods
For a C# method like the following:

```
void VariableParamsFunc(int a, params string[] strs)
```

It can be called in Lua like this:

```
testobj:VariableParamsFunc(5, 'hello', 'john')
```

##### Using Extension Methods

Defined in C#, they can be used directly in Lua.

##### Generic (Template) Methods

Not directly supported, but can be encapsulated and called through the Extension methods feature.

##### Enumeration Types

Enumeration values are like static properties under the enumeration type.

```
testobj:EnumTestFunc(CS.Tutorial.TestEnum.E1)
```

The `EnumTestFunc` function's parameter is of the `Tutorial.TestEnum` type.

Enumeration classes support the `__CastFrom` method, which can convert an integer or string to an enumeration value, for example:

```
CS.Tutorial.TestEnum.__CastFrom(1)
CS.Tutorial.TestEnum.__CastFrom('E1')
```

##### Delegate Usage (Call, +, -)

C#'s delegate call: the same as calling an ordinary Lua function.

`+` Operator: Corresponds to C#'s `+` operator, chaining two calls into one. The right operand can be a C# delegate of the same type or a Lua function.

`-` Operator: Opposite of `+`, removes a delegate from the call chain.

> Ps: A delegate property can be assigned a Lua function.

##### Event

For example, testobj has an event defined like this: `public event Action TestEvent;`

Add event callback:

```
testobj:TestEvent('+', lua_event_callback)
```

Remove event callback:

```
testobj:TestEvent('-', lua_event_callback)
```

##### 64-bit Integer Support

```
The Lua53 version maps 64-bit integers (long, ulong) to native 64-bit integers, while the LuaJIT version, equivalent to the Lua 5.1 standard, does not support 64-bit natively. xLua provides a 64-bit support extension library, mapping C#'s long and ulong to userdata:

Supports 64-bit operations, comparisons, and printing in Lua.

Supports operations and comparisons with Lua numbers.

Note that in the 64-bit extension library, there is actually only int64. ulong will also be cast to long before being passed to Lua. For some operations and comparisons on ulong, we adopt the same support method as Java, providing a set of APIs. For details, please see the API documentation.
```

##### Automatic Conversion of C# Complex Types and Tables

For a C# complex type with a parameterless constructor, a table on the Lua side can directly replace it. The table should have corresponding fields for the complex type's public fields. It supports function parameter passing, property assignment, etc. For example, the B structure (class is also supported) is defined as follows in C#:

```
public struct A
{
    public int a;
}

public struct B
{
    public A b;
    public double c;
}
```

A class has a member function like this:

```
void Foo(B b)
```

In Lua, you can call it like this:

```
obj:Foo({b = {a = 100}, c = 200})
```

##### Get Type (Equivalent to C#'s typeof)

For example, to get the Type information of the `UnityEngine.ParticleSystem` class, you can do this:

```
typeof(CS.UnityEngine.ParticleSystem)
```

##### "Strong" Cast

Lua has no types, so there's no "cast" as in strongly typed languages, but there's something similar: telling xLua to use the specified generated code to call an object. This is useful when a third-party library exposes an interface or abstract class with a hidden implementation class, preventing code generation for the implementation class. The implementation class will be recognized by xLua as ungenerated code and accessed using reflection. If this call is frequent, it can affect performance. In such cases, we can add the interface or abstract class to the generated code, then specify to use this generated code to access:

```
cast(calc, typeof(CS.Tutorial.Calc))
```

The above specifies using the generated code of `CS.Tutorial.Calc` to access the `calc` object.
