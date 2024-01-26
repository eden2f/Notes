# Python __new__与方法单例模式

``` python
# __new__ 方法

class Dog(object):
	def __init__(self):
		print("---init方法---")
	def __del__(self):
		print("---del方法---")
	def __str__(self):
		print("---str方法---")

	# 创建对象时就会被调用
	# 没重载之前,默认是调用父类的 __new__
	def __new__(cls): # cls 此时时Dog只想的那个类对象
		print("---new方法---")
		print(id(cls))
		return object.__new__(cls)

print(id(Dog))
d = Dog();
"""
1. 调用 __new__ 来创建对象,然后找了一个变量来接收__new__的返回值,这个返回值表示,创建出来的对象的引用
2. 调用__init__(刚刚常见出来的对象的引用)
3. 返回对象的引用
	所以 __new__ 只负责创建 , __init__ 只负责初始化 
	而Java的构造方法等于 __new__() + __init__()
"""


# 单例模式 实现
class Dog(object):

	__instance = None;

	# 创建对象时就会被调用
	# 没重载之前,默认是调用父类的 __new__
	def __new__(cls): # cls 此时时Dog只想的那个类对象
		print("---new方法---")
		if cls.__instance == None:
			cls.__instance = object.__new__(cls);
		return cls.__instance ;

a = Dog();
b = Dog();
print( id(a) == id(b) )

```