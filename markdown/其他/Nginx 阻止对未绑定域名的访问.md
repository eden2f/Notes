# Nginx 阻止对未绑定域名的访问

> [Bpazy](https://blog.csdn.net/hanziyuan08) 原文链接：[nginx 阻止对未绑定域名的访问](https://blog.csdn.net/hanziyuan08/article/details/104031318)

当请求走进 nginx 时，会依次匹配每一个 server 和 location 块。
当某个请求访问了未绑定的 server_name，由于每个 server 和 location 都访问不上，就会默认选择第一个，下面举例说明：
nginx配置文件节选
```nginx
server {
	location {
		server_name a.example.com;
		index index.html;
	}
	location {
		server_name b.example.com;
		index index.html;
	}
}
```
当请求访问的地址是 c.example.com 的时候会发生什么？
答案是请求匹配到了 a.example.com。
所以为了阻止这种情况的发生，可以配置一个默认的 server 块用于阻止非法请求：
```nginx
server {
	listen 80;
	listen 443;
	return 444;
}
```
```nginx
server {
	location {
		server_name a.example.com;
		index index.html;
	}
	location {
		server_name b.example.com;
		index index.html;
	}
}
```
另外，你还可以通过先显式指定 default_server 的方式：
```nginx
server {
	listen 80 default_server;
	listen 443 default_server;
	return 444;
}
```
这样你就不必依赖 server 块配置的顺序了，推荐。
