# Lua 运行环境变量

``` Lua 
-- 改变当前的环境 方法一
--[[ 
a = 1  -- 创建一个全局变量
setfenv( 1, { g = _G }) -- 使用 setfenv 改变当前的环境
g.print( a )
g.print( g.a )
--]]

-- 改变当前的环境 方法二
--[[
a = 1
local newgt = {}  -- 创建新环境
setmetatable( newgt , { __index = _G } )
setfenv( 1 , newgt )  -- 设置它
print(a)
a = 10
print(a)
print(_G.a)
_G.a = 20
print(_G.a)
--]]

function factory()
	return function()
		return a -- "全局的" a
	end
end

a = 3
f1 = factory()
f2 = factory()
print(f1())
print(f2())

setfenv( f1 , { a = 10 })

print(f1())
print(f2())

```