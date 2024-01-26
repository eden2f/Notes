# Python 类属性和实例属性

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
```