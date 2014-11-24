###Programming Ruby:类，对象和变量


我们之前多次说ruby是一个完全的面相对象语言，到底怎么完全法呢？学习这一章就知道了。这一章我们将学习ruby面相对象的内容，你会了解到ruby在某些方面比其他的面相对象语言强多了。我们会使用点唱机`(jukebox)`这个模型，贯穿整篇内容来写代码例子。

我们需要创建一个`Song`类，一首歌会有歌名、演唱者、时间等，我们会把这些内容放进`Song`类里面。类名的开头应该是大写字母，这个我们在上一章提到过的。

	class Song
		def initialize(namm, artist, duration)
			@name     = name
			@artist   = artist
			@duration = duration
		end
	end

`initialize`是一个特殊的方法。当你通过`Song.new`来创建一个新的`Song`对象的时候，ruby会创建一个未初始化的对象，然后调用`initialize`方法来做初始化工作，传给`new`的参数会传递给`initialize`。所以你可以在`initialize`方法中设置对象的初始状态。

对于`Song`类，`initialize`方法接受3个参数，参数可以像本地变量一样使用，以小写字母开头。

每一个`Song`对象代表代表一首不同的歌，所以需要实例变量来存储歌的信息。在ruby中，实例变量是以`@`开头的变量，如上面例子中的`@name, @artist, @duration`。

测试一下我们的类：

	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.inspect    >> "# <Song:0x401b4924 @duration=260, @artist=\"Fleck\", @name=\"Bicylops\">"

`inspect`方法默认会把输出对象的id和实例变量。

根据以往的编程经验来看，我们有时需要多次查看对象的内容，但是`inspect`方法的默认格式不是那么人性化，我们可以通过ruby的标准方法`to_s`来定制我们需要的信息和格式。

	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.to_s              >> "# <Song:0x401b4924>"

`to_s`默认也只是输出对象的id,我们可以在类中重载它来定制我们想到的信息。

在ruby中，你可以随时给已定义的类添加方法，不管这个类是你自己定义的还是ruby内建的。添加的方法对之前已经创建的对象也是有作用的。

	class Song
		def to_s
			"Song: #{@name}--#{@artist} {#{@duration}}"
		end
	end
	
	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.to_s                 >> "Song: Bicylops--Fleck (260)"
	

也许你会迷惑`to_s`、`inspect`这些方法是怎么回事，答案是继承，这是下一节的内容。

###继承和消息

你可以通过继承来细化某一个类，比如，我们的点唱机(jukebox)有歌曲的概念，所以我们定义了`Song`类。现在我们想让我们的点唱机实现卡拉ok的功能，一首用于卡拉ok的歌和普通的歌类似，只是没有原唱，然后还会有歌词。我们需要定义卡拉ok曲子的类`KaraokeSong`，但我们不需要从新来，因为`Song`类有了一些基础的东西，我们只需要继承它就有了这些基础。

	class KaraokeSong < Song
		def initialize(name, artist, duration, lyrics)
			super(name, artist, duration)
			@lyrics = lyrics
		end
	end
	
`"< Song"`声明`KaraokeSong`是继承`Song`的，现在先不用关注`initialize`方法里面的内容，一会我们会说到`super`这个东西。

生成一个`KaraokeSong`对象来测试一下我们代码。(在最终的结果里面，lyrics包含了歌词的文本和时间信息，为了测试方便，现在我们先把它看做字符串来处理。这就是非强类型语言的好处，不用先于说明数据的类型)

	aSong = KaraokeSong.new("My Way", "Sinatra", 225, "And now, the...")
	aSong.to_s           >> "Song: My Way--Sinatra (225)"
	
它运行起来了，不过`to_s`没有把`lyrics`的信息打印出来。

这和ruby方法调用搜索路径有关，在编译的阶段，ruby并不知道去哪里找`to_s`方法，直到运行的时候，ruby才决定运行哪一个。首先它会找到`aSong`所属的类(即`KaraokeSong`),如果这个类实现了这个方法那就会调用，否则它会按顺序搜索该类的父类(即`Song`),父类的父类,...直到找到为止，如果最终都没有找到就会报错。（你可以拦截这个错误，并按你的方式来处理错误，异常处理将再后面的章节讲到）

回到我们的例子，我们给`aSong`发送了`to_s`的消息，而`aSong`是`KaraokeSong`类的对象，所以ruby会在`KaraokeSong`类中搜索`to_s`方法的实现，可是我们没有实现这个方法，所以ruby找不到。接着，ruby会搜索`KaraokeSong`的父类`Song`，然后找到了该方法的实现，然后调用了`Song` 的`to_s`方法（我们在上一章中定义了该方法）。这就是为什么没有把`lyrics`的内容输出来的原因，因为`Song`类根本就不知道`lyrics`的存在。

我们可以通过在`KaraokeSong`类中实现`to_s`方法来解决这个问题。有几种方法来实现，我们先用一个比较'坏'的方法来实现。

	class KaraokeSong
		# ...
		def to_s
			"KS: #{@name}--#{@artist} (#{@duration})  [#{@lyrics}]"
		end
	end

	aSong = KaraokeSong.new("My Way", "Sinatra", 225, "And now, the...")
	aSong.to_s           >> "KS: My Way--Sinatra (225) [And now, the...]"
	

为什么说这是一种｀坏｀的方法呢？这是一种高耦合的方式，和我们追求低耦合的编程方式背道而驰。直接引用父类的变量有时候会导致一些问题，假设`@duraion`的单位是毫秒，输出毫秒的单位就不是合适了,最好是按照父类的方式来引用父类的变量。

	class KaraokeSong < Song
	 	def to_s
	 		super + " [#{@lyrics}]"
	 	end
	end
上面的方式则是｀好｀的方式。当你引用关键字`super`时，ruby会调用当前对象父类中相同名的方法，并把传给该方法的参数传给父类相应的方法。

我们显示的声明了`KaraokeSong`的父类为`Song`，但是没有指明`Song`类的父类是什么，注意，如果你没有声明一个类的父类，那么ruby会默认父类为`Object`类。这意味着`Object`是所有对象的祖先，`Object`的变量和方法对所有对象都可见。在前面我们提到所有对象可以调用`to_s`方法，因为这个方法是`Object`定义的35个实例方法之一，具体是哪35个，查看个完整的列表。

######多继承、揉合
像c++这些面相对象语言支持多继承，一个类可以有多个直接父类。尽管这技术很强大，但也是危险的，因为多继承的继承层次会变的模糊。而java这种语言只支持单继承，虽然没有了多继承的危险，但是也有缺点，因为现实世界中是存在多继承的，比如一个球既是圆的东西，有时有弹性的东西。

ruby混合了所有你想要的特性，既给你单继承的简洁，又给你多继承的强大功能。

ruby只能有一个直接父类，所以是单继承的语言。但是，你可以用`include`关键字把代码块包含到ruby的类中，就像是继承一样，这就是`揉合`, 这部分内容以后再说。

###对象、属性


######访问属性
`Song`类的对象拥有了它的内部状态（实例变量，也称为属性），但是它们是私有的，即在类外是无法访问和修改的，所以我们需要听过方法来访问它们和修改。

	class Song
	
		def name
			@name
		end
		
		def artist
			@artist
		end
		
		def duration
			@duration
		end
		
	end
	
	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.artist                 >>  "Fleck"
	aSong.name                   >>  "Bicylops"
	aSong.duration               >>  260
	
上面的代码中，我们定义了3个访问方法来访问3个实例变量的值。ruby提供了便捷方式生成访问实例方法:

	class Song
		attr_reader :name, :artist, :duration
	end
	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.artist                 >>  "Fleck"
	aSong.name                   >>  "Bicylops"
	aSong.duration               >>  260
	
	
######修改属性
	
上面是提供访问方法，如果我们要修改实例变量，我们需要提供修改方法。

在c++或者java中，你可能会提供类似下面的方法:

	class JavaSong{                //java 代码
		private Duration myDuration;
		public void setDuration(Duration newDuration){
			myDuration = newDuration;
		}
	}
	s = new JavaSong(...)
	s.setDuration(length)
	
在ruby中，我们的代码更加简介易读：
	
	class Song
	 	def duration=(newDuration)
	 		@duration = newDuration
	 	end
	end
	
	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.duration                 >>  260
	aSong.duration = 257           
	aSong.duration                 >>257
	
"aSong.duration = 257" 会调用`duration=`方法，并把`257`作为参数。

ruby也便捷方式生成修改实例方法。

	class Song
		attr_writer :duration
	end
	
	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.duration = 257

######虚拟属性

先上代码吧！

	class Song
		def durationInMinutes
    		@duration/60.0   # force floating point
    	end
    	def durationInMinutes=(value)
    		@duration = (value*60).to_i
    	end
	end
	
	aSong = Song.new("Bicylops", "Fleck", 260)
	aSong.durationInMinutes	         >>  4.333333333
	aSong.durationInMinutes = 4.2
	aSong.duration	                 >>  252

我们并没有声明`durationInMinutes`实例变量，但是我们却可以通过提供`durationInMinutes`、`durationInMinutes=`方法，就好像有`durationInMinutes`变量一样来使用，这就是虚拟属性。


###类变量和类方法

