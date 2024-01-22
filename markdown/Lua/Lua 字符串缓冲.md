# Lua 字符串缓冲

## 问题

代码中str2被创建后,会从str1字符串中把所有字符复制到str2,再添加"wwww"字符串,效率不高

``` Lua
str1 = "xxxxxxxx"
str2 = str1 .. "wwww"
```

## 解决方法 : 字符串缓冲

``` Lua 
local buff = ""
for line in io.lines() do
	buff = buff .. line .. "\n"
end

-- buff=20(Bytes/L) * 2500 (L) =5000(B) 50k
-- buff .. 20Byte --> 50020B

io.read("*all")

local t = {}
for line in io.lines() do
	t[#t + 1] = line
end
t[ #t + 1 ] = ""
local s = table.concat( t , "\n" )

-- 采用二分算法 连接字符串
function addstring( stack , s )
	stack[#stack + 1 ] = s  -- 将s压入栈
	for i = #stack - 1 , 1 , -1 do
		if #stack[i] > #stack[i+1] then
			break
		end
		stack[i] = stack[i] .. stack[i+1]
		stack[ i + 1 ] = nil
	end
end
```