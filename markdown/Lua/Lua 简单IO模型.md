# Lua 简单IO模型


## io.input(filename)
## io.output()
## io.write

print() 与 io.write
print 中 "," 会转为 制表符 ,会自动在末尾加 换行符

## io.read

"*all" 读取整个文件;
"*line"读取下一行;
"*number"读取一个数字;
<num>读取一个不超过<num>个字符的字符串
MIME quoted-printable 编码方式 :  会将 acs 码值 转化为 =<xx>

## io.lines

## 示例

``` Lua
io.write( "sin(3) = " , math.sin(3) , "\n") --> sin(3) = 0.14112000805987
io.write( string.format("sin(3)=%.4f\n", math.sin(3) ) )  --> sin(3)=0.1411

-- io.write(a..b..c)  io.write(a,b,c)

print("hello" , "Lua" );print("Hi")
io.write("hello" , "Lua" );io.write("Hi")


-- t = io.read("*all")  -- 读取整个文件
-- t = string.gsub( t , ... )  -- 做相关的处理
-- io.write(t)  -- 写输出
--[[
t = io.read("*all")
t = string.gsub( t , "(\128-\255=])" , function(c)
		return string.format( "=%2x" , string.byte(c) )
end)
io.write(t)
--]]

--[[
for count = 1 , math.huge do
	local line = io.read()
	if line == nil then break end
	io.write( string.format( "%6d  " , count ) , line , "\n" )
end
--]]

--[[
local lines ={}
-- 读取table 'lines' 中的所有行
for line in io.lines() do lines[#lines+1] = line end
table.sort(lines)
-- 输出所有行
for _,l in ipairs(lines) do io.write( l , "\n" ) end
--]]

--[[
while true do
	local n1 , n2 , n3 = io.read("*number" , "*number" , "*number")
	if not n1 then break end
	print("最大的数:" , math.max(n1 , n2 , n3 ))
end
--]]

while true do
	local block = io.read( 2^13 )  -- 缓冲大小为8k
	if not block then break end
	io.write(block)
end

io.read( 0 )  -- 用于检查是否到达文件末尾 ,
				-- 如果还有数据可以读取的话,会返回 空字符串
				-- 否则返回nil
```