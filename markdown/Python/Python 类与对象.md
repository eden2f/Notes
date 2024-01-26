# Python 类与对象

``` python

# 定义一个类
# class 类名:
	# 属性
	# 方法
	# def xxx():
		# pass

class Cat:
	"""定义了一个Cat类"""
	def eat(self):
		print("猫在吃鱼")

	def drink(self):
		print("猫正在喝水")

	def introduce(self): # self 用来传递当前对象 可以改成a b 等等, 但是一定第一个参数
		print("%s 的年龄是 : %d"%(self.name , self.age))

# 创建一个对象
tom = Cat();
# 调用对象的方法
tom.eat();
tom.drink();
# 设置属性
tom.name = "汤姆"
tom.age = 18
# 获取属性的第一种方式
print("%s 的年龄是 : %d"%(tom.name , tom.age))
# 第二种方式
tom.introduce()

lanman = Cat()
lanman.name = "蓝猫"
lanman.age = 10
lanman.introduce()


# __init__ 方法
class Cat:
	"""定义了一个Cat类"""
	# 初始化对象
	def __init__(self , name , age ):
		print("-----创建新对象------")
		self.name = name
		self.age = age

	def eat(self):
		print("猫在吃鱼")

	def drink(self):
		print("猫正在喝水")

	def introduce(self): # self 用来传递当前对象 可以改成a b 等等, 但是一定第一个参数
		print("%s 的年龄是 : %d"%(self.name , self.age))

lanman = Cat( "蓝猫" , 18 )
lanman.introduce()

# __str__ 方法
class Cat:
	"""定义了一个Cat类"""

	# 定义对象的描述信息
	def __str__(self):
		return "%s 的年龄是 : %d"%(self.name , self.age)

	# 初始化对象
	def __init__(self , name , age ):
		print("-----创建新对象------")
		self.name = name
		self.age = age

	def eat(self):
		print("猫在吃鱼")

	def drink(self):
		print("猫正在喝水")

	def introduce(self): # self 用来传递当前对象 可以改成a b 等等, 但是一定第一个参数
		print("%s 的年龄是 : %d"%(self.name , self.age))

lanman = Cat( "蓝猫" , 18 )
lanman.introduce()
print( lanman )
```