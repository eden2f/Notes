# Python 类的隐藏属性和隐藏方法

``` python
# 隐藏属性
class Dog():
	"""Dog"""
	def __init__(self, name , age ):
		self.name = name
		self.__age = age

	def setAge(self,age):
		self.__age = age

	def getAge(self):
		return self.__age;

d = Dog("小白" , 18 );
print( d.getAge()) 
d.age = 15
print( d.getAge()) 

# 隐藏方法
class Dog():
	"""Dog"""
	def __init__(self, name , age ):
		self.name = name
		self.__age = age # 私有属性

	def setAge(self,age):
		self.__age = age

	def getAge(self):
		return self.__age;

	def __getAge2(self):
		return self.__age
d = Dog("小白" , 18 );
# print( d.__getAge2()) # AttributeError: 'Dog' object has no attribute '__getAge2'
```