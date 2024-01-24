# Lua 对象默认属性

* 属性数量少,用第一种方式实现
* 属性数量多,用第二种方式实现

## 方式一 

``` Lua
local defualts = {}
setmetatable( defaults , { __mode = "k" } )
local mt = { __index = function(t) return deaults[t] end}
function setDefault( t , d )
defaults[t] = d
setmetatable( t , mt )
end
```

## 方式二

``` Lua
local metas = {}
setmetatable( metas , { __mode = "v" } )
function setDefault( t , d )
local mt = metas[d]
if mt == nil then
    mt = { __index = function() return d end}
    metas[d] = mt    -- 备忘录
end
setmetatable(t,mt)
end
```