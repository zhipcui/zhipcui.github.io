---
layout: post
title:  "Programming Ruby: 容器、blocks和迭代器"
date:   2014-11-25
categories: Ruby
tag: Programming Ruby

---


点唱机没有理由只有一首歌，我们需要有播放列表和歌曲类别列表，这都是**容器**：可以存储多个对象的引用。

播放列表和歌曲类别列表有一些相似的方法，如增添歌曲，删除歌曲，返回歌曲列表等。播放列表可能还有其他的方法，如插入广告、记录播放时间等，但是我们先不用关注这些，我们先来实现一个通用基类`SongList`先，歌曲类别和播放列表可以**继承**它。

###容器

在我们写具体代码前，我们要考虑怎么在`SongList`中存储歌曲列表。有三个选择，用`Array`类型（数组）、用`Hash`类型（字典)、或者我们自己来实现列表结构。既然有现成的，那就不要再自己做了。我们可以用ruby内置的**数组**类型或者**字典**类型。

####数组

数组是一个保存多个对象引用的集合，每个对象都有特定的索引位置，由非负整形数标识。你可以用`[ ]`直接创建数组或者通过`new`构造器。

	a = [ 3.14159, "pie", 99]
	a.type                 >>	Array
	a.length               >>	3
	a[0]                   >>	3.14159
	a[1]                   >>	"pie"
	a[2]                   >>	99
	a[3]                   >>	nil
	
	b = Array.new
	b.type                 >>	Array
	b.length               >>	0
	b[0] = "second"
	b[1] = "array"
	b                      >>	["second", "array"]

数组的检索用`[ ]`操作符，实质上这是个方法，因此你可以在子类中重载它。数组的位置从0开始，如果索引位置不存在返回`nil`。如果你提供的索引是负数，代表从后面开始数起。

	a = [1,3,5,7,9]
	a[-1]                  >>	9
	a[-2]                  >>	7
	a[-99]                 >>	nil
	
你可以提供一对数来访问数组，`[start, count]`,它会返回从`start`位置开始，后面`count`个对象。

	a = [1,3,5,7,9]
	a[1,3]                 >>	[3,5,7]
	a[3,1]                 >>	[7]
	a[-3,2]                >>	[5,7]

最后你可以通过范围来访问数组，提供开始和结束的位置。`[start..end]`表示包含最后一个元素，`[start...end]`不包含最后面的元素。

	a = [1,3,5,7,9]
	a[1..3]               >>	[3,5,7]
	a[1...3]              >>	[3,5]
	a[3..3]               >>	[7]
	a[-3..-1]             >>	[5,7,9]

`[ ]`运算符有相应的`[ ]=`运算符来修改数组的元素的值。数组中的间隙元素默认填充`nil`值。
	
	a = [1,3,5,7,9]       >>	[1,3,5,7,9]
	a[1] = 'bat'          >>	[1,"bat",5,7,9]
	a[-3] = 'cat'         >>	[1,"bat","cat",7,9]
	a[3] = [9,8]          >>   [1,"bat","cat",[9,8],9]
	a[6] = 99             >>	[1,"bat","cat",[9,8],9,nil,99]
	
如果`[ ]=`的索引是两个数（开始和长度）或者是范围，那么数组中原来的元素讲会被右边的值替代。如果长度为0，则右边的值将会插入开始所在的位置；如果右边的值本身就是数组，就会替换掉左边标志的位置的值。数组的长度更加右边的值自动变化。

	a = [1,3,5,7,9]      >>		[1,3,5,7,9]
	a[2,2] = 'cat'       >>		[1,3,"cat",9]
	a[2,0] = "dog"       >>		[1,3,"dog","cat",9]
	a[1,1] = [9,8,7]     >>		[1,9,8,7,"dog","cat",9]
	a[0..3] = []         >>		["dog","cat",9]
	a[5] = 99            >>    ["dog","cat",9,nil,nil,99]
	
数组还有大量的方法，你可以把数组看成是集合，序列，数列等，完整的数组方法列表查看文档。

####字典

字典和数组类似，不过字典的索引可以是任意类型。

	h = { 'dog' => 'canine', 'cat' => 'feline', 'donkey' => 'asinine' }
	h.length     >>		3
	h['dog']     >>		“canine”
	h['cow'] = 'bovine'
	h[12] = 'dodecine'
	h['cat'] = 99
	h  >> {"cow"=>"bovine","cat"=>99,12=>"dodecine","donkey"=>"asinine","dog"=>"canine"}
	
和数组相比，字典有一个缺点就是它的对象是没有顺序的。

####实现**”SongList“”**类

在研究完`Array`和字典之后，我们有了头绪了实现`SongList`。我们需要添加几个基础的方法：

* append(aSong)   >> list     #追加一首歌
* deleteFirst()   >> aSong    #删除第一首歌，并返回被删除的歌曲
* deleteLast()    >> aSong    #删除最后一首歌，并返回被删除的歌曲
* [anIndex]       >> aSong    #返回特定位置的歌曲

为了支持上面列出的方法我们可以用数组来存储歌曲列表，因为数组是有序的，并且可以通过索引来访问。

但是，我们还想通过歌名来搜索一首歌，这可能需要字典。我们可以使用字典来存储歌曲列表吗？可以，但是有几个问题，首先是字典是没有顺序的，为了实现有序需要提供额外的工作，另外一个更大的问题是，字典不支持同样的元素出现，但是一首歌可以在播放列表中出现多次。鉴于这些问题，还是选择用数组来做，当需要通过歌名来搜索歌曲的时候，只需要遍历一次就好，如果这导致性能问题，我们通常可以提供基于hash的方法来搜索。

我们从`initialize`方法开始实现我们的`SongList`类，它只是创建一个用来存储歌曲的数组。

	class SongList
		def initialize
			@songs = Array.new
		end
	end

`SongList#append`方法添加歌曲到`@songs`的尾部，并返回`@songs`自己，这样就可以链式的调研`append`来添加多首歌曲，后面将给出例子。

	class SongList
		def append(aSong)
			@songs.push(aSong)
			self
		end
	end

继续实现`deleteFirst`和`deleteLast`方法，直接用`Array`的`shift`方法和`pop`方法。

	class SongList
		def deleteFirst
			@songs.shift
		end
		
		def deleteLast
			@songs.pop
		end
	
	end

写些代码测试一下：

	list = SongList.new
	list.
	   append(Song.new('title1', 'artist1', 1)).
	   append(Song.new('title2', 'artist2', 2)).
	   append(Song.new('title3', 'artist3', 3)).
	   append(Song.new('title4', 'artist1', 4))
	   
	list.deleteFirst	>>	Song: title1--artist1 (1)
	list.deleteFirst	>>	Song: title2--artist2 (2)
	list.deleteLast   	>>	Song: title4--artist4 (4)
	list.deletelast  	>>	Song: title3--artist3 (3)
	list.deletelast 	>>	Song: nill
	   
它正确的运行了。

下一个方法是`[ ]`,通过索引来访问歌曲列表（我们会使用`Object#kind_of?`方法）。

	class SongList
		def [](key)
			if key.kind_of?(Integer)
				@songs[key]
			else
				# ...
			end
		end
	end
	
测试一下：

	list[0]          >> Song: title1--artist1 (1)
	list[2]          >> Song: title3--artist3 (3)
	list[9]          nil


现在是时候实现通过歌名来搜索歌曲了，我们将通过遍历歌曲列表来搜索，在实现之前，有一个知识点我们需要掌握的：迭代器。

###Block和迭代器

实现遍历歌曲列表来搜索歌曲：

	class SongList
		def [](key)
			if key.kind_of?(Integer)
				return @songs[key]
			else
				for i in 0..@songs.length
					return @songs.[i] if key==@songs[i].name
				end
			end
			return nil
		end
	end
	
上面的代码应该很容易理解，用到了`for`循环

数组有一个`find`方法用于查找元素，我们可以利用它：

	class SongList
		def [](key)
			if key.kind_of?(Integer)
				result = @songs[key]
			else
				result = @songs.find { |aSong| key == aSong.name}
			end
	end
	
我们之前提到过，如果`if`语句只有一句，则可以使用简化的方式：

	class SongList
		def [](key)
			return @songs[key] if key.kind_of?(Integer)
			return @songs.find { |aSong| aSong.name ==key }
		end
	end

`find`是一个迭代器－－－一个可以重复调用代码块的方法。迭代器和block是ruby最有趣的特性，所以我们花点时间来看一下它们是怎么工作的。

####实现迭代器

ruby的迭代器是可以调用block代码段的方法。初咋之下，ruby的block和c，java，perl语言的block类似，其实是不同的，block是组织代码语句的一种形式。

首先，block只能出现在一个方法调用的后面，与最后一个参数同一行。然后，在第一次遇到block代码的时候并不马上运行，而是ruby记住该block所出现的上下文（局部变量、当前对象等），进入调用方法之后在适当时候调用，接着就是block奇迹发生的时候。

在调用方法里面，通过`yield`语句来调用关联block,就好像是调用另外一个方法一样。每一个`yield`语句都会触发一次关联block，block运行结束后返回对应的`yield`位置，继续运行调用方法后面的语句。下面看一个例子：

	def threeTimes
		yield 
		yield
		yield
	end
	
	threeTimes { puts "	Hello" }
	
运行结果：

	Hello
	Hello
	Hello
	
这个block( `{ puts "Hello"}` )会和`threeTimes`这个调用方法关联起来。在调用方法里面，`yield`调用了3次，所以关联的block也被调用了3次。更有趣的是，你可以给block传递参数，也可以接收它的返回值。举个例子，我们写一个方法来打印Fibonacci序列，给定一个终结值。

	def fibUnTo(max)
		i1, i2 = 1, 1     ＃并行赋值语句
		while i <= max
			yield i1
			i1, i2 = i2, i1+i2
		end
	end

	fibUpTo(1000) { |f| print f, ""}
	
	
运行结果：
	
	1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
	
在这个例子中，`yield`有一个参数，这个参数将会传递给关联block。在block的定义中，参数列表出现在`| |`内。在这个例子中，变量`f`接收`yield`传递的值（在这个例子，出现了并行赋值语句，以后再讲）。通常只传输一个参数给block，但是这不是必须的，你可以传递任意个参数。如果block需要的参数和传递的参数在数量上不匹配会发生什么呢？无巧不成书，参数匹配规则和并行赋值匹配规则大同小异（不同的是：如果block只需要一个参数，那么传入多个参数会先转换为数组）。

block的参数有可能是局部变量，如果是这样的话，block返回之后新值将会被局部变量引用。在某些时候，这会引发问题。但是这会获得额外的性能回报，因为使用了已存在的变量，不需要再开一个空间。

block可以有返回值，类似地，最后一句语句的值作为返回值。这就解释了`Array`的`find`方法。其实`find`方法是在`Enumerable`模块定义，给模块在`Array`类中揉合(mixin)。它可能的实现如下：

	class Array
		def find
			for i in 0...size
				value = self[i]
				return value if yield(value)
			end
			return nil
		end
	end
	
	[1, 3, 5, 7, 9].find { |v| v*v >30 }    >> 7

有一些常见的迭代器已经在ruby的集合类中有了定义。我们已经熟悉了`find`,还有2个常见而简单的：`each`和`collect`。

	[1,3,5].each { |i| puts i}

运行结果：

	1
	3
	5

`collect`会把集合的每一个元素传递给block，并把每一个元素的处理结果收集，返回一个新的数组：

	["H","A","L"].collect { |x| x.succ }      >> ["I","B","M"]
	

###Ruby与C++、Java对比

用一个小节来比较Ruby与C++、Java在实现迭代器上异同是值得的。在ruby中，迭代器是一个和其他方法一样的方法。没有必要新建辅助类来实现迭代器。写ruby代码，你只需要关注你的问题，而不用写代码来支持ruby语言。

迭代器不仅可以访问数组和字典的数据，ruby的输入输出类也运用了迭代器来返回I/O流的每一行（每字节）。

	f = File.open("testfile")
	f.each do |line|
		print line
	end
运行结果：

	This is line one
	This is line two
	This is line three
	And so on ...

我们最后再看一个迭代器的实现。`Smalltalk`语言也支持迭代器，如果你写一个数组求和的程序，你可以使用一个叫`inject`的方法。

	sumofValue				"Smalltalk method"
	   ^self values
	         inject:0
	         into: [ :sum :element | sum+element value]

解释一下：关联block第一次被调用的时候，`sum`设置为`inject`的参数(初始化为0),`element`的值为数组的第一个元素。在后面的每一次运行，`sum`被设置为block上次运行的结果，`element`设置为数组的下一个元素。block最后运行的结果作为返回值。

ruby中没有`inject`方法，不过我们可以自己写一个。我们把它添加到`Array`类中，在后面我们会完善这个方法。

	class Array
		def inject(n)
			each { |value| n = yield(n,value) }
			return n
		end
		
		def sum
			inject(0) { |n, value| n + value}
		end
		
		def product
			inject(1) { |n,value| n*value }
		end
	
	end
	
	[1,2,3,4,5].sum              >> 15
	[1,2,3,4,5].product          >> 120
	
尽管block总是用来实现迭代器，其实它还有其他的用法，我们可以来看看。

###Block作为事务

block可以用来定义一段代码段，在发生特定事件后再运行。如，你会经常打开文件，操作完后，需要确保关闭打开的文件。你可以用平常的做法，直接调用相关方法。这一节我们讲介绍用block来做，一个比较简单的做法如下（不做错误处理）：

	class File
		def File.openAndProcess(*args)
			f = File.open(*args)
			yield f
			f.close()
		end
	end	
	

	File.openAndProcess("testfile","r") do |aFile|
		print while aFile.gets
	end
	
运行结果：

	This is line one
	This is line two
	This is line three
	And so on ...

上面的例子运用了几个技巧。`openAndProcess`方法是一个类方法，它的参数和`File.open`一致，但是我们并不没有对它作更多的处理，直接传给了`File.open`。我们定义参数为`*args`（注意前面添加了`*`），这代表我们想把传入的所有参数装进一个数组里面。

一旦文件被打开了，`openAndProcess`就调用`yield`，并把打开的文件标志符传入block。但block返回时，把文件关闭。

还有，这个例子用`do..end`来定义block，这种方式和用`{ }`的唯一区别就是它们的优先级不同：`do..end`的优先级比`{ }`低。我们将在后面讨论这个问题。

ruby提供的让文件来管理自己的生命周期是非常有用处的。如果`File.open`有关联的block，那么会运行该block，并在blcok返回后把文件关闭；如果没有关联block则返回打开文件的标志符。这用了`Kernel::block_given?`方法，该方法会根据单前方法是否有关联block返回`true`或则`false`。我们可以实现`File.open`，如下：

	class File
		def File.myOpen(*args)
			aFile = File.open(*args)
			# If there's a block, pass in the file and close
			# the file when it returns
			if block_given?
				yield aFile
				aFile.close
				aFile = nil
			end
			retn aFile
		end
	end
	
###block作为闭包

还记得我们的点唱机吧，我们来完善一下它，写一些"用户界面"的代码－－按钮（用来暂停和开始）。我们会用block来做，首先假设我们的硬件制造商实现了基础的按钮类`Button`。
	
	bStart = Button.new("Start")
	bPause = Button.new("Pause")

我们继承`Button`来实现按钮的按键方法：
	
	class StartButton < Button
		def initialize
			super("Start")
		end
		
		def butttonPressed
			# do start actions...
		end
		
	end
	
	bStart = StartButton.new
	
上面的做法有2个问题。第一，这将会子类爆炸，有多少个按钮就需要多少个子类，如果按钮的界面发生了变化，就要修改子类。
第二，按钮发生的动作的层次有问题，按钮的任务不是按钮的特性，而是点唱机的特性。我们可以用blcok来修正这些问题。

	class JukeboxButton < Button
		def initialize(label, &action)
			super(label)
			@action = action
		end
		def buttonPressed
			@action.call(self)
		end
	end
	
	bStart = JukeboxButton.new("Start") { songList.start }
	bPause = JukeboxButton.new("Pause") { songList.pause }

上面方法的关键点时`initialize`方法的第二个参数。如果一个方法的定义的最后一个参数名以`&`开头，那么ruby会搜索方法调用的关联blcok，并把关联block转换为`Proc`类的对象，然后传给方法内部。在我们的例子中，我们把关联block赋值给`@action`变量。当调用`buttonPressed`方法时，我们使用`Proc`类的`call`方法来调用关联block。

一个`Proc`对象里面具有什么内容呢？除了一段关联代码之外，还有block所在的上下文，包括`self`值，方法，变量和常量。神奇的地方在于，即使在block被定义的地方的环境已经不可见了，block依然可以使用上下文所有的内容。在其他语言里，这特性叫闭包。

举个例子：

	def  nTimes(aThing)
		return proc { |n| aThing * n}
	end
	
	p1 = nTimes(23)
	p1.call(3)                >>   69
	p1.call(4)                >>   92
	
	p2 = nTimes("Hello ")
	p2.call(3)                >>   "Hello Hello Hello"
	

方法`nTimes`返回一个`Proc`对象，这个对象引用了`aThing`这个参数。即使在后面的调用中这个参数已经不可见了，却不影响block的使用。
	
 