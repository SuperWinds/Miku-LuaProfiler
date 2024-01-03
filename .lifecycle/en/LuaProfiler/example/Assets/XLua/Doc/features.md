
# Features

## General

* Lua virtual machine support
*  Lua 5.3
*  LuaJIT 2.1
* Unity3D version support
*  Support for all versions
* Platform support
*  Windows 64/32
*  Android
*  iOS 64/32/bitcode
*  OSX
*  UWP
*  WebGL
* Interoperability technology
*  Generate adapter code
*  Reflection
* Ease of use
*  Ready to use upon extraction
*  No need to generate code during development
*  Seamless switching between generated code and reflection
*  Simpler non-GC API
*  Menu is straightforward and easy to understand
*  Configurations can be multiple, divided by module, or directly use Attribute tags on the target type
*  Auto-generate link.xml to prevent code stripping
*  Plugins are compiled with CMake, simplifying the process
*  Core code does not rely on generated code and can delete the generation directory at any time
* Performance
*  Lazyload technology to avoid overhead of unused types
*  Lua functions map to C# delegates, Lua tables map to interfaces, enabling interface-level operations without C# GC alloc
*  All basic value types, enums, and fields are value-type structs, with no C# GC alloc when passing between Lua and C#
*  LuaTable and LuaFunction provide interfaces for GC-free access
*  Optimal code generation through static analysis during the code generation phase
*  Support for pointer passing between C# and Lua
*  Automatically unreference destroyed UnityEngine.Objects
* Extensibility
*  Lua third-party extensions can be added without modifying code
*  Generation engine provides interfaces for secondary development

## Support for patching the following C# implementations

*  Constructors
*  Destructors
*  Member functions
*  Static functions
*  Generic functions
*  Operator overloading
*  Member properties
*  Static properties
*  Events

## Lua code loading

* Loading strings
*  Supports immediate execution after loading
*  Supports returning a delegate or LuaFunction after loading, which can accept script parameters upon invocation
* Files in the Resources directory
*  Directly require
* Custom loader
*  Triggered upon require in Lua
*  Require parameters passed transparently to the loader, which returns the Lua code
* Lua's original methods
*  All original Lua methods are preserved

## Lua calls C#

* Create C# objects
* C# static properties and fields
* C# static methods
* C# member properties and fields
* C# member methods
* C# inheritance
*  Subclass objects can directly call parent class methods and access parent class properties
*  Subclass modules can directly call parent class static methods and static properties
* Extension methods
*  Used as regular member methods
* Parameter input/output attributes (out, ref)
*  out corresponds to a Lua return value
*  ref corresponds to a Lua argument and a Lua return value
* Function overloading
*  Supports overloading
*  Due to fewer Lua data types compared to C#, indeterminate situations may arise, solvable via extension methods
* Operator overloading
*  Supported operators: +, -, *, /, ==, unary -, <, <=, %, []
*  Other operators can be invoked using extension methods
* Parameter default values
*  C# parameters with default values can be omitted in Lua
* Variadic parameters
*  Directly input parameters one by one in the corresponding variadic section, without needing to expand them into an array
* Generic method invocation
*  Static methods can be self-encapsulated for use
*  Member functions can be encapsulated using extension methods
* Enum types
*  Conversion from numbers or strings to enums
* Delegate
*  Invoke a C# delegate
*  + operator
*  - operator
*  Pass a Lua function as a C# delegate to C#
* Event
*  Add event callbacks
*  Remove event callbacks
* 64-bit integers
*  Pass without GC and with no loss of precision
*  Use native 64-bit support in Lua 5.3
*  Can perform operations with numbers
*  Java-style support for unsigned 64-bit integers
* Automatic conversion of tables to C# complex types
*  obj.complexField = {a = 1, b = {c = 1}}, where obj is a C# object and complexField is a nested struct or class
* typeof
*  Corresponds to C#'s typeof operator, returns a Type object
* Direct cloning on the Lua side
* decimal
*  Passed without GC and with no loss of precision

## C# calls Lua

* Invoke Lua functions
*  Invoke Lua functions as delegates
*  Invoke Lua functions with LuaFunction
* Access Lua tables
*  LuaTable's generic Get/Set interface, call without GC, specifying Key and Value types
*  Access using interfaces annotated with CSharpCallLua
*  Values copied to structs, classes

## Lua virtual machine

* Virtual machine GC parameter reading and setting

## Toolchain

* Lua Profiler
*  Sort by total function call duration, average call duration, and call count
*  Display Lua function names and their file names and line numbers
*  If a C# function, it will indicate that it is a C# function
* Support for real device debugging
