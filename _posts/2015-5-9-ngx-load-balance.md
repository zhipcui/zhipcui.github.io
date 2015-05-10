---
layout: post
title:  "nginx 负载均衡"
date:   2015-5-9
categories: nginx
tag: nginx

---
负载均衡是指按照一定的策略，在多个服务器的集群中选择一台合适的机器来服务请求，已达到性能最佳的机制。性能最佳是以负载均衡的算法来衡量的，比较常用的负载均衡算法有如下几种：
 
 1. 轮询：按顺序把每一个新的连接分配给服务器列表中的下一个服务器。
 2. 加权轮询：给每个服务器一个权值，并将连接按权值比例来分配。
 3. ip_hash：根据连接的IP地址的hash值来分配服务器，来自同一IP的请求总是由同一个服务器服务。
 4. 最少连接：新的连接将分配到当前连接数最少的服务器。
 5. 最快响应：新的连接将分配到具有最快响应时间的服务器。
 
###nginx负载均衡如何实现的
 
nginx作为一款高性能的反向代理服务器 ，实现了加权轮询、ip_hash和最少连接的负载均衡算法，而且nginx允许使用第三方开发的负载均衡算法。在上一篇文章中分析upstream机制时，没有提到如何设置上游服务器的连接地址，因为这一部分和负载均衡服务有关。nginx的负载均衡是对访问上游服务器的地址选择策略，是在upstream模块中实现的。upstream结构中有2个域与负载均衡密切相关:peer与resolved。

	struct ngx_http_upstream_s {
		...
		ngx_peer_connection_t            peer;
    	ngx_http_upstream_resolved_t    *resolved;
    	...
    }
 
resolved表示上游服务器的地址，而peer表示与上游服务器的连接。这2个域的值决定了upstream上游服务器的地址是如何确定的。上游服务器的地址有两种设置方式，一种是直接设置为固定地址，如一个IP地址或者域名。另一种是通过nginx的`upstream{}`的配置块组建一个集群，然后提供算法来从中选取一个。要使用负载均衡则必须使用第二种方式。下面是一个upstream配置例子：
 
	http{
		upstream cluster1 {
			server google.com;
			server bing.com;
			server baidu.com;
			...
		}
		
		upstream cluster2 {
			server facebook.com;
			server twitter.com;
			server weibo.com;
			...
		}
	}
 
nginx允许配置任意数量的upstream配置块，每一个配置块可以有独立的负载均衡实现，这是因为nginx为每一个upstream配置块分配了一个ngx_http_upstream_srv_conf_t结构用于存储一个配置块的信息。在解析完http配置块之后，就会调用每一个的配置块中upstream_srv_conf->peer->init_upstream()方法来初始化负载均衡模块，如果没有设置特定的负载均衡模块，则默认为轮询模块(ngx_http_upstream_round_robin)负责。在init_upstream()方法中需要设置upstream_srv_conf->peer->init()，

	//每一个upstream的配置
	struct ngx_http_upstream_srv_conf_s {
    	ngx_http_upstream_peer_t         peer;   //每一个upstream都有只有一个，用来决定选择server
    	void                           **srv_conf;

    	ngx_array_t                     *servers;  /* ngx_http_upstream_server_t */

    	ngx_uint_t                       flags;
    	ngx_str_t                        host;
    	u_char                          *file_name;
    	ngx_uint_t                       line;
    	in_port_t                        port;
    	in_port_t                        default_port;
    	ngx_uint_t                       no_port;  /* unsigned no_port:1 */
	}
	
	typedef struct {
    	ngx_http_upstream_init_pt        init_upstream;
    	ngx_http_upstream_init_peer_pt   init;
    	void                            *data;
	} ngx_http_upstream_peer_t;

![](/assets/image/ngx-load-balance.png)
 
在upstream启动之后，会调用ngx_http_upstream_init_request()方法来初始化连接，其中一个重要的工作就设置好上游服务器的地址，从上图也可以看出，负载均衡工作最后的结果就是设置好connect()函数的sockaddr参数。如果upstream->resolved->sockaddr不为空，则说明采用了第一种方式来设置上游服务器地址，即直接给定了ip地址或者域名，否则就会调用peer的get方法()来获取指定的集群中的某台服务器，而负载均衡算法就是在get()方法中实现的，所以要开发一个特大算法的负载均衡模块的话，最重要的工作就是实现peer的get()方法。peer是ngx_peer_connection_t结构的变量，代表了一个主动连接(nginx发起的连接)。

	struct ngx_peer_connection_s {
    	ngx_connection_t                *connection;

    	struct sockaddr                 *sockaddr;  //上游服务器的地址
    	socklen_t                        socklen;
    	ngx_str_t                       *name;

    	ngx_uint_t                       tries;

    	ngx_event_get_peer_pt            get;   //获取服务器，负载聚会算法实现的地方
    	ngx_event_free_peer_pt           free;  //结束连接时调用，负载结束工作
    	void                            *data;

	#if (NGX_SSL)
    	ngx_event_set_peer_session_pt    set_session;
    	ngx_event_save_peer_session_pt   save_session;
	#endif

	#if (NGX_THREADS)
    	ngx_atomic_t                    *lock;
	#endif

    	ngx_addr_t                      *local;

    	int                              rcvbuf;

    	ngx_log_t                       *log;

    	unsigned                         cached:1;

                                     /* ngx_connection_log_error_e */
    	unsigned                         log_error:2;
	};

 
###nginx轮询负载均衡模块

nginx提供了轮询负载均衡模块（支持加权），并作为默认的负载均衡配置。由于轮询算法是最基本的负载均衡算法，所以在实现其他负载均衡算法实现时，轮询算法可以作为备用方案，在其他负载算法出现异常时，依旧可以使用轮询算法来分配服务器，提高了系统的稳定性。

在配置upstream时，server配置项指定了服务器地址，地址可以是IP地址、域名或者UNIX句柄等，除此之外，还可以设置其他的参数：

1. weight=number：指定这台服务器的权值，默认值为1。
2. max_fails=number：与下面要说的fail_timeout配合使用，表示在fail_timeout时间内允许的最大转发失败次数，如果超过该数字则标志这台服务器不可用，默认值为1.
3. fail_timeout=time：见上，默认值为10。
4. backup：标志这台服务器为备用机器，只有其他可用服务器都不可用的时候才使用。
5. down：标志这台服务器处于下线状态。

在开发负载均衡模块时，需要说明模块支持的server配置参数。设置方法如下：

	ngx_http_upstream_srv_conf_t  *uscf;
	uscf->flags = NGX_HTTP_UPSTREAM_CREATE
                  	|NGX_HTTP_UPSTREAM_WEIGHT
                  	|NGX_HTTP_UPSTREAM_MAX_FAILS
                  	|NGX_HTTP_UPSTREAM_FAIL_TIMEOUT
                  	|NGX_HTTP_UPSTREAM_DOWN;

upstream模块定义了多个宏来标志server支持的参数：

	#define NGX_HTTP_UPSTREAM_CREATE        0x0001
	#define NGX_HTTP_UPSTREAM_WEIGHT        0x0002
	#define NGX_HTTP_UPSTREAM_MAX_FAILS     0x0004
	#define NGX_HTTP_UPSTREAM_FAIL_TIMEOUT  0x0008
	#define NGX_HTTP_UPSTREAM_DOWN          0x0010
	#define NGX_HTTP_UPSTREAM_BACKUP        0x0020
	
nginx实现的轮询负载均衡模块提供以上所有的server参数。


######nginx轮询负载均衡模块重要的数据结构定义
ngx_http_upstream_rr_peers_s结构保存了一个集群里面所有服务器的信息，包括每一个服务器的权值，备用服务器等。

	struct ngx_http_upstream_rr_peers_s {
    	ngx_uint_t                      number;         //peer的个数

    	ngx_uint_t                      total_weight;   //全部peer的权重之和

    	unsigned                        single:1;       //是否只有一个可用peer
    	unsigned                        weighted:1;     //是否设置了权重

    	ngx_str_t                      *name;           //所属upstream的host

    	ngx_http_upstream_rr_peers_t   *next;           //指向备用servers

    	ngx_http_upstream_rr_peer_t     peer[1];        //保存所有peer
	};

ngx_http_upstream_rr_peer_t结构表示一台服务器的信息，如服务器的权值，地址，已转发失败次数等。

	typedef struct {
   		struct sockaddr                *sockaddr;
    	socklen_t                       socklen;
    	ngx_str_t                       name;

    	ngx_int_t                       current_weight;     //当前权重
    	ngx_int_t                       effective_weight;
    	ngx_int_t                       weight;        //设置的权重

    	ngx_uint_t                      fails;         //已失败次数
    	time_t                          accessed;
    	time_t                          checked;       //已连接多长时间

    	ngx_uint_t                      max_fails;     //最大失败次数
    	time_t                          fail_timeout;  //超时时间

    	ngx_uint_t                      down;          /* unsigned  down:1; */

	#if (NGX_HTTP_SSL)
    	ngx_ssl_session_t              *ssl_session;   /* local to a process */
	#endif
	} ngx_http_upstream_rr_peer_t;
	
######peer->init_upstream方法

上面提到，在解析完http配置之后，会调用每个upstream配置块的init_upstream方法来初始化负载均衡算法。而默认的轮询模块的init_upstream方法为ngx_http_upstream_init_round_robin()。该方法的主要工作就创建ngx_http_upstream_rr_peers_t结构，并根据配置来设置好，
最后使得us->peer.data指向该结构。

	ngx_int_t
	ngx_http_upstream_init_round_robin(ngx_conf_t *cf,
    	ngx_http_upstream_srv_conf_t *us)
	{
		...
		
    	ngx_http_upstream_rr_peers_t  *peers, *backup;
    	...
        peers = ngx_pcalloc(cf->pool, sizeof(ngx_http_upstream_rr_peers_t)
                              + sizeof(ngx_http_upstream_rr_peer_t) * (n - 1));
        if (peers == NULL) {
            return NGX_ERROR;
        }

     ...
     //设置每一个peer
        for (i = 0; i < us->servers->nelts; i++) {
            if (server[i].backup) {
                continue;
            }

            for (j = 0; j < server[i].naddrs; j++) {
                peers->peer[n].sockaddr = server[i].addrs[j].sockaddr;
                peers->peer[n].socklen = server[i].addrs[j].socklen;
                peers->peer[n].name = server[i].addrs[j].name;
                peers->peer[n].weight = server[i].weight;
                peers->peer[n].effective_weight = server[i].weight;
                peers->peer[n].current_weight = 0;
                peers->peer[n].max_fails = server[i].max_fails;
                peers->peer[n].fail_timeout = server[i].fail_timeout;
                peers->peer[n].down = server[i].down;
                n++;
            }
        }
        ...
        us->peer.data = peers;
        ...
	}


######peer->init方法

在转发一个新的请求时，调用init方法来做一些初始化的工作，比如设置新连接的连接时间为0等。默认的轮询模块的init方法为ngx_http_upstream_init_round_robin_peer()。该函数的关键是把在init_upstream()方法中创建的ngx_http_upstream_rr_peers_t结构存放到request->upstream->peer.data上去。


	ngx_int_t
	ngx_http_upstream_init_round_robin_peer(ngx_http_request_t *r,
    	ngx_http_upstream_srv_conf_t *us)
	{
		....
		
		ngx_http_upstream_rr_peer_data_t  *rrp;
		rrp = r->upstream->peer.data;
		...
		
		rrp->peers = us->peer.data;
    	rrp->current = 0;
	
		...
		
		r->upstream->peer.get = ngx_http_upstream_get_round_robin_peer;
    	r->upstream->peer.free = ngx_http_upstream_free_round_robin_peer;
    	...
    }
		

######peer.get方法

upstream是通过调用peer->get方法来获取集群中具体哪一个服务器，默认的轮询模块的init方法为ngx_http_upstream_get_round_robin_peer()。

	ngx_int_t
	ngx_http_upstream_get_round_robin_peer(ngx_peer_connection_t *pc, void *data)
	{
		...
		peer = ngx_http_upstream_get_peer(rrp);
		...
	}
	
	static ngx_http_upstream_rr_peer_t *
	ngx_http_upstream_get_peer(ngx_http_upstream_rr_peer_data_t *rrp)
	{
    	time_t                        now;
    	uintptr_t                     m;
    	ngx_int_t                     total;
    	ngx_uint_t                    i, n;
    	ngx_http_upstream_rr_peer_t  *peer, *best;

    	now = ngx_time();

    	best = NULL;
    	total = 0;

    	for (i = 0; i < rrp->peers->number; i++) {

        	n = i / (8 * sizeof(uintptr_t));
        	m = (uintptr_t) 1 << i % (8 * sizeof(uintptr_t));

        	if (rrp->tried[n] & m) {
            	continue;
        	}

        	peer = &rrp->peers->peer[i];

        	if (peer->down) {
            	continue;
        	}

        	if (peer->max_fails
            	&& peer->fails >= peer->max_fails
            	&& now - peer->checked <= peer->fail_timeout)
        	{
            	continue;
        	}

       		peer->current_weight += peer->effective_weight;
        	total += peer->effective_weight;

        	if (peer->effective_weight < peer->weight) {
            	peer->effective_weight++;
        	}

        	if (best == NULL || peer->current_weight > best->current_weight) {
            	best = peer;
        	}
    	}

    	if (best == NULL) {
        	return NULL;
    	}

    	i = best - &rrp->peers->peer[0];

    	rrp->current = i;

    	n = i / (8 * sizeof(uintptr_t));
    	m = (uintptr_t) 1 << i % (8 * sizeof(uintptr_t));

    	rrp->tried[n] |= m;

    	best->current_weight -= total;

    	if (now - best->checked > best->fail_timeout) {
        	best->checked = now;
    	}

    	return best;
	}



