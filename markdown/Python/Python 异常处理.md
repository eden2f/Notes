# Python 异常处理

``` python
# 异常处理
try:
	open("asdfsa");
	print(num);
except ( NameError , FileNotFoundError ) as e:
	print("捕获到异常 :",e)
except Exception as e:
	print("捕获到异常 :",e)
else:
	print("没出现异常就会执行这里的代码")
finally:
	print("异常处理完成")

# 抛出自定义异常
# raise 引发一个自定i的异常
class ShortInputException(Exception):
	"""自定义的异常类"""
	def __init__(self , length , atleast ):
		# super().__init__()
		self.length = length ;
		self.atleast = atleast ;

try:
	raise ShortInputException(len(s), 3)
except ShortInputException as e:
	print("捕获到异常 :长度:%d %s"%(e.length,e.atleast))
except Exception as e:
	print("捕获到异常 :",e)
finally:
	print("处理自定义异常完成")
# 异常处理中抛出异常
```