# Lua 协同程序

## 协同程序

协同程序可以借助 Java 的线程来理解，不过 Java 的线程可以同时运行多个，而 Lua 的协同程序，同时只能运行一个。

## 协同程序的基本方法

### 创建一个协同程序

``` lua
co = coroutine.create(function() print("Hi") end)
print(co)
```

### 协同程序的4种状态

* 挂起(suspended)
* 运行(running)
*  死亡(dead)
* 正常(normal)

### status 检查协同程序的状态

``` lua
print(coroutine.status(co)) 
```

### resume 启动协同程序

``` lua
coroutine.resume(co)
print(coroutine.status(co))
```
### yield 让运行中的协同程序挂起

``` lua
coroutine.yield(co)
print(coroutine.status(co))
```

## 示例

``` lua
co = coroutine.create(
function()
    for i=1 , 2 do
        print("co",i)
        coroutine.yield()
    end
end
)
coroutine.resume(co)
coroutine.resume(co)
coroutine.resume(co)
print( coroutine.resume(co) )

co = coroutine.create(function(a,b,c)
print("co" .. a .. b .. c )
end
)
coroutine.resume(co,1,2,3)
print(coroutine.status(co))

co = coroutine.create(function(a,b)
coroutine.yield(a+b,a-b)
end
)
print( coroutine.resume(co,20,10) )

co = coroutine.create(function()
print("co",coroutine.yield())
end
)
coroutine.resume(co)
print(coroutine.status(co))
coroutine.resume(co,2,5)

co = coroutine.create(
function()
    return 6.7
end
)
print( coroutine.resume(co) )
print(coroutine.status(co))
```