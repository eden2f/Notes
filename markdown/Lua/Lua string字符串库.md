# Lua string字符串库

## 常用方法

1. len(s)  长度
2. rep(s,n)  s*n次  print( string.rep( "*" , 5 ) ) --> *****
3. lower(s)  将 s 字符串中的大写字母转为 小写
4. upper(s)  将 s 字符串中的小写字母转为 大写
5. sub(s,i,j)  复制   s 字符串 从第i位 到 第j位结束
6. char(n)  将 acs码转化为字符   print( string.char( 97 , 98 , 99 ) )  -->  abc
7. byte(s,i,j)  将 s(字符串)中从 第i个字符到第j个 转为acs码
  * print( string.byte( "sadfsadfsad"  ) )  --> 115
  * print( string.byte( "sadfsadfsad" , 1 ) )  --> 115
  * print( string.byte( "sadfsadfsad" , 1 , 3 ) )  -->  115	97	100
8. format  -- c语言 printf("%s",v)
  * print( string.format( "pi=%.4f" , math.pi ) )  -->  pi=3.1416

## 示例

``` Lua
print( string.rep( "*" , 5 ) )
print( string.lower( "KLJKiuiouioKLJK") )

a = { "ax" , "Dy" , "pk" , "bJlLk" }
table.sort(
	a ,
	function(a,b)
		return string.lower(a) < string.lower(b)
	end
)
print( table.concat(a,",") )

s = "[asdfjsalfksajflsa]"
print( string.sub( s , 2 , #s-1 ) )
print( string.char( 97 , 98 , 99 ) )
print( string.byte( "sadfsadfsad"  ) )
print( string.byte( "sadfsadfsad" , 1 ) )
print( string.byte( "sadfsadfsad" , 1 , 3 ) )

print( string.format( "pi=%.4f" , math.pi ) )

d = 5 ; m = 11 ; y = 1990

print( string.format("%02d/%02d/%04d",d,m,y) )  --> 05/11/1990

tag , title = "h1" , "a titile"
print(string.format("<%s>%s<%s>",tag,title,tag))  --> <h1>a titile<h1>
```
1. 模式匹配函数
  * string.find
  * string.match
  * string.gsub
  * string.gmatch

示例代码 : 
``` Lua
s = "hello world"
print( string.find(s,"hello") ) --> 1	5
print( string.find(s,"world") ) --> 7	11
print( string.find(s,"l" ) )  --> 3	3
print( string.find(s,"l",4 ) )  --> 4	4
print( string.find(s,"lasfsaf") )  --> nil

local t = {}  -- 存储索引的table
local i = 0
while true do  -- 将 s 中所有的换行符位置保存在t中
	i = string.find(s,"\n",i+1)  -- 查找下一个换行符
	if i == nil then break end
	t[#t+1] = 1
end

-----------------------

print( string.match("hello world" ,"hello") )  --> hello
date = " today is 17/7/1990"
print( string.match(date,"%d+/%d+/%d+") )  --> 17/7/1990  类似正则

------------------------

print( string.gsub( "Lua is cute" , "cute" , "greate" ) )  --> Lua is greate	1

s = string.gsub("all lii" , "l" , "x")
print(s)  --> axx xii

s = string.gsub("Lua is great" , "Sol" , "Sun" )
print(s)  --> Lua is great

s = string.gsub("all lii" , "l" , "x" , 1)
print(s)  --> axl lii

------------------------

string.gmatch
words = {}  -- 将 s 的所有字母存放到 words 中
for w in string.gmatch(s,"%a+") do        -- %a  表示一个字母
	words[#words + 1 ] = w
end
```

注意 : 
``` Lua
	模式:
		字符分类:
			%a  字母                      %A(表示所有非字母的字符)    (大写是小写的补集)
			%c  控制字符                  %C(表示非控制字符)
			%d  数字零到九
			%l  小写字母
			%p  标点符号
			%s  空白字符
			%u  大写字母
			%w  字母和数字
			%x  十六进制数字
			%z  内部表示为0的字符
		魔法字符: ( ) . % + - * ? ^ $
			%.  匹配一个 .
			%%  匹配一个 %
			...
		字符集:
			[%w_] 同时匹配 字母数字和下划线
			nvow = select( 2 , string.gsub( text , "[AEOUaeiou]" ,"" ) )  -- 文本text中 元音字符的数量
			[0-9] %d
			^ 表示 非 [^0-7] 表示所有非八进制数字的字符
		修饰符:
			+ 一个以上 %a+  print(string.gsub("one, and two ; and three ", "%a+","word"))  -- 把所有单词变成 word
			* 0个以上
			- 和*一样 , 但会匹配最短的子串  "[_%a][_%w]-"
			? 匹配一个 [-+]? + 或 -
			^ string.find(s,"^%d")  匹配 以数字开头的字符串
			$ string.find(s,"%d$")  匹配 以数字结束的字符串
```
#### 常用方法
1. 多个捕获,需要加上小括号 %s表示空格

  ``` Lua
  pair = "name=Anna"
  key , value = string.match(pair,"(%a+)%s*=%s*(%a+)")  -- 可多个捕获,需要加上小括号 %s表示空格
  print(key , value)

  date = "Today is 17/7/1990"
  d , m , y = string.match(date,"(%d+)/(%d+)/(%d+)")
  print(d,m,y)  -->  17 7 1990
  ```
2. %加数字表示捕获序号
  ``` Lua
  s = [[then he said:"it's all right"!]]
  q,quotedPart = string.match(s,"([\"'])(.-)%1")
  print(q,quotedPart)

  p = "%[(=*)%[(.-)%]%1%]"
  s = "a=[[[ something]]]==]]=];print(a)"
  print( string.match(s,p) )

  local s = "abcdefg"
  print(string.gsub(s,"(%w)(%w)(%w)","%3%2%1")) --cbafedg  %加数字表示捕获序号

  print(string.gsub("hello lua ! " , "%a" , "%O-%O" ))

  print(string.gsub("hello lua","(.)(.)","%2%1"))

  function trim(s)
	return (string.gsub(s,"^%s*(.-)%s*$","%1"))
  end
  ```
3. 去除字符串两端的空格 trim(s)
  ``` Lua
  s = "     safdsaf          "
  print( trim(s) .. "aa" )  
  ```
4. $varname varname 利用属性值修改字符串
  ``` Lua
  
  function expand(s)
	return ( string.gsub(s,"$(%w+)",_G) )
  end

  name = "Lua"
  status1 = "great"
  print( expand( "$name is $status , isn't it ? " ) )

  function expand(s)
	return ( string.gsub(s,"$(%w+)",function(n)
		return tostring(_G[n])
	end
	) )
  end
  status1 = "great"
  print( expand( "$name is $status , isn't it ? " ) )
  ```
5. "()"  空捕获应用
  ``` Lua
    print( string.match("hello" , "()ll()" ) )  --> 3	5  分别是 e 和 o 的位置
  ```
#### URL编码 \ tab扩展
1. URL编码解码
  ``` Lua
  function unescape(s)
	s = string.gsub( s , "+" , " " )
	s = string.gsub( s , "%%(%x%x)" , function(h)
			return string.char( tonumber( h , 16 ) )  -- tonumber( h , 16 ) 把16进制的h 转为 10进制的数字
		end
	)
	return s
  end

  s = "a%2Bb+%3D+c"
  print( unescape(s) )

  cgi = { key1 = "1" , key2 = "2" , key3 = "3" }
  function decode(s)
	for name , value in string.gmatch( s , "([^&=]+)=([^&=]+)") do
		name = unescape(name)
		value = unescape(value)
		cgi[name] = value
	end
  end
  print(  decode(cgi) )

  function escape(s)
	s = string.gsub( s , "[&=+%%%c]" , function(c)
			return string.format("%%%02X",string.byte(c))
		end)
		s = string.gsub( s , " " , "+" )
		return s
  end

  function encode()
	local b = {}
	for k , v in pairs(t) do
		b[#b+1] = ( escape(k) .. "=" .. escape(v) )

	end
	return table.concat(b,"&")
  end

  t = { name = "al" , quer = "a+b=c" , q = "yes or no" }

  print( encode(t) )
  ```
2. tab 扩展
  ``` Lua
  -- 需求 : 将字符串中的 制表符 转为 tab 个 空格 (默认为8个)
  function expandTabs( s , tab )
	tab = tab or 8  -- 制表符的大小默认为8
	local corr = 0
	s = string.gsub( s , "()\t" , function(p)
		local sp = tab - (p-1+corr)%tab
		corr = corr - 1 + sp
		return string.rep(" ", sp)
	end)
	return s
  end

  s = "This	is a	book."  -- this 和 is 之间 \ a 和 book 之间  有制表符
  print( expandTabs(s) )
  ```
#### 小技巧

``` Lua
  test = [[char s[] = "a/*here";/*a tricky string */]]
  -- 将 c 语言代码中的 注释代码 改为 <COMMENT>
  print( string.gsub(test,"/%*.-%*/" , "<COMMENT>") )  --> char s[] = "a<COMMENT>	1  理想是 char s[] = "a/*here";<COMMENT>

  -- "(.-)%$"   -- (.-)匹配一个任意字符  -- 时间复杂度 n^2  n为字符串长度
  -- "^(.-)%$"  -- 时间复杂度 n

  -- 小心空模式的使用
  -- "%a*"
  i , j = string.find(";$% **#$hello13" , "%a*")
  print( i , j )  --> 1	0
  -- "-"   ".*"   "[^\n]"
  pattern = string.rep("[^\n]" , 70 ) .. "[^\n]*"
  -- print( pattern )

  -- x "[xX]"
  function nocase(s)
	s = string.gsub( s , "%a" , function(c)
		return "[" .. string.lower(c) .. string.upper(c) .. "]"
	end)
	return s
  end

  print( nocase("Hi there!") )  --> [hH][iI] [tT][hH][eE][rR][eE]!


  -- 对字符串进行预处理
  s = [[follows a typical string:"This is \"great\"!.]]   -- "\x" \ddd

  function code(s)
	return (string.gsub(s,"\\(.)",function(x)
		return string.format("\\%03d",string.byte(x))
	end))
  end

  function decode(s)
	return (string.gsub(s,"\\(%d%d%d)" , function(d)
		return "\\" .. string.char(d)
	end))
  end

  s = code( s )
  print(s)
  s = string.gsub( s , '".-"' , string.upper )
  print(s)
  s = decode(s)
  print(s)
  ```