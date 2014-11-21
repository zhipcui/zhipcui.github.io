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

