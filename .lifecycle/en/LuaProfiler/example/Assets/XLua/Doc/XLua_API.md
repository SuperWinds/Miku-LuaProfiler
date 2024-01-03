
## C# API
### LuaEnv Class
#### object[] DoString(string chunk, string chunkName = "chunk", LuaTable env = null)
Description:

```
Executes a chunk of code.
```

Parameters:

```
chunk: A string of Lua code;
chunkName: Used in the debug display information when an error occurs, indicating the line of error in a certain code chunk;
env: The environment variables for this code chunk;
```
Return Value:

```
The return values of the return statement in the code chunk;
For example: return 1, "hello", DoString will return an array containing two objects, one is a double type 1, the other is a string type "hello"
```

Example:

```
LuaEnv luaenv = new LuaEnv();
object[] ret = luaenv.DoString("print('hello')\r\nreturn 1");
UnityEngine.Debug.Log("ret=" + ret[0]);
luaenv.Dispose();
```

#### T LoadString<T>(string chunk, string chunkName = "chunk", LuaTable env = null)

Description:

```
Loads a chunk of code but does not execute it, only returns a type that can be specified as a delegate or a LuaFunction
```

Parameters:

```
chunk: A string of Lua code;
chunkName: Used in the debug display information when an error occurs, indicating the line of error in a certain code chunk;
env: The environment variables for this code chunk;
```

Return Value:

```
A delegate or LuaFunction class representing the code chunk;
```

#### LuaTable Global;

Description:

```
Represents the Lua global environment LuaTable
```

### void Tick()

Description:

```
Clears Lua's unreleased LuaBase objects (such as LuaTable, LuaFunction), among other things.
It needs to be called regularly, such as in MonoBehaviour's Update method.
```

### void AddLoader(CustomLoader loader)

Description:

```
Adds a custom loader
```

Parameters:

```
loader: A delegate that includes the loading function, of the type delegate byte[] CustomLoader(ref string filepath). When a file is required, this loader will be called back, with the parameter being the argument used to call require. If the loader finds the file, it can read it into memory and return a byte array. If debugging support is needed, the filepath should be set to a path that the IDE can find (either relative or absolute)
```

#### void Dispose()

Description:

```
Disposes of the LuaEnv.
```

> LuaEnv usage suggestion: Have only one instance globally, call the GC method in Update, and call Dispose when it is no longer needed.

### LuaTable Class

#### T Get<T>(string key)

Description:

```
Gets the value under key of type T, returns null if it does not exist or if the type does not match;
```

#### T GetInPath<T>(string path)

Description:

```
The difference from Get is that this function recognizes the "." in the path, for example, var i = tbl.GetInPath<int>("a.b.c") is equivalent to executing i = tbl.a.b.c in Lua, avoiding multiple calls to Get just to obtain intermediate variables, which is more efficient.
```

#### void SetInPath<T>(string path, T val)

Description:

```
The setter corresponding to GetInPath<T>;
```

#### void Get<TKey, TValue>(TKey key, out TValue value)

Description:

```
 The above API only allows keys to be strings, while this API has no such restriction;
```

#### void Set<TKey, TValue>(TKey key, TValue value)

Description:

```
 The setter corresponding to Get<TKey, TValue>;
```

#### T Cast<T>()

Description:

```
Converts the table into a type specified by T, which can be an interface declared with CSharpCallLua, a class or struct with a default constructor, a Dictionary, List, etc.
```

#### void SetMetaTable(LuaTable metaTable)

Description:

```
Sets metaTable as the metatable for the table
```

### LuaFunction Class

> Note: Accessing Lua functions using this class incurs boxing and unboxing overhead. For performance considerations, avoid using this class where frequent calls are necessary. It is recommended to obtain a delegate through table.Get<ABCDelegate> and then call it (assuming ABCDelegate is a C# delegate). Before using table.Get<ABCDelegate>, please add ABCDelegate to the code generation list.

#### object[] Call(params object[] args)

Description:

```
Calls a Lua function with variable arguments and returns the return values of that call.
```

#### object[] Call(object[] args, Type[] returnTypes)

Description:

```
Calls a Lua function and specifies the types of the return parameters, which the system will automatically convert to the specified types.
```

#### void SetEnv(LuaTable env)

Description:

```
Equivalent to Lua's setfenv function.
```

## Lua API

### CS Object

#### CS.namespace.class(...)

Description:

```
Calls a C# type constructor and returns an instance of the type
```

Example:

```
local v1 = CS.UnityEngine.Vector3(1,1,1)
```

#### CS.namespace.class.field

Description:

```
Accesses a C# static member
```

Example:

```
Print(CS.UnityEngine.Vector3.one)
```

#### CS.namespace.enum.field

Description:

```
Accesses an enumeration value
```

#### typeof Function

Description:

```
Similar to the typeof keyword in C#, returns a Type object, such as one of the overloads of GameObject.AddComponent which requires a Type parameter
```

Example:

```
newGameObj:AddComponent(typeof(CS.UnityEngine.ParticleSystem))
```

#### Unsigned 64-bit Support

##### uint64.tostring

Description:

```
Converts an unsigned number to a string.
```

##### uint64.divide

Description:

```
Performs division on an unsigned number.
```

##### uint64.compare

Description:

```
Compares unsigned numbers, returns 0 if equal, a positive number if greater, and a negative number if less.
```

##### uint64.remainder

Description:

```
Calculates the remainder of an unsigned number.
```

##### uint64.parse

Description:

```
Converts a string to an unsigned number.
```

#### xlua.structclone

Description:

```
Clones a C# struct
```

#### xlua.private_accessible(class)
Description:

```
Makes a class's private fields, properties, methods, etc., accessible
```
Example:

```
xlua.private_accessible(CS.UnityEngine.GameObject)
```

#### xlua.get_generic_method
Description:

```
Obtains a generic method
```
Example:

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

#### cast Function

Description:

```
Specifies accessing an object with a particular interface, which is useful when the implementing class is not accessible (e.g., marked as internal). In such cases, do the following (assuming the calc object below implements the C# PerformentTest.ICalc interface)
```

Example:

```
cast(calc, typeof(CS.PerformentTest.ICalc))
```

Then there are no other APIs. Accessing a csharp object is like accessing a table, calling a function is like calling a lua function, and you can also access c# operators through operators. Here is an example:

```
local v1 = CS.UnityEngine.Vector3(1,1,1)
local v2 = CS.UnityEngine.Vector3(1,1,1)
v1.x = 100
v2.y = 100
print(v1, v2)
local v3 = v1 + v2
print(v1.x, v2.x)
print(CS.UnityEngine.Vector3.one)
print(CS.UnityEngine.Vector3.Distance(v1, v2))
```

## Type Mapping

### Basic Data Types

|C# Type|Lua Type|
|---|---|
|sbyte, byte, short, ushort, int, uint, double, char, float|number|
|decimal|userdata|
|long, ulong|userdata/lua_Integer(lua53)|
|byte[]|string|
|bool|boolean|
|string|string|

### Complex Data Types

|C# Type|Lua Type|
|---|---|
|LuaTable|table|
|LuaFunction|function|
|Instance of class or struct|userdata, table|
|method, delegate|function|

#### LuaTable:

If the C# side specifies LuaTable as the input type from the Lua side (including C# method input parameters or Lua method return values), the Lua side must be a table. Or, a Lua side table is converted to a LuaTable on the C# side if the type is not specified.

#### LuaFunction:

If the C# side specifies LuaFunction as the input type from the Lua side (including C# method input parameters or Lua method return values), the Lua side must be a function. Or, a Lua side function is converted to a LuaFunction on the C# side if the type is not specified.

#### LuaUserData:

Corresponds to Lua userdata for non-C# managed objects.

#### Instance of class or struct:

An instance of a class or struct passed from C# maps to Lua's userdata, and its members are accessed through __index. If the C# side specifies an object of a certain type as input from the Lua side, Lua side userdata of that type instance can be used directly; if the specified type has a default constructor, Lua side tables will be automatically converted. The conversion rule is: call the constructor to create an instance, and assign values to its members after converting the corresponding fields from the table to C# values.

#### method, delegate:

Member methods and delegates correspond to Lua side functions. Ordinary and reference parameters on the C# side correspond to Lua side function parameters; C# side return values correspond to Lua's first return value; reference and out parameters correspond to Lua's second to nth return values in sequence.

## Macros

#### HOTFIX_ENABLE

Enables the hotfix feature.

#### NOT_GEN_WARNING

Prints warnings during reflection.

#### GEN_CODE_MINIMIZE

Generates code in a way that minimizes code segments.
