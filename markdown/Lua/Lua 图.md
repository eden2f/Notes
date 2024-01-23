# Lua 图

``` Lua
-- table 的字段 : name (结点的名称) , adj (与该结点邻接的结点集合)

function name2node( graph , name )  -- 将名称对应到结点邻
	if not graph[name] then  -- 结点不存在,创建新的结点
		graph[name] = { name = name , adj = {} }
		print(#graph)
	end
	return graph[name]
end

-- 构造一个图
function readgraph()
	local graph = {}
	for line in io.lines() do
		local namefrom , nameto  = string.match(line, "(%S+)%s+(%S+)" )  -- 切分行中的两个名称
		-- 查找相应的结点
		local from = name2node( graph , namefrom )
		local to = name2node( graph , nameto )
		from.adj[to] = true   -- 将 to 添加到 from 的邻接集合
	end
	return graph
end

-- 采用深度优先遍历图
function findpath( curr , to , path , visited )
	path = path or {}
	visited = visited or {}25
	if visited[curr] then  -- 节点是否已经访问过了
		return nil  -- 这里没有路径
	end
	visited[curr] = true  -- 将结点标记为已访问过了
	path[ #path + 1 ] = curr  -- 将其加到路径中
	if curr == to then   -- 最后的结点吗?
		return path
	end
	for node in pairs( curr.adj ) do
		local p = findpath(node , to , path , visited)
		if p then return p end
		path [#path] = nil  -- 从路径中删除结点
	end
end

function printpath( path )  -- 打印路径
	for i = 1 , #path do
		print(path[i].name)
	end
end

g = readgraph()
a = name2node( g , "a" )
b = name2node( g , "b" )
p = findpath( a , b )
if p then printpath(p) end
```