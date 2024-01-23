# Lua 关系类的元方法

* __eq() 等于
* __lt() 小于
* __le() 包含 

``` Lua
mt = {}
mt.__le = function( a , b ) -- 集合包含 a<=b
	for k in pairs(a) do
		if not b[k] then
			return false
		end
	end
	return true
end

mt.__lt = function(a , b )  -- a<b
	return a <= b and not(b<=a)
end

mt.__eq = function( a , b )
	return a<=b and b<=a
end

mt.__tostring = function( set )
	local l = {} -- 用于存放集合中所有元素的列表
	for e in pairs(set) do
		l[#l + 1 ] = set[e]
	end
	return "{" .. table.concat( l , ",") .. "}"
end

mt.__metatable = "not your business"

s1 = { 2 , 4 }
s2 = { 4 , 10 , 2 }
setmetatable( s1 , mt )
setmetatable( s2 , mt )
print( s1 <= s2 )  --> true
print( s1 < s2 )  --> true
print( s2 < s1 )  --> false
print( s2 == s1 )  --> false
print( s1 == s1 )  --> true
print( s1 )  --> {2,4}
```