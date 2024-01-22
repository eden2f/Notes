# Lua 用协同程序实现迭代器

## 需求 : 对 {"a","b","c"} 所有的排列组合情况输出

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240123002618.png)


## 普通实现（没有使用协同程序）

``` lua 
function permgen( a , n ) -- a 是要进行排列组合的数字,n a的长度
n = n or #a  -- 默认n为a的大小
if n<=1 then
    printResult(a)
else
    for i=1 , n do
        a[n] , a[i] = a[i] , a[n]  -- 将第i个元素放到数组的末尾
        permgen( a , n-1 )  -- 生成其余的元素的排列
        a[n] , a[i] = a[i] , a[n]  -- 恢复第i个元素
    end
end
end

function printResult(a)
for i = 1 , #a do
    io.write(a[i] , " ")
end
io.write("\n")
end

permgen({1,2,3,4})
```

## 用协同程序实现迭代器

``` lua
function permgen( a , n ) -- a 是要进行排列组合的数字,n a的长度
n = n or #a  -- 默认n为a的大小
if n<=1 then
    coroutine.yield(a)  -- printResult(a)
else
    for i=1 , n do
        a[n] , a[i] = a[i] , a[n]  -- 将第i个元素放到数组的末尾
        permgen( a , n-1 )  -- 生成其余的元素的排列
        a[n] , a[i] = a[i] , a[n]  -- 恢复第i个元素
    end
end
end

function printResult(a)
for i = 1 , #a do
    io.write(a[i] , " ")
end
io.write("\n")
end

function permutations(a)
local co = coroutine.create(
    function() permgen(a) end
)
return function() -- 迭代器
    local code , res = coroutine.resume(co)
    return res
end
end

for p in permutations({"a","b","c"}) do
printResult(p)
end
```

## wrap 函数

``` lua
function permgen( a , n ) -- a 是要进行排列组合的数字,n a的长度
n = n or #a  -- 默认n为a的大小
if n<=1 then
    coroutine.yield(a)  -- printResult(a)
else
    for i=1 , n do
        a[n] , a[i] = a[i] , a[n]  -- 将第i个元素放到数组的末尾
        permgen( a , n-1 )  -- 生成其余的元素的排列
        a[n] , a[i] = a[i] , a[n]  -- 恢复第i个元素
    end
end
end

function printResult(a)
for i = 1 , #a do
    io.write(a[i] , " ")
end
io.write("\n")
end

function permutations(a)
return coroutine.wrap( function() permgen(a) end )
end

for p in permutations({"a","b","c"}) do
printResult(p)
end
```