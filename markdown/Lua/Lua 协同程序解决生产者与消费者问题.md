# Lua 协同程序解决生产者与消费者问题

## （基础版）消费者为程序入口

``` Lua
producer = coroutine.create( -- 创建一个协同程序
function()  -- 生产者
    while true  do
        local x = io.read() -- 产生新的值
        print("生产一个值",x)
        send(x)  -- 挂起生产者协同程序，发送给消费者
    end
end
)

function consumer()  -- 消费者
while true do
    local x = receive()  -- 激活生产者协同程序 ，从生产者接收值
    io.write(x,"\n")  -- 消费新的值
    print("生消费一个值：",x)
end
end

function receive() -- 激活一个协同程序 同时传递参数
local status,value = coroutine.resume(producer)
return value
end

function send(x)  -- 挂起一个协同程序 同时传递参数
coroutine.yield(x)
end

consumer()  -- 初始化消费者并启动程序
```

## （扩展版）引入过滤器概念，增强程序功能，消费者为程序入口

``` lua
function receive(prod)  -- 激活协同程序
local status,value = coroutine.resume(prod)
return value
end

function send(x)  -- 将协同程序挂起
coroutine.yield(x)
end

function producer()  -- 生产者
return coroutine.create( function()  -- 创建一个协同程序
        while true do
            local x = io.read() -- 产生新值
            send(x)
        end
    end
)
end

function filter(prod)  -- 过滤器
return coroutine.create(  -- 创建一个协同程序
    function()
        for line = 1, math.huge do
            local x = receive(prod)  -- 激活协同程序 获取新值
            x = string.format("%5d %s",line , x )
            send(x)  -- 挂起激活程序
        end
    end
)
end

function consumer(prod)
while true do
    local x = receive(prod) -- 获取新值
    io.write(x, "\n")
end
end

p= producer()  -- 初始化生产者
f = filter(p)  -- 初始化过滤器
consumer(f)  -- 初始化消费者 并 启动程序
```