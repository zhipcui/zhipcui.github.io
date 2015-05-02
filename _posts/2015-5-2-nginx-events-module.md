---
layout: post
title:  "nginx事件模型"
date:   2015-5-2
categories: nginx
tag: nginx

---

nginx采用了非阻塞的事件模型，可以同时监听多个事件。nginx目前实现了kqueue、epoll、select、poll、eventport、aio、devpoll、rtsig和基于windows的select共9个事件模型。具体使用哪一个可以在配置文件中设置，nginx关于事件模型的设置是在`events{}`块中设置，如:

	events{
		use kqueue;
	}nginx的事件模块中有2个关键模块:`ngx_events_module`、`ngx_event_core_module`。

####ngx_events_module

`ngx_events_module`是事件模块的基础，定义了`events{}`配置块，在遇到events配置时会启动事件模型的各个模块回调函数。	
ngx_events_module的定义：	
	
	static ngx_command_t  ngx_events_commands[] = {

    { ngx_string("events"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_events_block,     //遇到events配置块时会调用，
      0,
      0,
      NULL },

      ngx_null_command
	};


	static ngx_core_module_t  ngx_events_module_ctx = {
    	ngx_string("events"),
    	NULL,
    	ngx_event_init_conf	  //检查是否有events配置块，若如返回NGX_CONF_ERROR
	};


	ngx_module_t  ngx_events_module = {
    	NGX_MODULE_V1,
    	&ngx_events_module_ctx,                /* module context */
    	ngx_events_commands,                   /* module directives */
    	NGX_CORE_MODULE,                       /* module type */
    	NULL,                                  /* init master */
    	NULL,                                  /* init module */
    	NULL,                                  /* init process */
    	NULL,                                  /* init thread */
    	NULL,                                  /* exit thread */
    	NULL,                                  /* exit process */
    	NULL,                                  /* exit master */
    	NGX_MODULE_V1_PADDING
	};
	
ngx_events_block函数完成一下内容：

1. 检查是否有多个events配置块
2. 给所有的事件模块编号，即设置ctx_inde值
3. 调用所有事件模块的create_conf方法
4. 获取所有事件模块配置后，调用所有事件模块的init_conf方法


####ngx_events_core_module

`ngx_events_core_module`是一个事件模块，但是与其他具体的事件模型不同的是，它并没有提供特定的事件模型实现，而是用来管理nginx事件模型，如决定采用哪个事件模型，以及一些基本事件模型的设置。

ngx_events_core_module的定义：

	static ngx_str_t  event_core_name = ngx_string("event_core");


	static ngx_command_t  ngx_event_core_commands[] = {
		
		//连接池的大小
    	{ ngx_string("worker_connections"),
      	  NGX_EVENT_CONF|NGX_CONF_TAKE1,
      	  ngx_event_connections,
      	  0,
      	  0,
      	  NULL },
		
		//等同于worker_connections
    	{ ngx_string("connections"),
          NGX_EVENT_CONF|NGX_CONF_TAKE1,
      	  ngx_event_connections,
      	  0,
      	  0,
      	  NULL },
      	  
		//使用哪一个事件模型
    	{ ngx_string("use"),
      	  NGX_EVENT_CONF|NGX_CONF_TAKE1,
      	  ngx_event_use,
      	  0,
      	  0,
      	  NULL },

    	{ ngx_string("multi_accept"),
      	  NGX_EVENT_CONF|NGX_CONF_FLAG,
      	  ngx_conf_set_flag_slot,
      	  0,
      	  offsetof(ngx_event_conf_t, multi_accept),
      	  NULL },

    	{ ngx_string("accept_mutex"),
      	  NGX_EVENT_CONF|NGX_CONF_FLAG,
      	  ngx_conf_set_flag_slot,
      	  0,
      	  offsetof(ngx_event_conf_t, accept_mutex),
      	  NULL },

    	{ ngx_string("accept_mutex_delay"),
      	  NGX_EVENT_CONF|NGX_CONF_TAKE1,
      	  ngx_conf_set_msec_slot,
      	  0,
      	  offsetof(ngx_event_conf_t, accept_mutex_delay),
      	  NULL },

    	{ ngx_string("debug_connection"),
      	  NGX_EVENT_CONF|NGX_CONF_TAKE1,
      	  ngx_event_debug_connection,
      	  0,
      	  0,
      	  NULL },

      	  ngx_null_command
	};


	ngx_event_module_t  ngx_event_core_module_ctx = {
   		&event_core_name,
    	ngx_event_core_create_conf,            /* create configuration */
    	ngx_event_core_init_conf,              /* init configuration */

    	{ NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL }
	};


	ngx_module_t  ngx_event_core_module = {
    	NGX_MODULE_V1,
    	&ngx_event_core_module_ctx,            /* module context */
    	ngx_event_core_commands,               /* module directives */
    	NGX_EVENT_MODULE,                      /* module type */
    	NULL,                                  /* init master */
    	ngx_event_module_init,                 /* init module */
    	ngx_event_process_init,                /* init process */
    	NULL,                                  /* init thread */
    	NULL,                                  /* exit thread */
    	NULL,                                  /* exit process */
    	NULL,                                  /* exit master */
    	NGX_MODULE_V1_PADDING
	};






###如何定义一个事件模块

定义事件模块的通用接口：

	typedef struct {
    	ngx_str_t    *name;     //事件模型的名字如，kqueue、select等

    	void    *(*create_conf)(ngx_cycle_t *cycle);   //解析配置之前调用，用于创建存储配置的结构体
    	char    *(*init_conf)(ngx_cycle_t *cycle, void *conf);   //完成解析配置后调用，对获得的配置做进一步的处理

    	ngx_event_actions_t     actions;   //每个事件模块需要实现的事件处理接口
	} ngx_event_module_t;
	


事件模型的关键接口：

	typedef struct {
		//添加一个事件到监听队列
    	ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    	//删除一个事件
    	ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
		
		//启用一个事件
    	ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    	//禁用一个事件
    	ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

		//添加一个连接，连接上的读写事件会同时添加到监听队列
    	ngx_int_t  (*add_conn)(ngx_connection_t *c);
    	//删除一个连接，连接上的读写事件会从监听队列中删除
    	ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

		
    	ngx_int_t  (*process_changes)(ngx_cycle_t *cycle, ngx_uint_t nowait);
    	//处理、分发事件的核心
    	ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                   ngx_uint_t flags);
		
		//初始化事件模型
    	ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    	//退出事件模型时调用
    	void       (*done)(ngx_cycle_t *cycle);
	} ngx_event_actions_t;


###kqueue事件模型

	static ngx_str_t      kqueue_name = ngx_string("kqueue");

	static ngx_command_t  ngx_kqueue_commands[] = {

    	{ ngx_string("kqueue_changes"),
     	  NGX_EVENT_CONF|NGX_CONF_TAKE1,
     	  ngx_conf_set_num_slot,
     	  0,
     	  offsetof(ngx_kqueue_conf_t, changes),
     	  NULL },

    	{ ngx_string("kqueue_events"),
      	  NGX_EVENT_CONF|NGX_CONF_TAKE1,
      	  ngx_conf_set_num_slot,
      	  0,
      	  offsetof(ngx_kqueue_conf_t, events),
      	  NULL },

      	 ngx_null_command
	};


	ngx_event_module_t  ngx_kqueue_module_ctx = {
    	&kqueue_name,
    	ngx_kqueue_create_conf,                /* create configuration */
    	ngx_kqueue_init_conf,                  /* init configuration */

    	{
        	ngx_kqueue_add_event,              /* add an event */
        	ngx_kqueue_del_event,              /* delete an event */
        	ngx_kqueue_add_event,              /* enable an event */
        	ngx_kqueue_del_event,              /* disable an event */
        	NULL,                              /* add an connection */
        	NULL,                              /* delete an connection */
        	ngx_kqueue_process_changes,        /* process the changes */
        	ngx_kqueue_process_events,         /* process the events */
        	ngx_kqueue_init,                   /* init the events */
        	ngx_kqueue_done                    /* done the events */
    	}

	};

	ngx_module_t  ngx_kqueue_module = {
    	NGX_MODULE_V1,
    	&ngx_kqueue_module_ctx,                /* module context */
    	ngx_kqueue_commands,                   /* module directives */
    	NGX_EVENT_MODULE,                      /* module type */
    	NULL,                                  /* init master */
    	NULL,                                  /* init module */
    	NULL,                                  /* init process */
    	NULL,                                  /* init thread */
    	NULL,                                  /* exit thread */
    	NULL,                                  /* exit process */
    	NULL,                                  /* exit master */
    	NGX_MODULE_V1_PADDING
	};
	
	
###事件
事件模型是对事件的处理，nginx对事件的定义如下：


	struct ngx_event_s {
		
		//该事件相关的对象，通常是ngx_connection_t类型对象
    	void            *data;  
		
		//标志该事件是可写的
    	unsigned         write:1;
		
		//标志该事件可以新建连接
    	unsigned         accept:1;

    	//标志事件是否过期
    	unsigned         instance:1;

    	//标志事件是否活跃
    	unsigned         active:1;

		//标志事件是否被禁用
    	unsigned         disabled:1;

    	//标志事件是否准备就绪
    	unsigned         ready:1;

    	unsigned         oneshot:1;

    	/* aio operation is complete */
    	unsigned         complete:1;
		
		//标志当前处理的字符流是否结束
    	unsigned         eof:1;
    	
    	//标志事件处理过程是否出错
    	unsigned         error:1;
		
		//标志事件是否超时
    	unsigned         timedout:1;
    	
    	//标志事件是否存在与定时器中
    	unsigned         timer_set:1;

		//标志事件是否需要延迟处理
    	unsigned         delayed:1;
		
		//标志是否延迟建立tcp连接，在tcp三次握手之后不马上建立连接，而是等到客户端真正有数据包过来之后才建立tcp连接
    	unsigned         deferred_accept:1;

    	/* the pending eof reported by kqueue, epoll or in aio chain operation */
    	unsigned         pending_eof:1;

		#if !(NGX_THREADS)
    		unsigned         posted_ready:1;
		#endif

		#if (NGX_WIN32)
    		/* setsockopt(SO_UPDATE_ACCEPT_CONTEXT) was successful */
    		unsigned         accept_context_updated:1;
		#endif

		#if (NGX_HAVE_KQUEUE)
    		unsigned         kq_vnode:1;

    		/* the pending errno reported by kqueue */
    		int              kq_errno;
		#endif

    	/*
    	 * kqueue only:
    	 *   accept:     number of sockets that wait to be accepted
    	 *   read:       bytes to read when event is ready
    	 *               or lowat when event is set with NGX_LOWAT_EVENT flag
    	 *   write:      available space in buffer when event is ready
   	  	 *               or lowat when event is set with NGX_LOWAT_EVENT flag
    	 *
    	 * iocp: TODO
    	 *
    	 * otherwise:
    	 *   accept:     1 if accept many, 0 otherwise
    	 */

		#if (NGX_HAVE_KQUEUE) || (NGX_HAVE_IOCP)
    		int              available;
		#else
    		unsigned         available:1;
		#endif
	
		//事件发生时的处理方法
    	ngx_event_handler_pt  handler;


		#if (NGX_HAVE_AIO)

		#if (NGX_HAVE_IOCP)
    		ngx_event_ovlp_t ovlp;
		#else
    		struct aiocb     aiocb;
		#endif

		#endif

    	ngx_uint_t       index;

    	ngx_log_t       *log;

    	ngx_rbtree_node_t   timer;

    	unsigned         closed:1;

    	/* to test on worker exit */
    	unsigned         channel:1;
    	unsigned         resolver:1;

		#if (NGX_THREADS)

    		unsigned         locked:1;

    		unsigned         posted_ready:1;
    		unsigned         posted_timedout:1;
    		unsigned         posted_eof:1;

		#if (NGX_HAVE_KQUEUE)
    		/* the pending errno reported by kqueue */
    		int              posted_errno;
		#endif

		#if (NGX_HAVE_KQUEUE) || (NGX_HAVE_IOCP)
    		int              posted_available;
		#else
    		unsigned         posted_available:1;
		#endif

    	ngx_atomic_t    *lock;
    	ngx_atomic_t    *own_lock;

		#endif

    	/* the links of the posted queue */
    	ngx_event_t     *next;
    	ngx_event_t    **prev;


		#if 0

    	/* the threads support */

    	/*
    	 * the event thread context, we store it here
    	 * if $(CC) does not understand __thread declaration
    	 * and pthread_getspecific() is too costly
    	 */

    	void            *thr_ctx;

		#if (NGX_EVENT_T_PADDING)

    	/* event should not cross cache line in SMP */

    	uint32_t         padding[NGX_EVENT_T_PADDING];
		#endif
		#endif
	};
