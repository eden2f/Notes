# Lua 全局环境

``` Lua
--[[
-- 全局变量保存在 _G 中
-- 遍历当前的全局变量
for n in pairs (_G) do
	print(n)
end

-- 新增全局变量
huangMP = "huangMP"
for n in pairs (_G) do
	print(n)
end

--]]

--[[
-- 具有动态名字的全局变量
-- value = loadstring( "return " .. varname )
-- value = _G(varname)
-- _G(varname) = value

function getfield(f)
	local v = _G  -- 从全局变量的table开始
	for w in string.gmatch( f , "[%w_]+") do
		v = v[w]
	end
	return v
end

-- a.b.c.d = v  --> local temp = a.b.c; temp.d = v

function setfield( f, v )
	local t = _G  -- 从全局变量的table开始
	for w,d in string.gmatch( f , "([%w_]+)(%.?)" ) do
		if d == "." then  -- 是最后一个字段吗
			t[w] = t[w] or {}  -- 如果不存在就创建 table
			t = t[w]  -- 获取该table
		else
			t[w] = v
		end
	end
end

setfield("t.x.y" , 10)
print(t.x.y)

--]]

setmetatable( _G , {
	__newindex = function(t, n , v )
		local w = debug.getinfo(2,"S").what
		if w~="main" and w~="C" then  -- 是否是主程序 或者 是否是c程序
			error("attempt to write to undeclared vaiable " .. n , 2 )
		end
		rawset(t,n,v)
	end ,
	__index = function( _ , n )
		error("attemp to read undeclared variable " .. n , 2 )
	end
})
-- prinf(a)  -- attemp to read undeclared variable prinf
a = "HK"
print(a)
print( _G["a"])

-- 声明全局变量 1
function declare( name , initval )
	rawset( _G , name , intival or false )
end

-- 声明全局变量 2
a = 10 -- 这种方法要在主程序中才有效

-- debug.getinfo(2,"S").what 可查看当前是否是主程序块

function f(a)
	-- b = a
	return a
end

s = f("OK")
print(s)
```