# Lua 算数类的元方法

## 通过重写算数类的元方法，自定义该类的计算方式

``` Lua 
Set = {}

local mt = {}

-- 根据参数列表中的值创建一个新的集合
function Set.new(l)
	local set = {}
	setmetatable( set , mt )  -- 将mt设置为set的元表
	for _ , v in ipairs(l) do set[v] = true end
	return set
end

-- 返回两个table的并集
function Set.union( a , b)
	local res = Set.new{}
	for k in pairs(a) do res[k] = true end
	for k in pairs(b) do res[k] = true end
	return res
end

-- 返回两个table的交集
function Set.intersection( a , b )
	local res = Set.new{}
	for k in pairs(a) do
		res[k] = b[k]
	end
	return res
end

-- 返回集合的字符串形式
function Set.tostring( set )
	local l = {}  -- 用于存放集合中所有元素的列表
	for e in pairs(set) do
		l[#l + 1] = e
	end
	return  "{" .. table.concat( l , "," ) .. "}"
end

-- 打印集合
function Set.print(s)
	print(Set.tostring(s))
end

mt.__add = Set.union  -- 将 Set.union 方法加给元表的 __add字段 (加法)
mt.__mul = Set.intersection  -- Set.intersection 方法加给 mt 元表的 __mul字段(乘法)

s1 = Set.new{10 , 20 , 30 , 50 }
s2 = Set.new{30 ,1 }
s3 = s1+ s2
```