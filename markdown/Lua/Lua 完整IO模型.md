# Lua 完整IO模型

## io.open 函数

io.open( filename , mode )  :mode="r" "w" "a"表示追加 "b"表示二进制

## 性能小技巧

local lines,rest = f:read( BUFSIZE , "*line" )

## 二进制文件
## 其他文件的操作

tmpfile
flush  io.flush()   f:flush()
seek   f:seek(whence , offset)    :whence="set";"cur";"end"

## 示例

``` Lua
--[[

print( io.open("e:\\demo51.lua" , "r") )

local f = assert( io.open( "e:\\demo51.lua" , "r" ) )
local t = f:read("*all")
print(t)
--  f:clone()

-- 临时改变当前的输入文件
local temp = io.input()  -- 保存当前文件
io.input( "e:\\demo51_2.lua" )  -- 打开一个新的当前文件
-- <对新的输入文件做一些操作>
local t = io.read("*all")
print(t)
io.input():close()  -- 关闭当前文件
io.input(temp)

--]]



--[[
-- 用于统计文件中的字符数\单词数\和行数的程序
local BUFSIZE = 2^13  -- 8k
local f = io.input( "e:\\demo51_2.lua" )  -- 打开输入文件中
local cc,lc,wc = 0 , 0 , 0   -- 字符数\行\单词的数量
while true do
	local lines , rest = f:read( BUFSIZE , "*line")
	if not lines then break end
	cc = cc + #lines  -- 字符的数量
	-- 统计块中的单词数
	local _,t = string.gsub(lines , "%S+" , "")
	wc = wc + t
	-- 统计块中换行字符的数量
	_,t = string.gsub(lines , "\n" , "\n" )
	lc = lc + t
end
print( lc , wc , cc )
--]]


--将DOS格式的文本文件转换为UNIX格式的程序 prog.lua
local inp = assert( io.open(arg[1],"rb"))
local out =assert( io.open(arg[2],"wb"))
local data = inp:read("*all")
data = string.gsub( data , "\r\n" , "\n" )
out:write(data)
assert(out:close())

-- 在命令行中调用这个程序:>lua prog.lua file.dox  file.unix


--打印在一个二进制文件中的找到的
local f = assert( io.open(arg[1]) , "rd"))
local data = f:read("*all")
local validchars = "[%w%p%s]"
local pattern = string.rep( validchars , 6 ) .. "+%z"
for w in string.gmatch( data , pattern ) do
	print(w)
end

-- 在命令行中调用这个程序 : > lua printChar.lua  file.bin
```