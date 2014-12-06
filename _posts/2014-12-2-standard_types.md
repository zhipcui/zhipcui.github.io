---
layout: post
title:  "Programming Ruby: 标准类型"
date:   2014-12-2
categories: Ruby
tag: Programming Ruby

---

前面我们已经写了一些代码，讨论了数组、字典和block，但是一直都没有讨论ruby的标准类型，如数字、字符串、范围、正则表达式等，现在我们花点时间来看看这些内容。

###数字

ruby支持整型数和浮点数。整型数长度是任意的（由你的系统可用内存限制）。在特定范围内（通常是－2^30~2^30或－2^62~2^62)，整型以二进制的格式存在，是`Fixnum`类的对象。在这范围之外的整型将以`Bignum`类的对象存储。存储的过程是透明的，ruby会自动完成转换，你不需要了解其中的细节，有了这个妈妈再也不用担心数字益处了。

	num = 8
	7.times do
		print num.type, " ", num, "\n"
		num *= num
	end
	
运行结果：
	
	Fixnum 8
	Fixnum 64
	Fixnum 4096
	Fixnum 16777216
	Bignum 281474976710656
	Bignum 79228162514264337593543950336
	Bignum 6277101735386680763835789423207666416102355444464034512896

你可以给整型数提供一个前缀来标注是什么进制，（0：八进制，0x：16进制，0b：二进制）。下划线会被忽略。

	123456                          # Fixnum
	123_456                         # Fixnum (下划线会被忽略)
	-543                            # 负Fixnum
	123_456_789_123_456_789         # Bignum
	0xaabb                          # 16进制
	0377                            # 八进制
	-0b101_010                      # 二进制
	
所有的数字都是对象，都有一堆的方法。所以不像c＋＋这些语言，求一个值的绝对值用aNumber.abs,而不是abs(aNumber)。

整型数支持几个有用的迭代器，我们在上面的例子中已经用了这个`(7.times)`。其他的还有`upto`、`downto`、`step`等。

	3.times        { print "x" }
	1.upto(5)      { |i| print i, "" }
	99.downto(95)  { |i| print i, "" }
	50.step(80, 5) { |i| print i, "" }
	
运行结果：
	
	X X X 1 2 3 4 5 99 98 97 96 95 50 55 60 65 70 75 80
	
有一点要特别注意的是，包含数字的字符串不会自动的转换为数字，如果你对其作运算，将得不到你想要的结果。这问题经常出现在从文件读入数字。假设有一个文件，文件内容如下:

	3 4
	5 6
	7 8
	
我们的处理代码如下：

	DATA.each do |line|
		vals = line.split
		print vals[0] + vals[1]
	end
	
这样的输出结果是`34 56 78 `，而不是我们想要的`7 11 15`。原因在于输入的是字符串，而不是数字，我们需要用`to_i`方法把字符串转换为数字类型。

	DATA.each do |line|
		vals = line.split
		print vals[0].to_i + vals[1].to_i
	end
	
运行结果:

	7 11 15
	
	
###字符串

ruby的字符串可以存可打印字符，也可以存有二进制数据。字符串是`String`类的对象。

你可以在字符串中使用转义字符，如`\\`表示`\`。

	'escape using "\\" '               >> escape using "\"
	'That\'s rignt'                    >> That's right
	
由`" "`所构建的字符串可以插入运算表达式或者变量（ `#{ expr }`将被替换）。如果替换的变量是全局变量、类变量、和实例变量，那么可以忽略掉括号。

	"Seconds/day: #{24*60*60}"	     >> Seconds/day: 86400
	"#{'Ho! '*3}Merry Christmas"    >> Ho! Ho! Ho! Merry Christmas
	"This is line #$."              >> This is line 3 
	
除了上面提到的方法之外，还有3种方法来构造一个字符串：`%q`,`%Q`,和`"here document"`。这3种方法的关键是给定字符串的起始和终止标识符。

	%q/general single-quoted string/	>> general single-quoted string
	%Q!general double-quoted string!	>> general double-quoted string
	%Q{Seconds/day: #{24*60*60}}	    >> Seconds/day: 86400

`%q`和`%Q`后面的符号将作为字符串的限定符，可以是`{, (, /, *`等，如果是括号类的符号，则相应的关闭符号作为终止标识符，否则相同的符号作为终止标识符。

最后是`"here document"`这种方式，和上面的差不多，不过是以`<<`开始构造。

	aString = <<END_OF_STRING
    	The body of the string
    	is the input lines up to
    	one ending with the same
    	text that followed the '<<'
	END_OF_STRING

`<<`后面的字符串作为终止标识符，一般独自一行。限定符之间的内容作为字符串的值。你可以在`<<`后面添加`-`符号来指明终止标识符。

	print <<-STRING1, <<-STRING2
		Concat
		STRING1
      		enate
      		STRING2

####玩转 ***String*** 类

`String`类可能是最大的ruby内建类，拥有75个标准方法。我们不会每个都讲，（库参考文档有完整的列表），只是讲一些常见的。

让我们回到点唱机这个例子中。即使在有网络的情况下，我们也会把一些流行的音乐下载到本地上，避免因为网络问题，而使得点唱机没有歌曲。但是呢，由于设计问题，我们的歌曲列表存在一个文本类型的文件中，见例子。每行都包含了歌曲文件名，时间，演唱者，歌名，用`|`分割开。

	/jazz/j00132.mp3  | 3:45 | Fats     Waller     | Ain't Misbehavin'
	/jazz/j00319.mp3  | 2:58 | Louis    Armstrong  | Wonderful World
	/bgrass/bg0732.mp3| 4:09 | Strength in Numbers | Texas Red
         :                  :           :                   :

很明显，我们需要用`String`类的方法把这些数据从文件中解析出来，按步骤来完成：

* 把每一行各个域分开
* 歌曲时间格式从 mm:ss 转换为以秒为单位
* 移去名字中多余的空格

第一步是把每一行各个域分开，可以使用`String#split`方法。我们把`/\s*|\s*/`正则表达式传递给`split`方法。这个正则表达式用于确认分割符号。另外因为每行的数据都有换行符，我们需要用`String#chomp`方法把它去掉。

	songs = SongList.new

	songFile.each do |line|
		file, length, name, title = line.chomp.split(/\s*\|\s*/)
		songs.append Song.new(title, name, length)
	end
	puts songs[1]

运行结果：

	Song: Wonderful World--Louis    Armstrong (2:58)
	
如果稍微留意的话，你会发现上面文件中的名字项有多余的空格，这样看起来很丑，所以我们要把多余的空格去掉，只留下一个就够了。有很多方式来做，不过最简单的方式是用`String#squeeze`方法，它可以把重复的字符去掉，而只保留一个。我们将使用`squeeze!`方法来修改字符串。

	songs = SongList.new

	songFile.each do |line|
		file, length, name, title = line.chomp.split(/\s*\|\s*/)
		name.squeeze!(" ")
		songs.append Song.new(title, name, length)
	end
	
	puts songs[1]

运行结果：

	Song: Wonderful World--Louis Armstrong (2:58)

最后，还有一个问题就是转换时间格式问题，文件格式是2:58，但是我们想要显示178秒。我们可以再次使用`split`,分隔符为":" 。

	mins, secs = length.split(/:/)
	
我们使用`String#scan`来替代`split`，它的功能和`split`相似，不过它是基于模式把字符串分块，而不是基于分隔符。在这个例子总，我们想把时间中的数字提取出来，所以我们的模式是匹配多个数字，`/\d+/`。

	songs = SongList.new
	songFile.each do |line|
		file, length, name, title = line.chomp.split(/\s*\|\s*/)
		name.squeeze!(" ")
		mins, secs = length.scan(/\d+/)
		songs.append Song.new(title, name, mins.to_i*60+secs.to_i)
	end
	puts songs[1]
	
运行结果：

	Song: Wonderful World--Louis Armstrong (178)

我们的点唱机有一个关键字搜索的功能。输入一个词语，可以是歌名，歌手名字等，然后列出和该词语相关的歌曲。我们新建一个索引类来实现，提供一个对象，和相关的字符串做索引。其中会展示几个新的`String`方法。

	class WordIndex
		def initialize
    		@index = Hash.new(nil)
    	end
    	def index(anObject, *phrases)
    		phrases.each do |aPhrase|
      		  aPhrase.scan /\w[-\w']+/ do |aWord|   # extract each word
        		aWord.downcase!
        		@index[aWord] = [] if @index[aWord].nil?
        		@index[aWord].push(anObject)
      		  end
    		end
    	end
    	def lookup(aWord)
    		@index[aWord.downcase]
    	end
    end
    
`String#scan`方法把字符串中符合正则表达式的词语解析出来，在这个例子中正则表达式`/\w[-\w']+/`表示词语（词语的字符包括a~z, A~Z, -, '）。正则表达式将在后面讲。为了确保搜索是不区分大小写的，我们用`String#downcase`方法把索引都变为小写格式。留意我们在方法后面的顿号，这和没有添加顿号的方法类似，不同的是它不但会返回结果值，还会修改当前对象的值。［上面的代码例子有一个bug，如果我们把``Gone, Gone, Gone''这首歌放进去，将会存储3次，你要怎么解决它呢？］

我们需要扩展一下`SongList`类，当添加一首歌的时候，也为它们创建相关索引。

	class SongList
		def initialize
    		@songs = Array.new
    		@index = WordIndex.new
    	end
    	def append(aSong)
    		@songs.push(aSong)
    		@index.index(aSong, aSong.name, aSong.artist)
    		self
    	end
    	def lookup(aWord)
    		@index.lookup(aWord)
    	end
    end

好！测试一下：

	songs = SongList.new
	songFile.each do |line|
		file, length, name, title = line.chomp.split(/\s*\|\s*/)
		name.squeeze!(" ")
		mins, secs = length.scan(/\d+/)
		songs.append Song.new(title, name, mins.to_i*60+secs.to_i)
	end
	
	puts songs.lookup("Fats")
	puts songs.lookup("ain't")
	puts songs.lookup("RED")
	puts songs.lookup("WoRlD")

运行结果：

	Song: Ain't Misbehavin'--Fats Waller (225)
	Song: Ain't Misbehavin'--Fats Waller (225)
	Song: Texas Red--Strength in Numbers (249)
	Song: Wonderful World--Louis Armstrong (178)

我们再用50页来讨论`String`类的方法也说不完，感兴趣的可以查看文档。下面我们继续我们的标准类型的讨论，下一个是“范围”类型。

