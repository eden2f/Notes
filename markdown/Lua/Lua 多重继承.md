# Lua 多重继承

需求:创建一个对象,使其继承父类:Named,Account,成功调用父类方法

``` Lua 
  -- 在 table 'plist' 查找k
  local function search( k , plist )
	for i = 1 , #plist do
		local v = plist[i][k]  -- 尝试第i个基类
		if v then return v end
	end
  end

  function createClass(...)
	local c = {}  -- 新类
	local parents = {...}

	-- 类可在其父类列表中的搜索方法
	setmetatable( c , { __index = function(t, k )
		return search( k , parents )
	end
	})

	-- 将 'c'作为其实例的元表
	c.__index = c

	-- 为新类定义构造函数(construction)
	function c:new(o)
		o = o or {}
		setmetatable( o , c )
		return o
	end
	return c  -- 返回新建的类
  end

  -- 父类1 开始
  Named = {}
  function Named:getName()
	return self.name
  end
  -- 父类1 结束

  function Named:setName(n)
	self.name = n
  end

  -- 父类2 开始
  Account = { balance = 0 }

  function Account:new(o)
	o = o or {}
	setmetatable( o , self )
	self.__index = self
	return o
  end

  -- 存款方法
  function Account:deposit(v)
	self.balance = self.balance + v
  end

  -- 取款方法
  function Account:withdraw(v)
	if v > self.balance then
		error "insufficient funds"
	end
	self.balance = self.balance - v
  end
  -- 父类2 结束

  NamedAccount = createClass( Account , Named ) -- 创建新的 NamedAccount 类
  account = NamedAccount:new( { name="Paul" } )
  print("account:getName() : " .. account:getName())
  print("account.balance : " .. account.balance)
```

注意 : 改进 上面方法完成了多继承 但效率不高,可如下修改,将父类的变量直接复制过来 , 但这样缺点是, 运行后,不可进行修改,因为并没有真的继承父类,只是简单把父类的变量赋值过来而已
  ``` Lua
  -- 修改上面代码中的 setmetatable 方法
  setmetatable( c , { __index = function(t, k )
	local v = search( k , parents )
	t[k] = v -- 保存下来, 以备下次访问
	return v
  end
  })

  ```