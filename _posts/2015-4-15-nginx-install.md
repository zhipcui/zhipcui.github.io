


nginx会依赖几个通用的库，若你的系统没有安装，则会提示配置错误

###### PCRE library (用户处理正则表达式)
  
 1. 下载PCRE
 
 			wget http://softlayer-sng.dl.sourceforge.net/project/pcre/pcre/8.36/pcre-8.36.zip
 			
 2. 解压PCRE
 
 			unzip pcre-8.36.zip
 		
 3. 安装PCRE
 			
 			./configure 
 			make
 			make install
 	
###### gcc-c++安装
在安装PCRE时，需要支持对c＋＋的编译，大多数情况下，Linux都会默认支持，若出现改错误，则按下面步骤安装即可。


在CentOS版本下运行：

			yum install -y gcc gcc-c++
			
			
###### zlib library

Nginx HTTP gzip模块依赖该库，如系统没有则报错，按下面步骤安装即可。

在CentOS版本下运行：

			yum install -y zlib-devel
			


###安装nginx

安装nginx之前你需要考虑一个问题，那就是把你的nginx安装在哪里，如果你没有提供安装的路径的话，那么nginx会有个默认路径来存放各个文件，包括可运行文件，配置文件，日志文件等。为了避免麻烦，建议还是自己建立一个文件给它吧。

1. 新建nginx安装目录 （目录个人喜爱）
		
		mkdir /usr/nginx

2. 下载nginx ([nginx](www.nginx.com)的官网获得最新版本的nginx)
 		
 		wget http://nginx.org/download/nginx-1.6.2.tar.gz
 
3. 解压nginx

		tar -zvxf nginx-1.6.2.tar.gz
		
4. 配置(--prefix 选项提供nginx的安装路径)
		
		./configure --prefix=/usr/nginx
		
5.编译、安装

		make
		make install
		

###更新nginx

很多情况下，你需要更新nginx，比如说更新到最新版本的nginx，或者加入、删除nginx的某个模块等。

有2种方式来更新nginx:

1. 覆盖式
		
	这种方式会覆盖点之前的安装，如果是更新出现错误，那么就无法恢复到之前的安装。
		
		#这种更新方式其实就是重现安装一次nginx
		./configure --prefix=/usr/nginx
		make
		make install
	
2. 平缓更新
	
	这种方式保留之前的安装，在出现更新错误时候，开可以恢复到之前的安装。
		
		#备份久版本的nginx
		cp /usr/nginx/sbin/nginx /usr/nginx/sbin/nginx.bck
		
		./configure --prefix=/usr/nginx
		make
		
		#不要运行make install，否则和上面的就一样了
		
		#make编译好nginx之后，生成的可运行文件会放在objs/nginx中，替代原来版本的可运行文件即完成更新
		cp ./objs/nginx /usr/nginx/sbin/nginx




