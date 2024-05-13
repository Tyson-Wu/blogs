---
title: xlua 代码扩展
date: 2024-5-19 19:01:34
categories:
- [Unity, xlua]
tags:
- Unity
- xlua
- lua
---

在使用xlua的过程中，可能需要自定义xlua，比如所Astar寻路算法，如果在lua中实现的话，效率是非常低的，所以可以考虑将实现方法放到C/C++，这样运行效率可以大大提升。
这里参照xlua官方[文档](https://github.com/Tencent/xLua/blob/master/Assets/XLua/Doc/XLua%E5%A2%9E%E5%8A%A0%E5%88%A0%E9%99%A4%E7%AC%AC%E4%B8%89%E6%96%B9lua%E5%BA%93.md),总结了一下。

- 首先下载[xlua](https://github.com/Tencent/xLua)源码。
- 下载完后可以用Unity打开这个项目，跑一下01_Helloworld案例，后面扩展xlua也将在这个案例上修改。
- 然后添加扩展代码，扩展代码是在和Assets同目录的build目录下。
- 这里我们沿用官方文档的例子，扩展lua-rapidjson。可以在github上搜索，路径的话就是[xpol/lua-rapidjson](https://github.com/xpol/lua-rapidjson)
- 下载lua-rapidjson源码
- 在xlua的build目录下新建文件夹lua-rapidjson，然后在lua-rapidjson文件夹下新建文件夹include和src。在官方文档中说的是新建include和source,其实名字是什么都可以，后面配置的时候对应上就行。
- 然后将lua-rapidjson源码中的`lua-rapidjson/rapidjson/include`目录全部拷贝到上面新建的include目录下。
- 再将lua-rapidjson源码中的`lua-rapidjson/src`目录全部拷贝到上面新建的src目录下。
- 上面两步就准备好了扩展的源码。
- 然后是将上面的源码关联到xlua的cmake上。
- 在xlua的build目录下找到CMakeLists.txt文件。在这个文件里面配置。配置内容如下。下面这个`lua-rapidjson/include`和`ua-rapidjson/src/`就是上面建的源文件。
``` bash
MARK_AS_ADVANCED(XLUA_PROJECT_DIR)

 # 下面是新增部分
 #begin lua-rapidjson
 set (RAPIDJSON_SRC 
    lua-rapidjson/src/Document.cpp
    lua-rapidjson/src/Schema.cpp
    lua-rapidjson/src/rapidjson.cpp
    lua-rapidjson/src/values.cpp
 )
 set_property(
     SOURCE ${RAPIDJSON_SRC}
     APPEND
     PROPERTY COMPILE_DEFINITIONS
     LUA_LIB
 )
 list(APPEND THIRDPART_INC  lua-rapidjson/include)
 set (THIRDPART_SRC ${THIRDPART_SRC} ${RAPIDJSON_SRC})
 #end lua-rapidjson
 # 上面是新增部分

if (NOT LUA_VERSION)
    set(LUA_VERSION "5.3.5")
endif()
```

- 配置好后，就可以尝试重新编译xlua源码。在xlua的build目录下找到make_win_lua54.bat文件，双击，会运行控制台命令，看看如果没有报错就说明成功了。
- 成功后会导出plugin_lua54，这里就是Unity Plugins下要用到的dll。将生成的Plugins拷贝到Unity下替换原本的Plugins。当然拷贝要关闭Unity才行。
- 替换好Plugins后，重新打开Unity。还是打开01_Helloworld案例。这里将在01_Helloworld案例中测试新增的lua-rapidjson
- 首选可以新建一个脚本目录`XLuaExt`，用来存放Xlua扩展脚本。当然也可以不建，后面的扩展代码直接写在`LuaDLL.cs`脚本中，不过后面更新xlua时就会被覆盖。
- 在`XLuaExt`目录下新建`LuaDLL.Rapidjson.cs`，并且在脚本中添加如下内容。这样C#脚本就关联上了xlua中的扩展。
```C#
using System.Runtime.InteropServices;
namespace XLua.LuaDLL
{
    public partial class Lua
    {
        [DllImport(LUADLL, CallingConvention = CallingConvention.Cdecl)]
        public static extern int luaopen_rapidjson(System.IntPtr L);

        [MonoPInvokeCallback(typeof(LuaDLL.lua_CSFunction))]
        public static int LoadRapidJson(System.IntPtr L)
        {
            return luaopen_rapidjson(L);
        }
    }
}
```

- 然后打开01_Helloworld案例的`Helloworld.cs`脚本，修改这个脚本成如下：
```C#
using UnityEngine;
using XLua;

namespace XLuaTest
{
    public class Helloworld : MonoBehaviour
    {
        // Use this for initialization
        void Start()
        {
            LuaEnv luaenv = new LuaEnv();
            luaenv.AddBuildin("rapidjson", XLua.LuaDLL.Lua.LoadRapidJson);
            luaenv.DoString("CS.UnityEngine.Debug.Log('hello world')");

            luaenv.DoString(@"

local rapidjson = require('rapidjson')
local t = rapidjson.decode('{""a"":123}')
print(t.a)
t.a = 456
local s = rapidjson.encode(t)
print('json', s)

");

            luaenv.Dispose();
        }

        // Update is called once per frame
        void Update()
        {

        }
    }
}
```

- 然后运行01_Helloworld案例。可以看到`LUA: json	{"a":456}`打印日志。