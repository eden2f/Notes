# Lua 操作系统库

## 日期和时间

1. os.time()
2. os.date()

``` Lua
os.date("*t" , 3600)  返回一个表示日期的table
        %c : 日期和时间(09/16/98 23:12:50)
        %a : 星期中天数的简写
        %A : 行其中天数的全称
        %b : 月份的简写
        %B : 月份的全称
        %d : 一个月中的第几天(1-31)
        %H : 24小时制中的小时数[0-23]
        %I : 12小时制中的小时数([01-12])
        %j : 一年中的第几天([001-366])
        %M : 分钟数([00-59])
        %m : 月分数([1-12])
        %p : 上午(am) 或 下午(pm)
        %S : 秒数([00-59])
        %w : 一个星期中的第几天[0-6=等于星期天-星期六]
        %x : 日期(16/09/98)
        %X : 时间(23:48:10)
        %y : 两位数的年份(00-99)
        %Y : 完整的年份
        %% : 字符'%'
    os.clock()  返回cpu的秒数
    os.getenv("TEMP") -- 获取系统环境变量值
    os.execute("dir c:\\")  -- 执行一个系统命令
    os.setlocale(区域名,分类名) : -- 设置lua的区域
        分类: "collate";"ctype";"monetary";"numeric";"time";"all"
```

## 其他系统调用

``` Lua
print(os.time())
print( os.time({year=1990,month = 10 , day = 28 , hour = 1 , mintute = 58 , sec =30 , isdst = false }) )
print( os.date("*t" , 3600) )  -->{year = 1970 , month = 1 , day = 1 , hour = 9 , min = 0 , sec = 0 , isdst = false }

print( os.date("%a" , 3600) )
print( os.date("%A" , 3600) )

local x = os.clock()
local s = 0
for i = 1 , 1000000 do
s = s
end
print(string.format("elapsed time:%.2f\n",os.clock()-x))
```

``` Lua
print(os.getenv("PATH") )  -- 得到系统环境变量
print(os.execute("dir c:\\"))  -- 执行一个系统命令
```