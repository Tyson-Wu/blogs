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
```lua
--- my supper class
---@class Transport
local p = Class()

--- 子类
---@class Car : Transport
local cls = Class(p)
function c:test()
end
```lua
其格式如下：
```lua
--@class my_type[: parent_type] [@comment]
```lua
举个例子：
```lua
---@class Car : Transport @define class Car extends Transport
local cls = class()
function clas:test()
end
```lua
上面定义了一个Car类，我们可以使用@type Car来表明某个变量属于Car类

## 变量的声明
使用`@type`可以指明某个变量的类型
```lua
---@type Car
local car1 = getCar()
---@type Car
global_car = getCar()
global_car:test()
```lua
其格式如下：
```lua
---@type my_type[|other_type] [@comment]
```lua
举个例子：
```lua
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
```lua

## 别名注释
使用`@alias`将一些复杂不容易输入的类型注册为新的容易输入的类型
其格式如下：
```lua
---@alias new_name type
```lua
举个例子：
```lua
---@alias Handler fun( type: string, data: any):void
---@param handler Handler
function addHandler(handler)
end
```lua

## 参数说明
使用`@param`来指定函数的参数类型
其格式如下：
```lua
---@param param_name my_type[|other_type] [@comment]
```lua
举个例子：
```lua
---@param car Car
local function setCar(car)
end

---@param car Car
setCallback(function(car)
end)

---@param car Car
for k, car in pairs(list) do
end
```lua

## 函数返回值类型说明
其格式如下：
```lua
---@return my_type[|other_type] [@comment]
```lua
举个例子：
```lua
---@return Car|Ship
local function crate()
end

---here car_or_ship doesn't need @type annotation, EmmyLua has already inferred the type via "create" function
local car_or_ship = create()

---@return Car
function factory:create()
end
```lua

## 属性声明
使用`@field`可以标明类所具有的属性，即使类中没有使用到这个属性
其格式如下：
```lua
---@field [public|protected|private] field_name field_type[|other_type] [@comment]
```lua
举个例子：
```lua
---@class Car
---@field public name string @add name field to class Car, you'll see it in code completion
local cls = class()
```lua

## 泛型声明
使用`@generic`来模拟一些高级语言的泛型
其格式如下：
```lua
---@generic T1 [: parent_type] [, T2 [: parent_type]]
```lua
举个例子：
```lua
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
```lua
这里说明一下，上面的`T`和`K`都是类型，只不过`T`是继承自`Transport`的一种类型，而`K`可以是任何类型。
然后`param1`是类型`T`的实例

## 不定参数的注释
使用`@vararg`注解函数不定参数部分
其格式如下：
```lua
---@vararg type
```lua
举个例子：
```lua
---@vararg string
---@return string
local function format(...)
	local tbl = {...} --inferred as string[]
end
```lua

## 嵌入其他语言
使用`@language`可以在lua代码中嵌入其他语言
其格式如下：
```lua
---@language language_id
```lua
举个例子：
```lua
---@language JSON
local jsonText = [[{
	"name":"Emmy"
	}]]
```lua

## 数组类型声明
使用`@type mytype[]`可以指定变量类型为数组
其格式如下：
```lua
---@type my_type[]
```lua
举个例子：
```lua
---@type Car[]
local list = {}
local car = list[1]
-- car, and you 'll see completion

for i, car in pairs(list) do
	-- car. and you'll see completion
end
```lua

## table类型声明
使用`@type table`可以表示某变量为表格类型
其格式如下：
```lua
---@type table<key_type, value_type>
```lua
举个例子：
```lua
---@type table<string, Car>
local dict = {}
local car = dict['key']
-- car. and you'll see completion
for key, car in pairs(dict) do
	-- car. and you'll see completion
end
```lua

## 函数声明
使用`fun(param:my_type):return_type`来表示函数类型变量
其格式如下：
```lua
---@type fun(param:my_type):return_type
```lua
举个例子：
```lua
@type fun(key:string):Car
local carCreatorFn1

local car = carCreatorFn1('key')
-- car. and you see code completion

---@type fun():Car[]
local carCreatorFn2

for i, car in pairs(carCreatorFn2()) do
	-- car. and you see completion
end
```lua

## 字面常量类型声明
字面常量也可以指定特定字符串来作为代码提示，结合别名，可以起到类似于枚举的效果
举个例子：
```lua
---@alias Handler fun(type: string, data: any): void

---@param event string | "'onClised'" | "'onData'"
---@param handler Handler | "function(type, data) print(data) end"
function addEventListener(event, handler)
end
```lua
上面的写法还是比较复杂，我们可以用`@alias`再简化一下
```lua
---@alias Handler fun(type: string, data: any): void
---@alias IOEventEnum string | "'onClised'" | "'onData'"
---@param event IOEventEnum
---@param handler Handler | "function(type, data) print(data) end"
function addEventListener(event, handler)
end
```lua



