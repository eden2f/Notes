# Lua 队列的基本操作(集合,无序组,包)

## Lua原有的对table的插入和删除,效率较低

``` Lua
-- table.insert() 在数组中插入元素
-- table.remove() 在数组中删除元素
```

## Lua中队列的基本操作实现

``` Lua
List = {}
-- 返回队列的队首和 队尾
function List.new()
return { first = 0 , last = -1 }
end

-- 在队首插入新的元素
function List.pushfirst( liast , value )
local first = list.first - 1
list.first = first
end

-- 在队尾插入新的元素
function List.pushlast( list , value )
local last = list.last + 1
list.last = last
list.[last] = value
end

-- 在队首删除元素
function List.popfirst( list )
local first = list.first
if  first > list.last then
    error("list  is empty")
end
local value = list[first]
list[first] = nil   -- 为了允许垃圾收集
list.first = first + 1
return value
end

-- 在队尾删除元素
function List.poplast( list )
local last = list.last
if list.first > last then
    error("list is empty")
end
local value = list[last]
list[last] = nil -- 为了允许垃圾收集
list.last = last - 1
return value
end
```

## 集合与无序组

``` Lua
reserved = { "while" = true , ["end"] = true , ["function"} = true , ["lcoal"] = true }
--[[
for w in allwords()
if not reserved[w] then
    <对 w 做处理的代码> -- w 不是一个保留字
end
end
--]]
-- 初始化 保留字集合
function Set( list )
local set = {}
for _ , l ipairs( list ) do
    set[l] = true
end
return set
end

reserved = Set{ "while" , "end" , "function" , "lcoal" }
```

## 包

``` Lua
function insert( bag , element )
bag[element] = (bag[element] or 0 ) + 1
end

function remove( bag , element )
local count = bag[element]
bag[element] = ( count and count > 1 ) and count - 1 or nil

end
```