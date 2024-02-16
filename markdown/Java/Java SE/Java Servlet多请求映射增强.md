# Java Servlet多请求映射增强

## 前言

当浏览器发送⼀次请求到服务器时，Servlet容器会根据请求的url-pattern找到对应的Servlet类，执⾏对应的doPost或doGet⽅法，最后将响应信息返回给浏览器。

这种情况下，⼀个具体的Servlet类只能处理对应的web.xml中配置的url-pattern请求，⼀个Servlet类，⼀对配置信息。

如果业务扩展，需要三个Servlet来处理请求，就需要再加上两个具体的Servlet类，两份配置信息。以此类推，每新增一个接口都需要硬编码才能支持，且需要在多处新增代码，不易维护。

如何在一个Servlet中处理多个URL请求？类似SpringMVC的Controller。

## Servlet基类抽取

1. 前端访问的时候, 在请求参数中添加 method 属性, 其中值是将要访问的方法

2. 在 BaseServlet 中利用反射机制, 根据 method 的值, 调用对应的方法

``` java
public class BaseServlet extends HttpServlet {

	@Override
	protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		
		req.setCharacterEncoding("UTF-8");
		
		try {
			//1、获得请求的method的名称
			String methodName = req.getParameter("method");
			//2、获得当前被访问的对象的字节码对象
			Class clazz = this.getClass();//ProductServlet.class ---- UserServlet.class
			//3、获得当前字节码对象的中的指定方法
			Method method = clazz.getMethod(methodName, HttpServletRequest.class,HttpServletResponse.class);
			//4、执行相应功能方法
			method.invoke(this, req,resp);
			
		} catch (Exception e) {
			e.printStackTrace();
		}
		
	}
}
```

## 业务Servlet实现方式

 1. 继承 BaseServlet
 2. 实现业务方法

``` java
public class PrdocutServlet extends BaseServlet {

    //删除一个商品
    public void delProFromCart(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        // 删除一个商品的逻辑
    }

    //根据商品的类别获得商品的列表
    public void productList(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        // 逻辑代码
    }

}
```