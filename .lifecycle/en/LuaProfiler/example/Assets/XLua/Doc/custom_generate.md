
## Secondary Development of the Generation Engine

The xLua generation engine supports secondary development, allowing you to generate various text-based files (such as code, configurations, etc.). The generation of xLua's own link.xml file is accomplished by a generation engine plugin. Other use cases, like generating auto-complete configuration files for Lua IDEs, can also be achieved with this feature.

## General Introduction

Plugins need to provide two things: 1. A template for generating files; 2. A callback function that accepts user configurations and returns the data to be injected into the template and the file's output stream.

## Template Syntax

The template syntax is straightforward, consisting of only three elements:

* eval: The syntax is `<%=exp%>`, where `exp` is any expression. The value of `exp` will be evaluated and output as a string;
* code: The syntax is `<% if true then end%>`. The blue part represents any Lua code, which will be executed;
* literal: Everything other than `eval` and `code` is output exactly as it appears.

Example:

```xml
<%
require "TemplateCommon"
%>

<linker>
<%ForEachCsList(assembly_infos, function(assembly_info)%>
	<assembly fullname="<%=assembly_info.FullName%>">
	    <%ForEachCsList(assembly_info.Types, function(type)%>
		<type fullname="<%=type:ToString()%>" preserve="all"/>
	    <%end)%>
	</assembly>
<%end)%>
</linker>
```

TemplateCommon includes some predefined functions, such as `ForEachCsList`. You can search the project's `TemplateCommon.lua.txt` to see what functions are available; it's just standard Lua. You can also write your own set.

## API

```csharp
public static void CSObjectWrapEditor.Generator.CustomGen(string template_src, GetTasks get_tasks)
```

* `template_src`: The source code of the template;
* `get_tasks`: A callback function of type `GetTasks`, used to receive user configurations and return the data to be injected into the template and the file's output stream;

```csharp
public delegate IEnumerable<CustomGenTask> GetTasks(LuaEnv lua_env, UserConfig user_cfg);
```

* `lua_env`: The `LuaEnv` object, because the template data to be returned needs to be placed into a `LuaTable`, requiring the use of `LuaEnv.NewTable`;
* `user_cfg`: The user's configuration;
* `return`: In the return value, `CustomGenTask` represents a file to be generated, and the `IEnumerable` type indicates that multiple files can be generated from the same template;

```csharp
public struct UserConfig
{
    public IEnumerable<Type> LuaCallCSharp;
    public IEnumerable<Type> CSharpCallLua;
    public IEnumerable<Type> ReflectionUse;
}
```

```csharp
public struct CustomGenTask
{
    public LuaTable Data;
    public TextWriter Output;
}
```

Example:

```csharp
public static IEnumerable<CustomGenTask> GetTasks(LuaEnv lua_env, UserConfig user_cfg)
{
    LuaTable data = lua_env.NewTable();
    var assembly_infos = (from type in user_cfg.ReflectionUse
                          group type by type.Assembly.GetName().Name into assembly_info
                          select new { FullName = assembly_info.Key, Types = assembly_info.ToList() }).ToList();
    data.Set("assembly_infos", assembly_infos);

    yield return new CustomGenTask
    {
        Data = data,
        Output = new StreamWriter(GeneratorConfig.common_path + "/link.xml",
        false, Encoding.UTF8)
    };
}
```

* Here, only one file is generated, hence only one `CustomGenTask` is returned;
* `data` is the data to be used by the template, with an `assembly_infos` field included. How to use this field can be referred back to the template section;

## Label

Typically, you can execute a custom generation operation by opening a menu through `MenuItem`, but sometimes you may want the generation operation to be directly triggered by xLua's "Generate Code" menu. In such cases, you would use `CSObjectWrapEditor.GenCodeMenu`.

Example:

```csharp
[GenCodeMenu] // Add to the Generate Code menu
public static void GenLinkXml()
{
    Generator.CustomGen(ScriptableObject.CreateInstance<LinkXmlGen>().Template.text, GetTasks);
}
```

ps: All related code mentioned above is located in the `XLua/Src/Editor/LinkXmlGen` directory, which is also where the link.xml generation functionality, mentioned at the beginning of the article, is implemented.
