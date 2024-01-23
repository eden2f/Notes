# Lua 模块的定义和实现

``` Lua
local modname = "..."

local M = {}
_G[modname] = M
package.loaded[modname] = M
-- complex = M  -- 模块名

function M.new( r , i )
	return { r = r , i = i }
end

M.i = M.new( 0 , 1 ) -- 定义一个常量 'i'

-- 加法
function M.add( c1 , c2 )
	return M.new( c1.r + c2.r , c1.i + c2.i)
end

-- 减法
function M.sub( c1 , c2 )
	return M.new( c1.r - c2.r , c1.i - c2.i )
end

-- 乘法
function M.mul( c1 , c2 )
	return M.new( c1.r * c2.r - c1.i * c2.i , c1.r * c2.i + c1.i * c2.r )
end

local function inv( c )
	local n = c.r^2 + c.i^2
	return M.new( c.r/n , c.i/n )
end

-- 除法
function M.div( c1 , c2 )
	return M.mul( c1 , inv(c2) )
end

-- return M

```