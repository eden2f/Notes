# Python 继承

``` python
# 继承
class Animal:

	def eat(self):
		print("eat")

	def drink(self):
		print("drink")

	def sleep(self):
		print("sleep")


class Dog(Animal): # 继承 Animal

	def bark(self):
		print("汪汪汪")

class SmallDog(Dog, Animal): # 多继承

	def bark(self):
		print("旺旺 不是 汪汪")
		# Dog.bark(self) # 调用父类的方法
		# super().bark() # 通过 super(). 调用父类方法,和java一样

a = Animal()
a.eat()
d = Dog();
d.bark();
sd = SmallDog();
sd.eat();
sd.bark();

class A:
	def __init__(self):
		self.num1 = 100
		self.__num2 = 200

	def test1(self):
		print("test1")

	def __test2(self):
		print("test2")

class B(A):
	pass

b = B();
b.test1();
print(b.num1)
# 私有方法并不会被继承
# b.__test2(); # AttributeError: 'B' object has no attribute '__test2'
# 私有属性并不会被继承
# print(b.__num2) # AttributeError: 'B' object has no attribute '__num2'

# 所有类 默认 继承 object 类似java
class C(object):
	pass

class A:
	def test(self):
		print("testA")

class B:
	def test(self):
		print("testB")

class C(A,B):

	def test3(self):
		print("testC")

c = C()
print(C.__mro__) # 类的寻找方法  # (<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <class 'object'>)
c.test() # testA

```