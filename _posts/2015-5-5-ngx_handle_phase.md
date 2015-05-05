---
layout: post
title:  "nginx http的请求阶段"
date:   2015-5-5
categories: nginx
tag: nginx

---

当一个用户的http请求发送过来后，nginx的事件模块会监测到请求并建立连接，然后剩下的工作就交给http框架来处理了。那么从接收到http完整的请求头部到给用户返回响应的工程是怎么样的呢？第三方http模块又是如何介入http的处理流程的呢？

这篇文章假设你已经知道如何开发一个简单的http模块了。

在开发一个http模块前，你需要确认你的模块是对所有的http请求起作用呢，还是只是想对某个匹配到的uri起作用，这很重要，在后面的内容可以了解到，nginx针对这两种情况提供了不同的介入方法。

###nginx http请求的11个阶段
nginx允许多个http模块共同处理一个请求，这就必要出现各个模块之间的处理顺序问题，不同的处理顺序最后导致的结果可能是不一样的。为了解决这个问题，nginx将请求的处理分为11个阶段，每一个阶段都可以有任意数量的模块，模块之间通过流水线的方式来处理请求。

nginx定义的http请求阶段：

	typedef enum {
		//接收到完整的http头部之后的http阶段
    	NGX_HTTP_POST_READ_PHASE = 0,
		
		//在将请求URI与配置的location块匹配之前，修改请求URI的http阶段
    	NGX_HTTP_SERVER_REWRITE_PHASE,
		
		//匹配location阶段
    	NGX_HTTP_FIND_CONFIG_PHASE,
    	
    	//匹配到locaiotn之后，需要再次修改URI
    	NGX_HTTP_REWRITE_PHASE,
    	
    	//修改完URI之后，防止错误的配置导致循环修改URI
    	NGX_HTTP_POST_REWRITE_PHASE,
	
		//判断是否允许请求访问之前的处理阶段
    	NGX_HTTP_PREACCESS_PHASE,
		
		//判断是否允许请求访问
    	NGX_HTTP_ACCESS_PHASE,
    	
    	//决定是否允许访问之后的处理阶段
    	NGX_HTTP_POST_ACCESS_PHASE,
		
    	NGX_HTTP_TRY_FILES_PHASE,
    	
    	//处理请求内容阶段，大部分http模块介入的阶段
    	NGX_HTTP_CONTENT_PHASE,
		
		//处理完请求后记录日志的阶段
    	NGX_HTTP_LOG_PHASE
	} ngx_http_phases;

###http模块处理方法介入请求的例子
我们以一个nginx实现的http模块为例（`ngx_http_dav_module`），看看如何添加一个http模块。

	static ngx_http_module_t  ngx_http_dav_module_ctx = {
    	NULL,                                  /* preconfiguration */
    	ngx_http_dav_init,                     /* postconfiguration */

    	NULL,                                  /* create main configuration */
    	NULL,                                  /* init main configuration */

    	NULL,                                  /* create server configuration */
    	NULL,                                  /* merge server configuration */

    	ngx_http_dav_create_loc_conf,          /* create location configuration */
    	ngx_http_dav_merge_loc_conf            /* merge location configuration */
	};
	
	static ngx_int_t
	ngx_http_dav_init(ngx_conf_t *cf)
	{
    	ngx_http_handler_pt        *h;
    	ngx_http_core_main_conf_t  *cmcf;

    	cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
		
    	h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    	if (h == NULL) {
        	return NGX_ERROR;
    	}

    	*h = ngx_http_dav_handler;

    	return NGX_OK;
	}

可以从上面的模块代码看出，将自己的模块处理方法介入到nginx请求的过程是在module_ctx结构的`postconfiguration`方法中设置在某个阶段执行某个方法，在这个模块里面是在`NGX_HTTP_CONTENT_PHASE`阶段执行`ngx_http_dav_handler`方法。


###nginx实现阶段请求的数据结构

nginx是如何实现以多个阶段处理请求的呢？在上一篇文章中我们分析了nginx是如何解析配置的，其中有一个很重要的全局变量，指向http_core_module的main配置，所有阶段的方法都保存在main配置中：

	typedef struct {

    	ngx_http_phase_engine_t    phase_engine;
		 ...
    	ngx_http_phase_t           phases[NGX_HTTP_LOG_PHASE + 1];
    	
	} ngx_http_core_main_conf_t;


ngx_http_core_main_conf_t结构中有2个域是关于阶段请求的，其中phases是在解析nginx配置的过程中使用，而phase_engine则是在解析配置结束后利用phase的值来设置，phase_engine的值包含了一个http请求所有的处理方法。在http 请求的处理过程中只使用了phase_engine，而phase只是充当一个临时变量角色，用来设置在phase_engine。

其中涉及的数据结构如下：

	typedef struct ngx_http_phase_handler_s {
    	ngx_http_phase_handler_pt  checker;
    	ngx_http_handler_pt        handler;
    	ngx_uint_t                 next;
	};
	
	struct {
    	ngx_http_phase_handler_t  *handlers;
    	ngx_uint_t                 server_rewrite_index;
    	ngx_uint_t                 location_rewrite_index;
	} ngx_http_phase_engine_t;


	typedef struct {
    	ngx_array_t                handlers;
	} ngx_http_phase_t;


ngx_http_phase_handler_t结构代表了一个处理方法，它不仅包含了实际的处理方法，还有一个checker函数指针，它的作用是用来调度所有的处理方法，如跳转到下一阶段的处理方法中，第三方开发者一般不需去实现它或者修改它。nginx的http框架在处理请求时是直接调用checker方法，并由checker来决定如何执行handle。next用来实现跳转，指向下一阶段的第一个处理方法的位置。


在解析http块配置的时候，nginx会调用方法来设置phase_engine的值。

![](/assets/image/ngx_http_phase.png)

其中，`ngx_http_init_phases()`方法初始化`ngx_http_core_main_conf_t`的`phases`的值：

	static ngx_int_t
	ngx_http_init_phases(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
	{
    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_POST_READ_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_SERVER_REWRITE_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers,
                       cf->pool, 2, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers,
                       cf->pool, 4, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	if (ngx_array_init(&cmcf->phases[NGX_HTTP_LOG_PHASE].handlers,
                       cf->pool, 1, sizeof(ngx_http_handler_pt))
        	!= NGX_OK)
    	{
        	return NGX_ERROR;
    	}

    	return NGX_OK;
	}

接着调用所有http模块的`postconfiguration()`方法，第三方模块需要在postconfiguration()中设置main_conf的phases值。

在所有的http模块的postconfiguration()方法调用结束之后，意味着所有需要介入http请求的处理方法都已经保存在phases变量中了，这时候就调用ngx_http_init_phase_handlers()方法来设置phase_engine值。

	static ngx_int_t
	ngx_http_init_phase_handlers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
	{
    	ngx_int_t                   j;
    	ngx_uint_t                  i, n;
    	ngx_uint_t                  find_config_index, use_rewrite, use_access;
    	ngx_http_handler_pt        *h;
    	ngx_http_phase_handler_t   *ph;
    	ngx_http_phase_handler_pt   checker;

    	cmcf->phase_engine.server_rewrite_index = (ngx_uint_t) -1;
    	cmcf->phase_engine.location_rewrite_index = (ngx_uint_t) -1;
    	find_config_index = 0;
    	use_rewrite = cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers.nelts ? 1 : 0;
    	use_access = cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers.nelts ? 1 : 0;

    	n = use_rewrite + use_access + cmcf->try_files + 1 /* find config phase */;

    	for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
        	n += cmcf->phases[i].handlers.nelts;
    	}

    	ph = ngx_pcalloc(cf->pool,
                     n * sizeof(ngx_http_phase_handler_t) + sizeof(void *));
    	if (ph == NULL) {
        	return NGX_ERROR;
    	}

    	cmcf->phase_engine.handlers = ph;
    	n = 0;

    	for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
        	h = cmcf->phases[i].handlers.elts;

        	switch (i) {

        	case NGX_HTTP_SERVER_REWRITE_PHASE:
            	if (cmcf->phase_engine.server_rewrite_index == (ngx_uint_t) -1) {
                	cmcf->phase_engine.server_rewrite_index = n;
            	}
            	checker = ngx_http_core_rewrite_phase;

            	break;

        	case NGX_HTTP_FIND_CONFIG_PHASE:
            	find_config_index = n;

            	ph->checker = ngx_http_core_find_config_phase;
            	n++;
            	ph++;

            	continue;
	
        	case NGX_HTTP_REWRITE_PHASE:
            	if (cmcf->phase_engine.location_rewrite_index == (ngx_uint_t) -1) {
                	cmcf->phase_engine.location_rewrite_index = n;
            	}
            	checker = ngx_http_core_rewrite_phase;

            	break;

        	case NGX_HTTP_POST_REWRITE_PHASE:
            	if (use_rewrite) {
                	ph->checker = ngx_http_core_post_rewrite_phase;
                	ph->next = find_config_index;
                	n++;
                	ph++;
            	}

            	continue;

        	case NGX_HTTP_ACCESS_PHASE:
            	checker = ngx_http_core_access_phase;
            	n++;
            	break;

        	case NGX_HTTP_POST_ACCESS_PHASE:
            	if (use_access) {
                	ph->checker = ngx_http_core_post_access_phase;
                	ph->next = n;
                	ph++;
            	}

            	continue;

        	case NGX_HTTP_TRY_FILES_PHASE:
            	if (cmcf->try_files) {
                	ph->checker = ngx_http_core_try_files_phase;
                	n++;
                	ph++;
            	}

            	continue;

        	case NGX_HTTP_CONTENT_PHASE:
            	checker = ngx_http_core_content_phase;
            	break;

        	default:
            	checker = ngx_http_core_generic_phase;
        	}

        	n += cmcf->phases[i].handlers.nelts;

        	for (j = cmcf->phases[i].handlers.nelts - 1; j >=0; j--) {
            	ph->checker = checker;
            	ph->handler = h[j];
            	ph->next = n;
            	ph++;
        	}
    	}

    	return NGX_OK;
	}
	
	
如果你读懂了上面的这段代码，会发现，并不是添加到phases的所有handle都会起作用，比如在处理`NGX_HTTP_FIND_CONFIG_PHASE`阶段时，直接跳到了下一阶段的处理，意味在postconfiguration()方法中添加到`phases[NGX_HTTP_FIND_CONFIG_PHASE]`的handle方法，最终并不会添加到phase_engines中，也就无法介入http的请求。相类似的还有NGX_HTTP_POST_REWRITE_PHASE、NGX_HTTP_POST_ACCESS_PHASE、NGX_HTTP_TRY_FILES_PHASE这三个阶段。


Nginx的http框架是通过每个阶段的checker方法来调度handle的，不同阶段的checker有不同的实现，下面是对nginx提供的NGX_HTTP_POST_READ_PHASE阶段的checker方法ngx_http_core_generic_phase()的分析，其他的checker与此类似。

	ngx_int_t
	ngx_http_core_generic_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
	{
    	ngx_int_t  rc;

    	/*
     	* generic phase checker,
     	* used by the post read and pre-access phases
     	*/

    	ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "generic phase: %ui", r->phase_handler);

    	rc = ph->handler(r);

    	if (rc == NGX_OK) {
        	r->phase_handler = ph->next;
        	return NGX_AGAIN;
    	}

    	if (rc == NGX_DECLINED) {
        	r->phase_handler++;
        	return NGX_AGAIN;
    	}

    	if (rc == NGX_AGAIN || rc == NGX_DONE) {
        	return NGX_OK;
    	}

    	/* rc == NGX_ERROR || rc == NGX_HTTP_...  */

    	ngx_http_finalize_request(r, rc);

    	return NGX_OK;
	}
	
checker方法会根据hander方法的返回值来判断下一步的动作，下面给出各个返回值的意义：1)	NGX_OK：执行下一个阶段的第一个hander方法，忽略掉当前阶段后面没有执行的handler。2)	NGX_DECLINED：顺序执行下一个hander。3)	NGX_AGAIN：当前的hander方法还没有结束，可能需要再次调用。通常会把处理控制权交个事件模块，在下一次可写事件触发时会再次执行该hander。4)	NGX_DONE：与NGX_AGAIN返回值意义相同。5)	NGX_ERROR：出错，结束请求。

