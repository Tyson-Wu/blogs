---
title: EmmyLua 的使用方法
date: 2021-01-30 10:38:34
categories:
- [Lua, EmmyLua]
tags:
- Lua
- EmmyLua
- IntelliJ IDEA
---

EmmyLua是Jetbrains IDE的一个插件，它提供了一套注解的语法，可以帮助IDE进行代码之间索引跳转，以及变量名称提示等功能。总而言之，EmmyLua是丰富IDE的功能，和实际的代码没有关系，IDE的跳转也只是限于注释之间。

## 类的声明
EmmyLua使用@class来模拟面向对象编程，支持继承和属性
```
--- my supper class
---@class Transport
local p = Class()

--- 子类
---@class Car : Transport
local cls = Class(p)
function c:test()
end
```
其格式如下：
```
--@class my_type[: parent_type] [@comment]
```
举个例子：
```
---@class Car : Transport @define class Car extends Transport
local cls = class()
function clas:test()
end
```
上面定义了一个Car类，我们可以使用@type Car来表明某个变量属于Car类

## 变量的声明
使用`@type`可以指明某个变量的类型
```
---@type Car
local car1 = getCar()
---@type Car
global_car = getCar()
global_car:test()
```
其格式如下：
```
---@type my_type[|other_type] [@comment]
```
举个例子：
```
---@type Car @instance of car
local car1 = {}

---@type Car|Ship @transport tools, car or ship. Since lua is dynamic-typed, a variable may be of different types
---use | to list all possible types
local transport = {}

---@type Car @global variable type
global_car = {}

local obj = {}
---@type Car @property type
obj.car = getCar()
```

## 别名注释
使用`@alias`将一些复杂不容易输入的类型注册为新的容易输入的类型
其格式如下：
```
---@alias new_name type
```
举个例子：
```
---@alias Handler fun( type: string, data: any):void
---@param handler Handler
function addHandler(handler)
end
```

## 参数说明
使用`@param`来指定函数的参数类型
其格式如下：
```
---@param param_name my_type[|other_type] [@comment]
```
举个例子：
```
---@param car Car
local function setCar(car)
end

---@param car Car
setCallback(function(car)
end)

---@param car Car
for k, car in pairs(list) do
end
```

## 函数返回值类型说明
其格式如下：
```
---@return my_type[|other_type] [@comment]
```
举个例子：
```
---@return Car|Ship
local function crate()
end

---here car_or_ship doesn't need @type annotation, EmmyLua has already inferred the type via "create" function
local car_or_ship = create()

---@return Car
function factory:create()
end
```

## 属性声明
使用`@field`可以标明类所具有的属性，即使类中没有使用到这个属性
其格式如下：
```
---@field [public|protected|private] field_name field_type[|other_type] [@comment]
```
举个例子：
```
---@class Car
---@field public name string @add name field to class Car, you'll see it in code completion
local cls = class()
```

## 泛型声明
使用`@generic`来模拟一些高级语言的泛型
其格式如下：
```
---@generic T1 [: parent_type] [, T2 [: parent_type]]
```
举个例子：
```
---@generic T : Transport, K
---@param param1 T
---@param param2 K
---@return T
local function test(param1, param2)
	-- todo
end

---@type Car
local car = ...
local value = test(car)
```
这里说明一下，上面的`T`和`K`都是类型，只不过`T`是继承自`Transport`的一种类型，而`K`可以是任何类型。
然后`param1`是类型`T`的实例

## 不定参数的注释
使用`@vararg`注解函数不定参数部分
其格式如下：
```
---@vararg type
```
举个例子：
```
---@vararg string
---@return string
local function format(...)
	local tbl = {...} --inferred as string[]
end
```

## 嵌入其他语言
使用`@language`可以在lua代码中嵌入其他语言
其格式如下：
```
---@language language_id
```
举个例子：
```
---@language JSON
local jsonText = [[{
	"name":"Emmy"
	}]]
```

## 数组类型声明
使用`@type mytype[]`可以指定变量类型为数组
其格式如下：
```
---@type my_type[]
```
举个例子：
```
---@type Car[]
local list = {}
local car = list[1]
-- car, and you 'll see completion

for i, car in pairs(list) do
	-- car. and you'll see completion
end
```

## table类型声明
使用`@type table`可以表示某变量为表格类型
其格式如下：
```
---@type table<key_type, value_type>
```
举个例子：
```
---@type table<string, Car>
local dict = {}
local car = dict['key']
-- car. and you'll see completion
for key, car in pairs(dict) do
	-- car. and you'll see completion
end
```

## 函数声明
使用`fun(param:my_type):return_type`来表示函数类型变量
其格式如下：
```
---@type fun(param:my_type):return_type
```
举个例子：
```
@type fun(key:string):Car
local carCreatorFn1

local car = carCreatorFn1('key')
-- car. and you see code completion

---@type fun():Car[]
local carCreatorFn2

for i, car in pairs(carCreatorFn2()) do
	-- car. and you see completion
end
```

## 字面常量类型声明
字面常量也可以指定特定字符串来作为代码提示，结合别名，可以起到类似于枚举的效果
举个例子：
```
---@alias Handler fun(type: string, data: any): void

---@param event string | "'onClised'" | "'onData'"
---@param handler Handler | "function(type, data) print(data) end"
function addEventListener(event, handler)
end
```
上面的写法还是比较复杂，我们可以用`@alias`再简化一下
```
---@alias Handler fun(type: string, data: any): void
---@alias IOEventEnum string | "'onClised'" | "'onData'"
---@param event IOEventEnum
---@param handler Handler | "function(type, data) print(data) end"
function addEventListener(event, handler)
end
```



