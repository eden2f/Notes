# Python 爬虫脚本示例

### 案例环境

1. Python 3.6.5
2. selenium 3.12.0
3. Chrome 版本 66.0.3359.181（正式版本） （64 位）
4. IntelliJ PyCharm 2017

### 示例代码

``` python
# coding=utf-8
from selenium import webdriver

if __name__ == '__main__':
    driver = webdriver.Chrome()
    driver.get("http://www.baidu.com")

    driver.find_element_by_id("kw").send_keys("Selenium2")
    driver.find_element_by_id("su").click()
    driver.quit()
```

## 代码解析

1. coding=utf-8
 为了防止乱码问题， 以及方便地在程序中添加中文注释，把编码统一成UTF-8
1. from selenium import webdriver
 导入 Selenium 的WebDriver包。 皱皮导入 WebDriver 包， 才能够使用 WebDriver AP 进行自动化脚本开发。
1.  driver = webdriver.Chrome()
 把 webdriver 的 Chrome 对象赋值给变量driver，只有获得了浏览器对象后， 才可以启动浏览器。打开网址， 操作页面元素， 火狐浏览器驱动默认已经在 Selenium WebDriver 包里了， 所以可以直接调用。如果要使用 Chrome 或 其他浏览器运行 Webb 自动化测试用例， 则需要先安装相应的浏览器驱动才行。
1. driver.get("http://www.baidu.com")
 获得浏览器对象后， 通过 get() 方法， 可以向浏览器发送网址（URL）。
1. driver.find_element_by_id("kw").send_keys("Selenium2")
 关于页面元素的定位，这里通过 id = kw ，定位到百度的输入框， 并通过键盘输入方法 send_keys() 向百度输入框输入“Selenium2” 搜索关键字
1.  driver.find_element_by_id("su").click()
 这一步通过 id=su 定位“百度一下”搜索按钮， 并向搜索按钮发送单击事件
1. driver.quit()
 退出并关闭浏览器，及相关的驱动程序。