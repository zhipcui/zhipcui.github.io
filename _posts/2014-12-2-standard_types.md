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

我们再用50页来讨论`String`类的方法也说不完，感兴趣的可以查看文档。下面我们继续我们的标准类型的讨论，下一个是`Range`(范围)类型。


###Ranges

表示范围的值经常得到使用，如从一月到十二月，0到9，第50行到67行等。如果ruby有助于我们构建现实世界模型的话，那么支持表示范围的数据类型那是再自然不过的事情了。事实上ruby的范围类型做了更多的扩展，它使用范围实现3种独特的特性：序列，条件，和间隔。

####range作为序列

range(范围)最主要的作用应该是用来表示一个序列了。序列有一个起始值、终止值和生成连续值的方法。在ruby中，你使用`..`或`...`范围操作符来声明一个序列。2个点的操作符包含最大值，3个点的操作符不包含最大值。

	1..10
	'a'..'z'
	0...anArray.length

在ruby中，范围值不是以数组的形式存储，而是是一个`range`对象，保存了2个`Fixnum`对象的引用。如果你需要把范围的值存到数组中，要用`to_a`方法转换为数组类型。

	(1..10).to_a		    >> [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
	('bar'..'bat').to_a	    >> ["bar", "bas", "bat"]

'Range'实现了多个方法来迭代它的元素。

	digits = 0..9
	
	digits.include?(5)	»	true
	
	digits.min      >> 0
	
	digits.max      >> 9
	
	digits.reject { |i| i < 5 }     >> [5, 6, 7, 8, 9]
	
	digits.each do |digit|
		dial(digit)
	end

到目前为止，我们表示的范围值都是数字或者字符串。对象也可以用来表示范围，但是要求在你定义的类中实现2个方法：`succ`与`<=>`。`succ`方法用来返回序列的下一个值；`<=>`方法用来实现对象的比较，`<=>`也是比较运算符，根据第一个值小于、等于、大于第二个值而对应返回-1,0,1。

下面的例子实现用'#'符号来表示声音大小，我们可以用来实现点唱机的声音调节器。

	class VU
	
		include Comparable
		
		attr :volume
		
		def initialize(volume)  # 0..9
    		@volume = volume
    	end
    	
    	def inspect
    		'#' * @volume
    	end
    	
    	# Support for ranges
    	def <=>(other)
    		self.volume <=> other.volume
    	end
    	
    	def succ
    		raise(IndexError, "Volume too big") if @volume >= 9
    		VU.new(@volume.succ)
    	end
    end

测试一下：

	medium = VU.new(4)..VU.new(7)
	medium.to_a	     >> [####, #####, ######, #######]
	medium.include?(VU.new(3))	   >> false
	
####Range作为条件

范围可以用来做条件语句，例如，下面的代码段是把从标准输入的数据输出，条件就是输入的数据首行要包含"start"词，末行包含"end"这个词语。

	while gets
		print if /start/../end/
	end
	
我们将在后面讲循环语句是给出更多的例子。

####Range作为中间值判断

范围最后一个运用时作为中间只判断，可以看看某个值是不是在某一范围内，使用的是`===`运算符。

	(1..10)    === 5	     >> true
	(1..10)    === 15	     >> false
	(1..10)    === 3.14159	 >>	true
	('a'..'j') === 'c'	     >> true
	('a'..'j') === 'z'	     >> false

`case`语句会运用到Range的这个用法。


###正则表达式 

会想一下上次我们提到的从文件中创建歌曲列表时，我们用了一个正则表达式来匹配各个域之间的分隔符。我们声明的语句`line.splint(/\s*\|\s*/)`会匹配到`|`和包围其之外多个的空格符。我们现在来探索一下正则表达式是怎么工作的。

正则表达式是匹配字符串的模式。ruby原生支持正则表达式，这一小节将会学习到ruby的正则表达式的所有主要特性。

正则表达式是`Regexp`类的对象。可以通过显示的构造器生成，也可以用`/pattern/`或`%r\pattern\`的形式。

	a = Eegexp.new('^\s*[a-z]')     >> /^\s*[a-z]/
	b = /^\s*[a-z]/                 >> /^\s*[a-z]/
	c = %r{^\s*[a-z]}               >> /^\s*[a-z]/
	
一旦你拥有一个正则表达式对象，你可以用它与字符串对象来做匹配，使用`Regexp#match(aString)`方法，或者匹配操作符`=~`与匹配操作符的否`!~`。匹配操作符在`String`和`Regexp`对象中都有定义。如果2个字符串做匹配的话，右边的字符串讲转换为正则表达式。

	a = "Fats Waller" 
	a =~ /a/                >> 1
	a =~ /z/                >> nil
	a =~ "ll"				 >> 7
	
匹配操作符会返回匹配到的字符位置。这些操作符会有边缘效果，意思是有把特定的值赋给ruby的全局变量。" $& "保存了匹配的字符串；“ $｀ ”保存了匹配位置之前的字符串，" $' "保存了匹配位置之后的字符串。我们可以写一个方法`showRE，显示匹配位置在哪里。


	def showRE(a,re)
  		if a =~ re
    	"#{$`}<<#{$&}>>#{$'}"
  		else
    		"no match"
  		end
	end
	showRE('very interesting', /t/)	»	very in<<t>>eresting
	showRE('Fats Waller', /ll/)	»	Fats Wa<<ll>>er
	
匹配操作工程还会设置几个线程全局的变量:`$~`,`$1`~`$9`。`$~`变量是一个`MatchData`类的对象，其中包含了所有有关匹配的信息。`$1`和后面其他的变量保存了多个匹配的信息。我们将在后面见到这些变量的运用。


####模式

每一个正则表达式都包含一个模式，描述了匹配规则。

在正则表达式的模式中，除了`.`,`|`,`(`,`)`,`[`,`{`,`+`,`\`,`^`,`$`,`*`,`?`这些字符之外，其他的字符都会表示匹配自己。
	
	showRE('kangaroo',/angar/)    >> k<<angar>>oo
	showRE('!@%&-_=+',/%&/)       >> !@<<%&>>-_=+
	
如果你想匹配上面这些特殊字符，你得使用相应的转义字符。这就解释了为什么我们在从文件中获取歌曲列表时用的正则表达式`/\s*\|\s*/`的`|`前加上`\`符号了。`\|`代表要匹配`|`符号。

	showRE('yes | no', /\|/)	>>	yes <<|>> no
	showRE('yes (no)', /\(no\)/)	>>	yes <<(no)>>
	showRE('are you sure?', /e\?/)	>>	are you sur<<e?>>

另外，正则表达式也可以使用`#{...}`来插入表达式的值。


####锚点

默认情况下，正则表达式会返回第一个匹配的字符串。`/iss/`会匹配`"Mississippi"`的第一个"iss"，并返回位置1。如果你想匹配字符串的最后一个或者第一个呢？

`^`、`$`分别匹配行首、行末。所以你可以使用它们来做锚点，标志你想匹配行首还是行末。比如，`/^option/`只会匹配出现在行首的`option`。 `\A`表示匹配字符串的起始，`\z`或者`\Z`匹配字符串的结束。（如果字符串以换行符`\n`结束的话，`\Z`会匹配`\n`前面的字符）

	showRE("this is\nthe time", /^the/)	»	this is\n<<the>> time
	showRE("this is\nthe time", /is$/)	»	this <<is>>\nthe time
	showRE("this is\nthe time", /\Athis/)	»	<<this>> is\nthe time
	showRE("this is\nthe time", /\Athe/)	»	no match
	
类似地，`\b`和`\B`分别匹配单词的边界与非边界。单词字符意思是指26字母，数字还有下划线。

	showRE("this is\nthe time", /\bis/)	»	this <<is>>\nthe time
	showRE("this is\nthe time", /\Bis/)	»	th<<is>> is\nthe time
	
####字符集合

一个字符集合是在`[ ]`里面的一推字符，可以匹配集合的任意一个字符。`[aeiou]`表示匹配一个元音字母。在字符集合中，`.`,`|`,`(`,`)`,`[`,`{`,`+`,`\`,`^`,`$`,`*`,`?`这些字符的作用会消失，只可以作为普通字符。但是，转义字符还是有效果的。`\b`表示退格键，`\n`表示换行。`\s`表示空格(空格，tab等)。

	showRE('It costs $12.', /[aeiou]/)	 >> It c<<o>>sts $12.
	showRE('It costs $12.', /[\s]/)     >> It<< >>costs $12.

在方括号内，`c1-c2`表示从`c1`到`c2`的所有字符。

如果你想在在字符集合中包含`]`和`-`,必须把它们放到第一个位置上。

	a = 'Gamma [Design Patterns-page 123]'
	showRE(a, /[]]/)	>>	Gamma [Design Patterns-page 123<<]>>
	showRE(a, /[B-F]/)	>>	Gamma [<<D>>esign Patterns-page 123]
	showRE(a, /[-]/)	>>	Gamma [Design Patterns<<->>page 123]
	showRE(a, /[0-9]/)	>>	Gamma [Design Patterns-page <<1>>23]

如果`^`是字符集合的第一个字符，那么表示取非。`[^a-z]`表示任一非小写字母。

有一些集合比较常用，所以ruby提供了它们的缩写。看下表。

	showRE('It costs $12.', /\s/)	>>	It<< >>costs $12.
	showRE('It costs $12.', /\d/)	>>	It costs $<<1>>2.
	
**字符集合缩写表**：

缩写          | 集合格式       | 描述
------------ | ------------- | ------------
\d           | [0-9]         | 数字字符
\D           | [^0-9]        | 非数字字符
\s           | [\s\t\r\n\f]  | 空白字符
\S           | [^\s\t\r\n\f] | 非空白字符
\w           | [A-Za-z0-9_]  | 单词字符
\W           | [^A-Za-z0-9_] | 非单词字符

最后，在中括号外，`.`可以表示任一字符，除了换行符（在多行模式下也可以表示换行符）。


	a = 'It costs $12.'
	showRE(a, /c.s/)	>>	It <<cos>>ts $12.
	showRE(a, /./)	    >>	<<I>>t costs $12.
	showRE(a, /\./)	    >>	It costs $12<<.>>

####重复

当我们想从文件中解析出歌曲的时候，用了正则表达式`/\s*\|\s*/`，由上面的表知道，`\s`表示空白字符，那么后面的`*`应该是表示多个特定字符的意思。事实上，`*`是数次限定标志的符号之一，说明前面的字符可以出现的次数。

设`r`为出现在正则表达式的符号，那么：

	r*       匹配0个或者更多的字符r
	r+       匹配1个或者更多的字符r
	r?       匹配0个或者1个字符r
	r{m,n}   匹配最少m个，最多n个字符r
	r{m,}    匹配最少m个字符r

这些表示重复的标志符号有比较高的优先级－－它只是绑定它的前面的那个字符。/ab+/匹配由一个a，后面接着是大于等于1个b的字符串，而不是多个ab的字符串。你要小心`*`的使用，`/a*`可以匹配任意的字符串，因为任意的字符串都可以匹配有0个或多个`a`。

默认情况下，这些模式会尽可能多的匹配字符串，这就是称它们为贪婪地的原因。你可以修改默认行为，通过添加`?`可以设置匹配最少。

	a = "The moon is made of cheese"
	showRE(a, /\w+/)	         >> 	<<The>> moon is made of cheese
	showRE(a, /\s.*\s/)	         >>	    The<< moon is made of >>cheese
	showRE(a, /\s.*?\s/)         >> 	The<< moon >>is made of cheese
	showRE(a, /[aeiou]{2,99}/)	  >>	The m<<oo>>n is made of cheese
	showRE(a, /mo?o/)	          >>	The <<moo>>n is made of cheese

####选择

我们知道`|`是个特殊的符号，因为在我们划分行内各个域的值的时候我们在`|`符号上加上了反斜线来表示符号本身（这叫做转义字符）。如果没有反斜线的话，出现`|`的模式代表了匹配符号之前的内容或者之后的内容。


	a = "red ball blue sky"
	showRE(a, /d|e/)	              >>	r<<e>>d ball blue sky
	showRE(a, /al|lu/)	              >>	red b<<al>>l blue sky
	showRE(a, /red ball|angry sky/)	   >>	<<red ball>> blue sky

这里有一点要特别注意的，因为`|`的优先级比较低，所以最后一个例子会匹配`red ball`或者`angry sky`,而不是`red ball sky`或则会`red angry sky`，如果你要表示这个意思的话，你需要用到下面讲到的内容：分组。

####分组

在表达式中你可以用括号来改变匹配的优先级，也就是分组，一个组会看成是一个单独的表达式。


	showRE('banana', /an*/)     >>   b<<an>>ana
	showRE('banana', /(an)*/)   >>   <<>>banana
	showRE('banana', /(an)+/)   >>   b<<anan>>a
	
	a = 'red ball blue sky'
	showRE(a, /blue|red/)        >>  <<red>> ball blue sky
	showRE(a, /(blue|red) \w+/)	  >>  <red ball>> blue sky
	showRE(a, /(red|blue) \w+/)	  >>  <<red ball>> blue sky
	showRE(a, /red|blue \w+/)    >>  <<red>> ball blue sky

	showRE(a, /red (ball|angry) sky/)    >>  no match
	a = 'the red angry sky'
	showRE(a, /red (ball|angry) sky/)    >>  the <<red angry sky>>
	
使用分组来匹配还有一个特殊的作用，那就是匹配到的每一个分组的结果都会存储在一个特殊的全局变量中。ruby是根据`(`所出现的位置来给分组标号，然后将相应的结果存储到相应的位置上去。你可以在模式中和ruby代码中使用匹配结果的变量。在模式表达式中`\1`表示第一个分组的结果,	`\2`表示第二个，以此类推。在代码中，可以使用全局变量`$1,$2,..`。

	"12:50am" =~ /(\d\d):(\d\d)(..)/         >>  0
	"Hour is #$1, minute #$2"                >>  "Hour is 12, minute 50"
	"12:50am" =~ /((\d\d):(\d\d))(..)/       >>  0
	"Time is #$1"                            >>  "Time is 12:50"
	"Hour is #$2, minute #$3"                >>  "Hour is 12, minute 50"
	"AM/PM is #$4"                           >>  "AM/PM is am"

使用这些特殊的变量可以让你搜索字符串中的重复子串。


	# match duplicated letter
	showRE('He said "Hello"', /(\w)\1/)      >>  He said "He<<ll>>o"
	# match duplicated substrings
	showRE('Mississippi', /(\w+)\1/)         >>  M<<ississ>>ippi
	
	
	showRE('He said "Hello"', /(["']).*?\1/)  >>  He said <<"Hello">>
	showRE("He said 'Hello'", /(["']).*?\1/)  >>  He said <<'Hello'>>


####基于模式的替换

####




