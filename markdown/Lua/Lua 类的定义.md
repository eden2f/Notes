# Lua 类的定义

在Lua中模拟类的实现方法

## 方法1
``` Lua
a = { x = 1 }
b = { x = 2 , y = 3 }
setmetatable( a , { __index = b } )  -- 设置 b 为 a 的父类
print(a.x)
print(a.y)
```
## 方法2
``` Lua
function Account:new(o)
o = o or {}  -- 如果用户没有提供table , 则创建一个table
setmetatable( o , self )
self.__index = self
return o
end
a = Account:new{balance = 0}
a:deposit(100.00)  -- 等效与 a.deposit(a, 100.00) 也可以简化成 Account.deposit(a,100.00)
print(a.balance)

b = Account:new()
print(b.balance)
b.deposit(b,10)  -- b.balance = b.balance + v
print(b.balance)
```
## 注意:
给对象添加方法时, 使用 : 和 . 是一样的 , ":" 只是隐藏了 self 参数 (self是其本身)