# Lua table的元方法

## __index元方法 (用于检索)

``` Lua
window = {}  -- 创建一个名称空间

-- 使用默认值来创建一个原型
window.prototype = { x = 0 , y = 0 , width = 100 , height = 100 }
window.mt = {}  -- 创建元表
-- 声明构造函数
function window.new(o)
setmetatable( o , window.mt )
return o
end

-- window.mt.__index = function( table , key )
-- 	return window.prototype[key]
-- end

window.mt.__index = window.prototype

w = window.new{ x = 10 , y = 20 }
print(w.width)
print(w.x)
print(w.xxxx)
```

## __newindex 元方法 (用于更新)

``` Lua 
local key = {}
local mt = { __index = function(t)  return t[key] end }
function setDefault( t , d )
t[key] = d
setmetatable( t , mt )
end
tab = { x = 10 , y = 20 }
print( tab.x , tab.z )
setDefault( tab , 100 )
print( tab.x , tab.z )
--]]

--[[
-- 跟踪 table 的访问
t = {}  -- 原来的table(在其他地方创建的)
-- 保持对原table的私有的访问
local _t = t

-- 创建一个代理
t = {}

-- 创建一个元表
local mt = {
__index = function( t , k )
    print("*access to element " .. tostring(k) )
    return _t[k]  -- 访问原来的table
end ,
__newindex = function(t,k,v)
    print("*udate of element " .. tostring(k) .. " to " .. tostring(v) )
    _t[k] = v
end
}

setmetatable( t , mt )

t[2] = "Hello"
print(t[2])
```

## 定义一个只读的 table

``` Lua
function readonly(t)
local proxy = {}
local mt = {
    __index = t ,
    __newindex = function( t , k , v)
        error("attempt to update a read-only table" , 2 )
    end
}
setmetatable(proxy , mt)
return proxy
end

days = readonly{"Sunday" , "Monday" , "Tuesday" , "Wednesday" , "Thursday" , "Friday" , "Sataurday" }

print(days[1])
```