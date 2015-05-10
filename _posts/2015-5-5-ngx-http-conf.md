---
layout: post
title:  "nginx http配置解析"
date:   2015-5-5
categories: nginx
tag: nginx

---


nginx的配置结构是由各种配置块组成，配置块内又可以嵌套多层配置块。然而nginx是如何来解析这种层层嵌套的结构的呢，第三方模块又如何定义自己的模块的配置项的呢？下面从nginx的源码来分析。

先来看一下一个简单的http配置例子：

	http {
		some 1;
		server {
			server_name a;
			listen 80;
			some 2;
			location /a{
				some 3;
			}
		}
		
		...
		
		server {
			server_name b;
			listen 8000;
			some 4;
			location /a{
				some 5;
			}
			location /b{
			}
		}
	}
	
相同的指令可能出现在http块，也可能出现server块、location块，那就有一个问题，假设现在访问server a的locaion a，some指令取什么值？一般来说，会以最近设置的值为准，但是nginx允许各个模块定制这个行为。but how?


http框架是nginx内容最丰富的部分，有非常多的http模块，但是具体实现http框架的模块主要有2个：`ngx_http`、`ngx_http_core_module`。

`http{}`配置项在`ngx_http`模块处理，`server{}`和`location{}`块在`ngx_http_core_module`模块处理。

###代码处理过程


![](/assets/image/ngx_http_conf_code.png)


###数据结构分析

![](/assets/image/ngx_http_conf_data.png)


下面看一下设计的数据结构的定义，只列出与配置相关的域。

ngx_http_conf_ctx_t结构体的定义：

	typedef struct {
    	void        **main_conf;
    	void        **srv_conf;
    	void        **loc_conf;
	} ngx_http_conf_ctx_t;

ngx_http_core_main_conf_t结构体的定义：
	
	typedef struct {
    	ngx_array_t                servers;         /* ngx_http_core_srv_conf_t */
		...
	} ngx_http_core_main_conf_t;

ngx_http_core_srv_conf_t结构体的定义：

	typedef struct {

    	/* server ctx */
    	ngx_http_conf_ctx_t        *ctx;

    	ngx_str_t                   server_name;
    
	} ngx_http_core_srv_conf_t;
	
ngx_http_core_loc_conf_s结构体的定义：

	struct ngx_http_core_loc_conf_s {
    	ngx_str_t     name;          /* location name */

    	/* pointer to the modules' loc_conf */
    	void        **loc_conf;

    	ngx_http_handler_pt  handler;

    	ngx_queue_t  *locations;

	};


