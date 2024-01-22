# Lua 文件读取和写入

``` Lua
--[[
-- C:\\Users\\eden\\Desktop\\LuaLearning\\demo\\data 中的内容
Entry{
	author = "eden1" ,
	title = "bookname1" ,
	publishing = "publishing company1" ,
	year = "1997"
}

Entry{
	author = "eden2" ,
	title = "bookname2" ,
	publishing = "publishing company2" ,
	year = "1997"
}

Entry{
	author = "eden3" ,
	title = "bookname3" ,
	publishing = "publishing company3" ,
	year = "1997"
}

Entry{
	author = "eden4" ,
	title = "bookname4" ,
	publishing = "publishing company4" ,
	year = "1997"
}
--]]

-- Entry{ <code> } 与 Entry({<code>}) 完全相同的

do
local count = 0
	function Entry(...) count = count + 1 end

	dofile("C:\\Users\\huangMP.DESKTOP-88ENV6P\\Desktop\\LuaLearning\\demo\\data")

	print("number of entries : " .. count)

	local authors = {}   -- 作者姓名集合
	function Entry(b) authors[b[1]] = true end
	dofile("C:\\Users\\DESKTOP-88ENV6P\\Desktop\\LuaLearning\\demo\\data")
	for name in pairs(authors) do
		print(name)
	end

end

-- 采用键值对的方式保存

--[[
-- C:\\Users\\DESKTOP-88ENV6P\\Desktop\\LuaLearning\\demo\\data2 中的内容
Entry{
	author = "eden1" ,
	title = "bookname1" ,
	publishing = "publishing company1" ,
	year = "1997"
}

Entry{
	author = "eden2" ,
	title = "bookname2" ,
	publishing = "publishing company2" ,
	year = "1997"
}

Entry{
	author = "eden3" ,
	title = "bookname3" ,
	publishing = "publishing company3" ,
	year = "1997"
}

Entry{
	author = "eden4" ,
	title = "bookname4" ,
	publishing = "publishing company4" ,
	year = "1997"
}
--]]

do
	local authors = {}   -- 作者姓名集合
	function Entry(b) authors[b.author] = true end
	dofile("C:\\Users\\DESKTOP-88ENV6P\\Desktop\\LuaLearning\\demo\\data2")
	for name in pairs(authors) do
		print(name)
	end
end
```