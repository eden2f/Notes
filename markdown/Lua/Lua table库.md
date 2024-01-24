# Lua table库

## table常用方法

1. insert t={10,20,30} table.insert(t,1,15) --> t = { 15 , 10 , 20 , 30 }
  * table.insert(t,1,15) 第二个参数 位置 第三个参数 值
  * table.insert(t,15) 默认插入到末尾
2. remove(t,2) 删除 位置为 2 的值
3. sort 排序
4. concat : 连接

## 示例代码

``` Lua
t = {}
table.insert( t , 1 , 1 )
print("#t",#t)
table.remove( t , 1 )
print("#t",#t)


lines = {
luaH_set = 10 ,
luaH_het = 24 ,
luaH_present = 48
}

a = {}
for n in pairs( lines ) do
a[#a+1] = n
end
table.sort(a)
for i,n in ipairs(a) do
print(" a : " , n)
end

function pairsByKeys( t , f )
local a = {}
for n in pairs(t) do a[#a+1] = n end
table.sort( a , f )
local i = 0  -- 迭代器变量
return function()  -- 迭代器函数
    i = i + 1
    return a[i],t[a[i]]
end
end

for name, line in pairsByKeys(lines) do
print( "pairsByKeys : ", name , line)
end


function rconcat( l )
if type(l) ~= "table" then
    return l
end
local res = {}
for i = 1 , #l do
    res[i] = rconcat( l[i] )
end
return table.concat(res)
end

print("concat(lines) : " , rconcat( {{" a ",{" nice "}}, {{" long "},{" list "}}} )
```