# Lua 实现私密性

模拟面向对象的封装思想,隐藏内部属性访问,对外提供方法对其进行修改

## 方法1 原理:使用两个table来操作一个对象,一个用于保存对象的属性,另一个作为对外的接口

``` Lua
-- 实体类
function newAccount( initialBalance , LIM )
local self = { balance  = initialBalance , LIM = LIM or 0 }

-- 取款
local withdraw = function(v)
    self.balance = self.balance - v
end

-- 存款
local deposit = function(v)
    self.balance = self.balance + v
end

local extra = function()
    if self.balance > 10000.00 then
        return self.balance * 0.10
    else
        return 0
    end
end

-- 账号存款余额
local getBalance = function()
    return self.balance + extra()
end


return {  -- 这个 table 是 实体类接口
    withdraw = withdraw ,
    deposit = deposit ,
    getBalance = getBalance
}
end

acc1 = newAccount( 100.00 )
acc1.withdraw( 40.00 )
print( " acc1.getBalance() : " .. acc1.getBalance() )
acc2 = newAccount( 100000.00 )
print( " acc2.getBalance() : " .. acc2.getBalance() )
```
## 方式2 
``` Lua
function newObject(value)
return function( action, v )
    if action == "get" then
        return value
    elseif action == "set" then
            value = v
        else
        error("incalid action")
    end
end
end

d = newObject(0)
print(d("get"))
d( "set" , 10 )
```