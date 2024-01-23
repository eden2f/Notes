# Lua 弱引用

## table中会被回收的的对象

1. 弱引用的key  
2. 具有弱引用value
3. 具有弱引用key和value

## 弱引用类型

一个table的弱引用类型是通过其元素表中的__mode字段来决定的，这个字段的值应为一个字符串，如果这个字符串中包含字母'k'/'v'那么这个table 的value是弱引用

## 注意

Lua只会回收弱引用对象，而像数字和bool这样的值(常量)，是不会被回收的

## 实例代码

``` Lua
-- __mode 字段 'k' 'v'
a = {}
b = { __mode = "k" }
setmetatable( a , b )    -- 现在 'a' 的 key 就是弱应用
key = {}   -- 创建第一个key
a[key] = 1
key = {}  -- 创建第二个key
a[key] = 2
collectgarbage()  -- 强制进行一次垃圾收集 第一个key被回收了
for k , v in pairs(a) do
print(v)
end
```

## 备忘录(memorize)函数

``` Lua
local results = {}
function mem_loadstring(s)
local res = results[s]
if res == nil then  -- 是否已记录过
    res = assert( loadstring(s) )  -- 计算新结果
    results[s] = res  -- 存下以备之后复用
end
return res
end

function createRGB( r , g , b )
return { red = r , green = g , blue = b }
end

local results = {}
setmetatable( results , { __mode = "v" })    -- 使value成为弱应用
function createRGB(r , g , b)
local key = r .."-" .. g .. "-" .. b
local color = results[key]
if color == nil then
    color = { red = r , green = g , blue = b }
    results[key] = color
end
return color
end
```