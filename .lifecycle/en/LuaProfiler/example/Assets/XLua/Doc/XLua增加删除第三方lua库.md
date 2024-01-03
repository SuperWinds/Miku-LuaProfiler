
## What & Why

XLua's current built-in extension libraries include:

* 64-bit integer support for LuaJIT;
* Tools for function call time consumption and memory leak detection;
* The luasocket library for supporting ZeroBraneStudio;
* tdr for Lua;

As the number of projects using XLua increases and their usage deepens, these few extensions are no longer sufficient for project teams. Due to the significant differences in extensions across projects and the sensitivity of mobile platforms to the size of installation packages, XLua cannot meet these needs through pre-integration. This is the reason for this tutorial.

This tutorial will use lua-rapidjson as an example to step by step explain how to add C/C++ extensions to XLua. Of course, once you know how to add them, you'll naturally understand how to remove them, allowing project teams to delete pre-integrated extensions that are not needed.

## How

In three steps:

1. Modify the build files and project settings to compile the extensions to be integrated into the XLua Plugin;
2. Use xLua's C# API to allow extensions to be loaded on demand (when required in Lua code);
3. Optionally, if your extension requires the use of 64-bit integers, you can implement cooperation with C# using XLua's 64-bit extension library.

### 1. Add Extension & Compile

Preparation:

1. Unzip the XLua C source code package to a directory at the same level as the Assets in your Unity project.

   Download the lua-rapidjson code and place it as you prefer. In this tutorial, the rapidjson header files are placed in the `$UnityProj\build\lua-rapidjson\include` directory, and the extension source code rapidjson.cpp is placed in the `$UnityProj\build\lua-rapidjson\source` directory (note: `$UnityProj` refers to your project directory).

2. Add the extension to CMakeLists.txt

   XLua's platform plugins are compiled using CMake, which has the advantage of having all platform compilations written in a single Makefile, with most of the compilation logic being cross-platform.

   The CMakeLists.txt that comes with XLua provides extension points (all are lists) for third-party extensions:

   1.  THIRDPART_INC: The search path for third-party extension header files.
   2.  THIRDPART_SRC: The source code for third-party extensions.
   3.  THIRDPART_LIB: The libraries that third-party extensions depend on.

   Below is how to add rapidjson:

   ```
    #begin lua-rapidjson
    set (RAPIDJSON_SRC lua-rapidjson/source/rapidjson.cpp)
    set_property(
        SOURCE ${RAPIDJSON_SRC}
        APPEND
        PROPERTY COMPILE_DEFINITIONS
        LUA_LIB
    )
    list(APPEND THIRDPART_INC lua-rapidjson/include)
    set (THIRDPART_SRC ${THIRDPART_SRC} ${RAPIDJSON_SRC})
    #end lua-rapidjson
   ```

   For the complete code, please see the attachment.

3. Compile for Each Platform

   All compilation scripts are named in this format: `make_<platform>_<lua_version>.<suffix>`.

   For example, the Windows 64-bit Lua 5.3 version is `make_win64_lua53.bat`, and the Android LuaJIT version is `make_android_luajit.sh`. Execute the corresponding script for the version you want to compile.

   After executing the compilation script, it will automatically be copied to the `plugin_lua53` or `plugin_luajit` directory, the former for the Lua 5.3 version and the latter for LuaJIT.

   The accompanying Android script is used under Linux, and the NDK path at the beginning of the script should be modified according to actual conditions.

### 2. C# Side Integration

All Lua C extension libraries will provide a `luaopen_xxx` function, where `xxx` is the name of the dynamic library. For example, the function for the lua-rapidjson library is `luaopen_rapidjson`. This type of function is automatically called by the Lua virtual machine when loading the dynamic library. However, on mobile platforms, due to iOS restrictions, we cannot load dynamic libraries but instead compile them directly into the process.

To this end, XLua provides an API to replace this functionality (a member method of LuaEnv):

```
public void AddBuildin(string name, LuaCSFunction initer)
```

Parameters:

```
name: The name of the buildin module, the parameter entered when using `require`;
initer: The initialization function, with the prototype `public delegate int LuaCSFunction(IntPtr L)`. It must be a static function and decorated with the `MonoPInvokeCallbackAttribute` attribute. This API will check for these two conditions.
```

Let's look at how to use it with the call to `luaopen_rapidjson`.

Extend the LuaDLL.Lua class, use P/Invoke to export `luaopen_rapidjson` to C#, and then write a static function that conforms to the definition of `LuaCSFunction`. You can do some initialization work in it, such as calling `luaopen_rapidjson`. Here is the complete code:

```
namespace LuaDLL
{ 
    public partial class Lua
    { 
        [DllImport("LUADLL", CallingConvention = CallingConvention.Cdecl)]
        public static extern int luaopen_rapidjson(IntPtr L);

        [MonoPInvokeCallback(typeof(LuaCSFunction))]
        public static int LoadRapidJson(IntPtr L)
        {
            return luaopen_rapidjson(L);
        }
    }
}
```

Then call `AddBuildin`:

```
luaenv.AddBuildin("rapidjson", LuaDLL.Lua.LoadRapidJson);
```

After that, you can try the extension in Lua code:

```
local rapidjson = require('rapidjson')
local t = rapidjson.decode('{"a":123}')
print(t.a)
t.a = 456
local s = rapidjson.encode(t)
print('json', s)
```

### 3. 64-bit Transformation

Include the `i64lib.h` file into the file that requires 64-bit modification. The APIs of this header file are as follows:

```
// Push an int64/uint64 onto the stack
void lua_pushint64(lua_State* L, int64_t n);
void lua_pushuint64(lua_State* L, uint64_t n);
// Check if the value at stack position pos is an int64/uint64
int lua_isint64(lua_State* L, int pos);
int lua_isuint64(lua_State* L, int pos);
// Retrieve an int64/uint64 from stack position pos
int64_t lua_toint64(lua_State* L, int pos);
uint64_t lua_touint64(lua_State* L, int pos);
```

The use of these APIs depends on the situation. You can refer to the attachment that comes with this article (the rapidjson.cpp file).

Compilation project related modifications
