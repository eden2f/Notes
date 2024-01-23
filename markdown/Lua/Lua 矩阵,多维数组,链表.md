# Lua 矩阵,多维数组,链表

``` Lua

-- 矩阵与多维数组
-- 链表

a = {}
for i = 1 , 1000 do
	a[i] = 0
end

print("数组的大小 : " .. #a)
print("a[500] : "  , a[500])
print("a[2000] : " , a[2000])

squares = { 1, 4, 9 , 16 , 25 , 36 , 49 , 64 }

-- 使用数组的数组来创建一个 n * m 的矩阵
N = 5
M = 3
mt = {} -- 创建矩阵
for i = 1 , N do
	for j = 1 , i do  -- 三角形矩阵
		mt[i][j] = 0
	end
	mt[i] = {} -- 创建一个新行
	for j = 1 , M do  -- 长方形矩阵
		mt[i][j] = 0
	end
end

-- 将两个索引合并为一个索引的方式创建一个 n * m 的矩阵
mt = {}  -- 创建矩阵
for i = 1 , N do
	for j = 1 , M do
		mt[ (i-1) * M + j ] = 0
	end
end

mt[ s .. ":" .. t ]

--  稀疏矩阵
--  表示一个图(graph)
-- m,n x     如果 x != nil , 则表示点m和点n 是相连的 , 权值为 x ; 如果 x == nil ,则不相连

function mult( a , rowindex , k )
	local row = a[rowindex]
	for ii , v in pairs(row) do
		row[i] = v * k
	end
end

-- 链表
list = nil
list = { next = list , value = v }
local l = list
while l do
	print(l.value)
	l = l.next
end

```