# Lua 序列化(Serialization)

``` Lua
-- "varname=<exp>"

--varname = "]] .. os.execute(' rm I*) .. [["

function serialize(o)
	if type(o) == "number" then
		io.write(o)
	elseif type(0) == "string" then
		-- io.write( "'" , o , "'" )
		io.write( "[[" , o , "]]" )
	end
end

-- 得到 varname = "[[]] .. os.execute(' rm I*) .. [[]]"


--1 用 format( "%q" , s ) 解决上面存在的问题
a  = 'a "rblematic" \\string'
print( string.format("%q",a)) --得到结果 "a \"rblematic\" \\string"


--2 [===[ string ]===] 注意:两端等号数量相等 , 等号数量要比字符串中的连续等号大
print([===[ [[]] .. os.==execute(' rm I*) .. [[]] ]===])


function quote(str)
	-- 查找最长的等号序列
	local n = -1
	for w in string.gmatch( str , "]=*") do  -- ]=* 这是正则表达式
		n = match.max(n,#w-1)
	end

	-- 产生 n + 1 个等号
	local eq = string.rep( "=" , n + 1 )
	-- 生成长字符串的字面表示
	return string.format(" [%s[\n%s]%s] ", eq , str , eq )
end

print( "quote : " , quote(a) )

```