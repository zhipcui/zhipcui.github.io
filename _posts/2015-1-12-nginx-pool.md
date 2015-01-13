---
layout: post
title:  "nginx内存池"
date:   2015-1-12
categories: nginx
tag: nginx

---

nginx定义了自己的内存池机制，nginx的高性能与其息息相关。

**定义所在路径**:

 * `/src/core/ngx_palloc.c (.h)`
 * `/src/os/unix/ngx_alloc.c (.h)`
 
**中心思想**：

nginx的内存池由2部分组成，一个是头部信息，和数据部。数据部包含了要分配的大块连续内存，而头部信息存储了关于当前分配内存的信息。

**图解**
![ngx_pool_t](/assets/image/nginx-pool-t.jpeg)

**定义的结构**：
 
 	ngx_pool_t
 	ngx_pool_data_t
 	ngx_pool_large_t
	ngx_pool_cleanup_t
	ngx_pool_cleanup_file_t


**定义**：

	struct ngx_pool_s {
    	ngx_pool_data_t       d;
    	size_t                max;
    	ngx_pool_t           *current;
    	ngx_chain_t          *chain;
    	ngx_pool_large_t     *large;
    	ngx_pool_cleanup_t   *cleanup;
    	ngx_log_t            *log;
	};
	
	typedef struct {
    	u_char               *last;
    	u_char               *end;
    	ngx_pool_t           *next;
    	ngx_uint_t            failed;
	} ngx_pool_data_t;

	struct ngx_pool_large_s {
    	ngx_pool_large_t     *next;
    	void                 *alloc;
	};
	
	struct ngx_pool_cleanup_s {
    	ngx_pool_cleanup_pt   handler;
    	void                 *data;
    	ngx_pool_cleanup_t   *next;
	};
	
	typedef struct {
    	ngx_fd_t              fd;
    	u_char               *name;
    	ngx_log_t            *log;
	} ngx_pool_cleanup_file_t;



**定义的函数**

	ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
	void ngx_destroy_pool(ngx_pool_t *pool);
	void ngx_reset_pool(ngx_pool_t *pool);

	void *ngx_palloc(ngx_pool_t *pool, size_t size);
	void *ngx_pnalloc(ngx_pool_t *pool, size_t size);
	void *ngx_pcalloc(ngx_pool_t *pool, size_t size);
	void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment);
	ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);


	ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size);
	void ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd);
	void ngx_pool_cleanup_file(void *data);
	void ngx_pool_delete_file(void *data);



**理解**

1.  ngx_create_pool（创建内存池）
 
		ngx_pool_t * ngx_create_pool(size_t size, ngx_log_t *log){
		
			ngx_pool_t  *p;
	
			/*
			    关于内存对齐：
 				* Linux has memalign() or posix_memalign()
 				* Solaris has memalign()
 				* FreeBSD 7.0 has posix_memalign(), besides, early version's malloc()
 				* aligns allocations bigger than page size at the page boundary
 			*/
 			
    		p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    		if (p == NULL) {
        		return NULL;
    		}

    		p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    		p->d.end = (u_char *) p + size;
    		p->d.next = NULL;
    		p->d.failed = 0;

    		size = size - sizeof(ngx_pool_t);
    		p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;

    		p->current = p;
    		p->chain = NULL;
    		p->large = NULL;
    		p->cleanup = NULL;
    		p->log = log;
	
    		return p;
		}
		
		
		
2.  ngx_memalign（nginx封装内存对齐函数，`/src/os/ngx_alloc.c`）
		
			//宏“NGX_HAVE_POSIX_MEMALIGN”如系统为linux，则在系统检查的时候会设置
		
			#if (NGX_HAVE_POSIX_MEMALIGN)

			void *ngx_memalign(size_t alignment, size_t size, ngx_log_t *log){
    			void  *p;
    			int    err;

    			err = posix_memalign(&p, alignment, size);

    			if (err) {
        			ngx_log_error(NGX_LOG_EMERG, log, err,
                      "posix_memalign(%uz, %uz) failed", alignment, size);
        			p = NULL;
    			}

    			ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "posix_memalign: %p:%uz @%uz", p, size, alignment);

    			return p;
			}

			#elif (NGX_HAVE_MEMALIGN)

			void *ngx_memalign(size_t alignment, size_t size, ngx_log_t *log){
    			void  *p;

    			p = memalign(alignment, size);
    			if (p == NULL) {
        			ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "memalign(%uz, %uz) failed", alignment, size);
    			}

    			ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "memalign: %p:%uz @%uz", p, size, alignment);

    			return p;
			}

			#endif


3.	ngx_palloc（分配内存）
	
		void *ngx_palloc(ngx_pool_t *pool, size_t size){
    		
    		u_char      *m;
    		ngx_pool_t  *p;
			
			
    		if (size <= pool->max) {

        		p = pool->current;

        		do {
        		
        			//内存对齐
            		m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT);

            		if ((size_t) (p->d.end - m) >= size) {
               			p->d.last = m + size;

                		return m;
            		}

            		p = p->d.next;

        		} while (p);
				//内存池中的内存不能满足要求，则添加新的内存块
        		return ngx_palloc_block(pool, size);
    		}
    		
			//要求分配的内存大于最大允许量，则分配大内存块
    		return ngx_palloc_large(pool, size);
		}

4. ngx_pnalloc（分配内存,与ngx_palloc不同的是没有对内存对齐）

		void *ngx_pnalloc(ngx_pool_t *pool, size_t size){
    		u_char      *m;
    		ngx_pool_t  *p;

    		if (size <= pool->max) {

        		p = pool->current;

        		do {
            		m = p->d.last;

            		if ((size_t) (p->d.end - m) >= size) {
                		p->d.last = m + size;

                		return m;
            		}

            		p = p->d.next;

        		} while (p);

        		return ngx_palloc_block(pool, size);
    		}

    		return ngx_palloc_large(pool, size);
		}

5. 关于内存对齐的宏(`/src/core/ngx_config.h`)
	
	
		#ifndef NGX_ALIGNMENT
		#define NGX_ALIGNMENT   sizeof(unsigned long)    /* platform word */
		#endif
	
		#define ngx_align(d, a)     (((d) + (a - 1)) & ~(a - 1))
		#define ngx_align_ptr(p, a)                                                   \
    		(u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))
    
		
	align是内存对齐的模，是2的n次幂。内存地址是对齐的特征：**内存地址是模的倍数，即2进制为后n位的值为0**；
	对齐操作实际是对原地址向后移动了(a-p%a)位。
	![ngx_align_ptr](/assets/image/ngx_align.gif)




