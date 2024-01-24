# Lua 依赖引入(require函数)

* 模块\包
* require函数

``` Lua 
-- module
-- require "mod"
-- local f = mod.foo

local m = require "io"
m.write("Hello")

-- loadfile
-- loadlid 加载c文件

--[[
-- require 方法源码
function require(name)
	if not package.loaded[name] then  -- 模块是否已加载
		local loaser = findloader(name)  --
		if loaser == nil then
			error("unable to load module " .. name )
		end
		package.loade[name] = true  -- 将模块标记为已加载
		local res = loader(name)
		if res ~= nil then
			package.loaded[name] = res
		end
	end
	return package.loaded[name]
end
--]]

require("foo")
-- 如果需要多次加载同一个文件
package.loaded["foo"] = nil
require("foo")
```