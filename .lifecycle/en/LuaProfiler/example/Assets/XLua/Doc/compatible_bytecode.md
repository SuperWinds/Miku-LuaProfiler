
# Universal Bytecode

Many projects wish to load their Lua source code by compiling it with luac, but official Lua has a drawback: bytecode is divided into 32-bit and 64-bit versions. In other words, a 32-bit Lua environment can only run bytecode compiled by a 32-bit luac.

To address this, xLua has made some modifications to the Lua source code, allowing the compilation of a single bytecode that can be used across platforms.

## Precautions

* 1. If you implement the changes described in this document, your xLua will not be able to load bytecode compiled by the official luac;

* 2. As of September 14, 2018, it is known that this modification method has been running normally in a project that has been online for over a month, but this does not guarantee that the method will be problem-free in all situations.

## Operation Guide

### 1. Compile xLua's Plugins

Modify the compilation scripts for each platform by adding the -DLUAC_COMPATIBLE_FORMAT=ON parameter to the cmake command. For example, after modifying make_win64_lua53.bat, it should look like this:

```bash
mkdir build64 & pushd build64
cmake -DLUAC_COMPATIBLE_FORMAT=ON -G "Visual Studio 14 2015 Win64" ..
popd
cmake --build build64 --config Release
md plugin_lua53\Plugins\x86_64
copy /Y build64\Release\xlua.dll plugin_lua53\Plugins\x86_64\xlua.dll
pause
```

Use the modified compilation scripts to recompile the xLua libraries for each platform and replace the corresponding files in the original Plugins directory.

## 2. Compile luac to Generate Compatible Format Bytecode (this specific luac must be used in conjunction with the Plugins from step 1)

Go to [this location](../../../build/luac/). If you want to compile a Windows version, run make_win64.bat; for Mac or Linux, use make_unix.sh.

## 3. Load Bytecode

Load it through CustomLoader. For detailed information about CustomLoader, please refer to the tutorial. A common mistake at this step is to use some Encoding to load a binary file, which will corrupt the Lua bytecode file format. Remember to load in binary mode.

## PS: OpCode Modification

If a project wants to modify the bytecode into a proprietary format, simply make the changes directly in the Lua source code (currently lua-5.3.5) and then re-execute steps 1 and 2 above.
