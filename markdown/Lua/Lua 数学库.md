# Lua 数学库

## 常用方法

* 三角函数( sin , cos , tan , asin , acos ... )
* 指数和对数函数( exp , log , log10 )   e=2.71828...
* 取整函数( floor , ceil )
* 取最大值和最小值  max , min
* 生成伪随机数函数( random , randomseed )
* 常量 ( pi 圆周率 , huge 表示最大的数字 )
* 度与角度

## 示例

``` Lua
--[[
数学库:
三角函数( sin , cos , tan , asin , acos ... )
指数和对数函数( exp , log , log10 )   e=2.71828...
取整函数( floor , ceil )
取最大值 和 最小值  max , min
生成伪随机数函数( random , randomseed )
常量 ( pi 圆周率 , huge 表示最大的数字 )
度与角度
--]]

-- exp(2) --> e^2
print("exp(1) : " .. math.exp(1) )
print("exp(2) : " .. math.exp(2) )

-- log 以e为底的对数
print("log(e) : " .. math.log(2.718281828459) )
-- log10 以10为底的对数
print("log10 : " .. math.log10(10000) )

-- 取整函数( floor , ceil )
-- floor(3.15) = 3  放回小于该数的最大整数
-- ceil(3.15) = 4  放回大于该数的最小整数
print( "floor(3.15) : " ..  math.floor(3.15) )
print( "floor(3.85) : " ..  math.floor(3.85) )
print( "ceil(3.15) : " ..  math.ceil(3.15) )
print( "ceil(3.85) : " ..  math.ceil(3.85) )

-- 取最大值 和 最小值  max , min

-- 生成伪随机数函数( random , randomseed )
-- random 用于生成伪随机数
print("math.random() : " .. math.random() )  -- 默认生成 0~1 的小数
print("math.random(6) : " .. math.random(6) )  -- 1~6
print("math.random( 2, 8 ) : " .. math.random(2, 8 ) )  -- 2~8

-- 设置随机种子
math.randomseed( os.time() )  --> 使用 c库 rand
for i = 1 , 5 do
print( "randomseed : " .. math.random(6) )
end

-- 常量 ( pi 圆周率 , huge 表示最大的数字 )
print( "pi : " .. math.pi )
print( "huge : " .. math.huge )

-- 度
print( " math.sin(90) : " .. math.sin(90) )  -- 90 是弧度 , 先把角度转成角度
print( " math.sin( math.pi / 2 ): " .. math.sin( math.pi / 2 ) )

-- math.deg 把弧度转化为角度  math.rad  把角度转化成弧度
print( "math.deg( 90 ) : " .. math.deg( 90 ))

local sin = math.sin
local asin = math.asin
local deg = math.deg
local rad = math.rad
math.sin = function(x)
return sin( rad(x) )
end
math.asin = function(x)
return deg(asin(x))
end
print( "math.asin(1) : " .. math.asin(1))
print( "math.sin(90) : " .. math.sin(90))
```