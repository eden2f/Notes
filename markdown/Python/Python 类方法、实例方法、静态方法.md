# Python 类方法、实例方法、静态方法

``` python
# 类属性 实例属性

class Tool(object):

	# 类属性 
	num = 0 # 相当于java 的静态属性

	def __init__(self , name):
		self.name = name # 这些是实例属性
		# 对类属性修改
		Tool.num += 1

tool1 = Tool("铁锹")
tool2 = Tool("工兵铲")
tool3 = Tool("水桶")
tool4 = Tool("票子")
print(Tool.num)

# 类方法 实例方法 静态方法
class Game(object):

	# 类属性
	num = 0

	# 实例方法
	def __init__(self):
		self.name = "laowang"

	# 加上 @classmethod 后变类方法  
	@classmethod
	def addNum(cls): # 不用一定叫 cls , 也可以叫别的, 但是要在第一个
		cls.num += 1 ;

	# 这是静态方法  和 实例方法的区别是,实例方法需要第一个参数 为 self
	@staticmethod
	def title(): # 
		print("----------title-------")

game = Game();
Game.addNum(); # 调用类方法
print(Game.num)
game.addNum(); # 调用类方法
print(Game.num)
print(game.num)

game.title(); # 静态方法
Game.title(); # 静态方法
```