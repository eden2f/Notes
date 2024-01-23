# Lua table的序列化

## 无环的 table 的保存

``` Lua
-- 将对象序列化
function serialize( o )
if type(o) == "number" then  -- 是否是数字
    io.write(o)
elseif type(o) == "string" then  -- 是否是 字符串
    io.write(string.format( "%q", o ))  -- %q 接受一个字符串并将其转化为可安全被Lua编译器读入的格式
elseif type(o) == "table" then  -- 是否是 table
    io.write("{\n")
    for k,v in pairs(o) do
        -- io.write(" " ,  k , "=")
        io.write("[" ,  k , "]=")
        serialize( v )
        io.write(",\n")
    end
    io.write("}\n")
else
    error("cannot serialize a " .. type(o))
end
end

a = {a=12 , b = 'lua' , key = 'another "one"'}

serialize(a)

serialize{a=12 , ["if"] = 'lua' , key = 'another "one"'}
```

## 有环 table 的保存

``` Lua
-- 保存有环的table

function basicserialize(o)
if type(o) == "number" then
    return tostring(o)
else -- 假定是一个字符串
    return string.format("%q", o )
end
end

function save( name , value , saved )
saved = saved or {  }  -- 初始值
io.write( name , "=" )
if type(value) == "number" or type( value ) == "string" then
    io.write( basicserialize(value) , "\n" )
elseif type(value) == "table" then
    if saved[value] then -- 该value是否已保存过
        io.write(saved[value],"\n")  -- 使用先前的名字
    else
        saved[value] = name  -- 为下次使用保持名字
        io.write("{}\n")
        for k , v in pairs(value) do
            k = basicserialize( k )
            local fname =string.format( "%s[%s]" , name , k )
            save( fname , v , saved )
        end
    end
else
    error("cannot save a " .. type(value) )
end
end

--[[
a = { x = 1 , y = 2 , key = 128 }
save( "s" , a )
--]]

--[[
a = { x = 1 , y = 2 , { 3 ,4 ,5} }
a[2] = a   -- 环
a.z = a[1]  -- 共享子table
save( "s" , a )
--]]

a = {{ "one" , "two" } , 3 }
b = { k = a[1] }
local t = {}
save("a" , a , t )
```