# Lua 跨模块通讯

将模块内的环境为内部环境，同时添加对外部环境的访问

## 方法一、二

``` Lua
local modname = ...
local M = {}
_G[modname] = M
package.loaded[modname] = M
-- setmetatable( M , { __index = _G} ) -- 实现对原来的 环境的访问 方法一
local _G = _G  -- 访问原来的全局变量需要加 _G    方法二  （效率比一快）
setfenv( 1 , M)

function add( c1 , c2 )
return new( c1.r + c2.r , c1.i + c2.i )  -- new 是同一模块中的函数
end
```

## 方法三 最正规的方式：需要用什么外部环境就引入什么

``` Lua 
-- 方法三 最正规的方法
local modname = ...
local M = {}
_G[modname] = M
packaged.loaded[modname] = M

-- 导入端：
-- 声明这个模块从外界所需的所有东西
local sqrt = math.sqrt
local io = io
```

## 方法四 ：使用 module 函数 （Lua 版本 5.1 以后）

``` Lua
-- module 函数 Lua 版本 5.1 以后才有的
-- 默认没有开放对外界的访问
module(...)

-- 实现对外部的访问
module( ... , package.seeall)  -- 效果相当 stemetatable( M , { __index = _G })

mod.sub
-- package.loaded / package.preload
```