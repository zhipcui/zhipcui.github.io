---
layout: post
title:  "nginx http请求的执行过程"
date:   2015-5-6
categories: nginx
tag: nginx

---


上一篇博文中介绍了http请求的11个过程，以及第三方模块应该如何使得自定义的方法介入http请求中。这篇文章将分析nginx是怎么以及何时调用这11个过程的各个handle的。下图展示了nginx从接收到用户请求到执行handle的整个过程。

![](/assets/image/ngx_http_request.png)

nginx的事件模块一直在后台监听多个端口，端口是在nginx.conf配置文件中配置的所有端口。nginx定义了ngx_listening_t结构来表示一个监听，该结构有个handle方法，在每次监听到用户请求的时候就会调用handle方法。handle方法是在解析http配置的时候就已经设置为`ngx_http_init_connection()`。

nginx定义了ngx_connection_t结构来表示一个连接，每一个连接都有一个可写事件和可读事件。一个用户请求对应了一个connection，ngx_http_init_connection()的工作就是初始化connection，设置连接的可写事件和可读事件。其中ngx_http_init_connection()会设置connection的可读事件为`ngx_http_wait_request_handler()`，并添加到事件模块的事件队列中，它表示用户在当前连接上发送数据到服务器上时，应该调用ngx_http_wait_request_handler()，而接收的数据就是http的请求行。ngx_http_init_connection()除了把可读可写事件添加到事件模块的事件队列中外,还会把可读事件添加的定时器中，实现用户请求超时的处理。

当用户的第一次请求数据到达服务器时，即请求行，connnection的可读事件触发，调用ngx_http_wait_request_handler()方法。这个函数就开始接收数据，并开始解析请求行和请求头的数据，在完整的接收到请求的数据之后，就有能力开始对该请求做处理了，这时会创建一个ngx_http_request_t类型的数据来表示这个用户请求，之后所有的处理都依赖该数据，如请求的头部信息，请求的URL，包括之后返回的响应数据都是存储在该结构中。这里有个巧妙的设计，nginx并没有在成功建立连接之后就创建request，而是在完整地接收到请求头部时才创建，这样可以避免一些用户建立连接之后一值都没有发送数据过来而导致的内存资源的浪费。

在解析完请求头之后，就可以开始处理这个请求了，ngx_http_core_run_phases()方法会依次调用所有阶段的checker方法，也就是说每个http模块开始可以处理这个请求了。

ngx_http_core_run_phases()方法的实现：

	void
	ngx_http_core_run_phases(ngx_http_request_t *r)
	{
    	ngx_int_t                   rc;
    	ngx_http_phase_handler_t   *ph;
    	ngx_http_core_main_conf_t  *cmcf;

    	cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    	ph = cmcf->phase_engine.handlers;

    	while (ph[r->phase_handler].checker) {

        	rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        	if (rc == NGX_OK) {
            	return;
        	}
    	}
	}
	
	
然而并不是所有的连接都能顺利的进行，有可能会出现状况导致请求中断，或者请求本事就不合法，这时候需要结束掉请求和断开连接，nginx提供了几个方法来做这类结束工作，并保持服务器的整洁性，整洁性表现在对系统资源的回收上。

在连接建立后，请求初始化之前，在这个时候发送错误需要调用ngx_http_close_connection()断开连接，然后销毁对应的内存池，关于ngx_http_close_connection()的定义如下：

	void
	ngx_http_close_connection(ngx_connection_t *c)
	{
    	ngx_pool_t  *pool;

    	ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   	"close http connection: %d", c->fd);

	#if (NGX_HTTP_SSL)

    	if (c->ssl) {
        	if (ngx_ssl_shutdown(c) == NGX_AGAIN) {
            	c->ssl->handler = ngx_http_close_connection;
            	return;
        	}
    	}

	#endif

	#if (NGX_STAT_STUB)
    	(void) ngx_atomic_fetch_add(ngx_stat_active, -1);
	#endif

    	c->destroyed = 1;

    	pool = c->pool;

    	ngx_close_connection(c);

    	ngx_destroy_pool(pool);
	}
	
在ngx_http_close_connection()方法中调用ngx_close_connection()方法来把连接响应的读写事件从事件模块中清除。

在初始化请求之后发送错误，需要ngx_http_finalize_request()来结束请求。
	