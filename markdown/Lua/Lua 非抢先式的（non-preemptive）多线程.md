# Lua 非抢先式的（non-preemptive）多线程

## 依赖

* LuaSocket 库
* 使用HTTP下载远程文件实例

## LuaSocket实现文件下载

``` lua
require "socket"                                 -- 加载 LuaSocket
host = "www.w3.org"                              -- 定义主机
file = "/Help/Webmaster.html"                    -- 下载的文件
c = assert(socket.connect(host,80))              -- 链接到该站点的80端口
c:send("GET " .. file .. " HTTP/1.0\r\n\r\n")     -- 发送文件请求
while true do
    local s, status, partial = c:receive(2^10)
    io.write(s or partial )
    if status == "closed" then
        break
    end
end
c:close()
```
## 多线程下载

``` lua
require "socket"
function download(host , file )
    local c = assert(socket.connect( host , 80))
    local count = 0 -- 记录接收到的字节数
    c:send("GET " .. file .. " HTTP/1.0\r\n\r\n")
    while true do
        local s,status,partial = receive(c)
        count = count + #( s or partial)
        if status == "closed" then
            break
        end
    end
    io.write("文件下载完成,大小: " .. count .. "\n" )
end
-- 顺序下载
function receive(connection)
    return connection:receive(2^10)
end
--  并发下载
function receive(connection)
    connection:settimeout(0)                     -- 使用receive调用不会堵塞
    local s , status , partial = connection:receive(2^10)
    if status == "timeout" then
        coroutine.yield(connection)
    end
    return s or partial , status
end
threads = {}                  -- 用于记录所有正在运行的线程
function get( host , file )
    -- 创建协同程序
    local co = coroutine.create(
        function()
            download(host,file)
        end
    )
    table.insert(threads,co)
end
function dispatch()
    local i = 1
    while true do
        if threads[i] == nil then  -- 还有线程吗？
            if threads[1] == nil then   -- 判断列表是否为空
                break
            end
            i = 1
        end
        local status , res = coroutine.resume(threads[i])
        if not res then
            table.remove(threads,i)
        else
            i = i + 1
        end
    end
end
host = "www.w3.org"                              -- 定义主机
file1 = "/standards/webofservices/"               -- 下载的文件
file2 = "/Help/Webmaster.html"                    -- 下载的文件
file3 = "/participate/"                    -- 下载的文件
get(host , file1)
get(host , file2)
get(host , file3)
dispatch()
```

## 实现多线程下载 (假如超时处理逻辑)

``` lua
    require "socket"
    function download(host , file )
    local c = assert(socket.connect( host , 80))
    local count = 0 -- 记录接收到的字节数
    c:send("GET " .. file .. " HTTP/1.0\r\n\r\n")
    while true do
        local s,status,partial = receive(c)
        count = count + #( s or partial)
        if status == "closed" then
            break
        end
    end
    io.write("文件下载完成,大小: " .. count .. "\n" )
end
-- 顺序下载
function receive(connection)
    return connection:receive(2^10)
end
--  并发下载
function receive(connection)
    connection:settimeout(0)                     -- 使用receive调用不会堵塞
    local s , status , partial = connection:receive(2^10)
    if status == "timeout" then
        coroutine.yield(connection)
    end
    return s or partial , status
end
threads = {}                  -- 用于记录所有正在运行的线程
function get( host , file )
    -- 创建协同程序
    local co = coroutine.create(
        function()
            download(host,file)
        end
    )
    table.insert(threads,co)
end
function dispatch()
    local i = 1
    while true do
        if threads[i] == nil then  -- 还有线程吗？
            if threads[1] == nil then   -- 判断列表是否为空
                break
            end
            i = 1    --　重新开始循环
        end
        local status , res =   coroutine.resume(threads[i])
        if not res then
            table.remove(threads,i)
        else        -- 超时
            i = i + 1
            if connections==nil then
                connections = {}   -- connections 用来存放当前超时的 线程
                connections[1] = res
            else
                connections[#connections + 1 ] = res
                if #connections == #threads then    -- 所有线程都堵塞了
                    socket.select(connections)
                end
            end
        end
    end
end
host = "www.w3.org"                              -- 定义主机
file1 = "/standards/webofservices/"               -- 下载的文件
file2 = "/Help/Webmaster.html"                    -- 下载的文件
file3 = "/participate/"                    -- 下载的文?
?
get(host , file1)
get(host , file2)
get(host , file3)
dispatch()
```