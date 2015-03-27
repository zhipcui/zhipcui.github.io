---
layout: post
title:  "nginx数据结构"
date:   2015-1-5
categories: nginx
tag: nginx

---



为了适应多个平台，和性能上的要求，nginx定义了多个高级数据结构，理解并灵活运用这些结构才可以更好的对nginx做开发或者debug。

 * ngx_list_t
 * ngx_queue_t
 * ngx_array_t
 * ngx_hash_t
 * ngx_buf_t
 * ngx_file_t
 
####1、ngx_list_t--链表

**定义所在路径**:
`/src/core/ngx_list.h`、`/src/core/ngx_list.c`

**中心思想**：
采用了分组的方式实现链表。由于链表是动态数据结构，数据的数量是非固定的，所以采取的实现要有具备添加任意个数据项的能力。

**图解**：
![ngx_list_t](/assets/image/nginx-data-struct-list.png)

**定义的结构**：   

	 ngx_list_part_s
	 ngx_list_part_t
	 ngx_list_t

**定义**：

	typedef struct ngx_list_part_s  ngx_list_part_t;

	struct ngx_list_part_s {
    	void             *elts;    //该分组第一个数据项地址
    	ngx_uint_t        nelts;   //该分组已经有多少项
    	ngx_list_part_t  *next;    //下一个分组
	};


	typedef struct {
    	ngx_list_part_t  *last;     //最后一个分组的地址
    	ngx_list_part_t   part;     //第一个分组
    	size_t            size;     //每个数据项的内存空间大小
    	ngx_uint_t        nalloc;   //每一个分组可以容纳的数据项
    	ngx_pool_t       *pool;     //内存池对象，分组的内存有该池管理
	} ngx_list_t;


**定义的函数**
	
	//创建一个链表
	ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);
	
	//往链表中添加一个数据项
	void *ngx_list_push(ngx_list_t *list);
	
	//链表初始化（静态内联函数）
	//static ngx_inline ngx_int_t ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size);
	
	
####2、ngx_queue_t--双向链表

**定义所在路径**:
`/src/core/ngx_queue.h`、`/src/core/ngx_queue.c`

**中心思想**：
nginx定义的双向链表的结构很简单，就是一个包含前后指针的结构体。与链表截然不同的是双向链表并不负责数据元素的内存分配。

**图解**：
无

**定义的结构**：
   
	typedef struct ngx_queue_s  ngx_queue_t;

**定义**：

	struct ngx_queue_s {
    	ngx_queue_t  *prev;
    	ngx_queue_t  *next;
	};


**定义的函数(或者宏)**
	
	//初始化
	ngx_queue_init(q) 
	
	//判断是否为空                                                    
	ngx_queue_empty(h)  
	 
	//在双向链表h头部插入x元素                                                  
	ngx_queue_insert_head(h, x)  
	 
	//同 ngx_queue_insert_head(h, x)                                          
	ngx_queue_insert_after  

    //在双向链表h尾部插入x元素
	ngx_queue_insert_tail(h, x) 
	      
	//返回双向链表h首元素                                    
	ngx_queue_head(h) 
	
	//返回双向链表h最后一个元素                                                    
	ngx_queue_last(h)
	
	//返回双向链表h头部                       
	ngx_queue_sentinel(h)  
	
	//返回q的下一个元素                                              
	ngx_queue_next(q)
	
	//返回q的前一个元素                                               
	ngx_queue_prev(q)
	
	//将x从双向链表中删除                                                     
	ngx_queue_remove(x)
	
	//以q为界限（不包含q）拆分h链表，前半部分为h，后半部分为n。其中q为h链表的一个元素                                                
	ngx_queue_split(h, q, n)
	
	//合并h,n两个双向链表                                              
	ngx_queue_add(h, n)
	
	//返回q所在结构体的地址                                                   
	ngx_queue_data(q, type, link)                                         
	
	//返回双向链表的中心元素，若有N个元素，将返回第(N/2+1)个元素。
	ngx_queue_t *ngx_queue_middle(ngx_queue_t *queue);
	
	//对双向链表排序
	void ngx_queue_sort(ngx_queue_t *queue, ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *));
	
	
**解释**

1.  ngx_queue_data(q, type, link) 
	
	这是定义于`/src/core/ngx_queue.h`的宏，具体内容:
	 
	 	#define ngx_queue_data(q, type, link)   (type *) ((u_char *) q - offsetof(type, link))  
	 
	`offsetof(type,link)`函数返回type类型中link成员的地址偏移量。
	
	例子：
	
		struct Node{
			u_char* str;
			ngx_queue_t qElem;
			ngx_int_t num;
		}Node
		
		Node node;
		ngx_queue_t queue=node.qElem;
		
		//np == &node
		Node * np = ngx_queue_data(queue, Node, qElem)
	                                     
                                                
2.  void ngx_queue_sort(ngx_queue_t *queue, ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *));
   
    对双向链表queue排序，采用的是插入排序算法，实现在`/src/core/ngx_queue.c`。
    参数`cmp`要自己实现。
   
    例子：
   		
   		ngx_int_t compare(const ngx_queue_t * que_a, const ngx_queue_t * que_b){
   			
   			Node * node_a=ngx_queue_data(que_a, Node, qElem);
   			Node * node_b=ngx_queue_data(que_b, Node, qElem);
   			return (node_a->num > node_b->num)
   			
   		}
   		
   		//将对queue做升序排序
   		ngx_queue_sort(&queue,compare);
	


####3、ngx_array_t--数组

**定义所在路径**:
`/src/core/ngx_array.c (.h)`

**中心思想**：
可动态扩张数据项数组

**图解**：
无

**定义的结构**：   

	ngx_array_t

**定义**：

	typedef struct {
    	void        *elts;    //数组首地址
    	ngx_uint_t   nelts;   //数组可容元素数目
    	size_t       size;    //元素内存大小
    	ngx_uint_t   nalloc;  //已含元素数目
    	ngx_pool_t  *pool;    //所属内存池
	} ngx_array_t;


**定义的函数**

	ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
	void ngx_array_destroy(ngx_array_t *a);
	void *ngx_array_push(ngx_array_t *a);
	void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);
	
	static ngx_inline ngx_int_t ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size);

**解释**


1. ngx_array_destroy （销毁数组）

	从代码上可以看出，对于数组的销毁，只有在内存池还未分配给其他结构时，才会销毁成功，否则的话不会回收。所以在使用数组的时候，要注意销毁时机。

		void ngx_array_destroy(ngx_array_t *a){
    		ngx_pool_t  *p;

    		p = a->pool;

    		if ((u_char *) a->elts + a->size * a->nalloc == p->d.last) {
				p->d.last -= a->size * a->nalloc;
    		}

    		if ((u_char *) a + sizeof(ngx_array_t) == p->d.last) {
				p->d.last = (u_char *) a;
    		}
		}

2. ngx_array_push（填充元素,若数组内存不够则发生扩存）

		void *ngx_array_push(ngx_array_t *a){
    		void        *elt, *new;
    		size_t       size;
    		ngx_pool_t  *p;

    		if (a->nelts == a->nalloc) {

        		/* the array is full */

        		size = a->size * a->nalloc;

        		p = a->pool;

        		if ((u_char *) a->elts + size == p->d.last && p->d.last + a->size <= p->d.end){
            	   /*
             		* the array allocation is the last in the pool
             		* and there is space for new allocation
             	    */

            		p->d.last += a->size;
            		a->nalloc++;

        		} else {
            		/* allocate a new array */

            		new = ngx_palloc(p, 2 * size);
            		if (new == NULL) {
                		return NULL;
            		}

            		ngx_memcpy(new, a->elts, size);
            		a->elts = new;
            		a->nalloc *= 2;
        		}
    		}

    		elt = (u_char *) a->elts + a->size * a->nelts;
    		a->nelts++;

    		return elt;
		}


####4、ngx_hash_t--哈希表

**定义所在路径**:
`/src/core/ngx_hash.c (.h)`

**中心思想**：
nginx的hash是静态的，创建之后不能插入和删除，只能查找,所以在初始化的时候就已经确认了hash的表内容，这样就可以做到内存和查找速度的最优化，而nginx为了做到最优，做了一下很精妙的设计。为了理解nginx的hash设计，下面几点要了解：

1. nginx解决的冲突的方法是开放地址法，在插入时发生冲突后，往后遍历桶内下个位置是否为空，若为空则占用，否则知道找到为止。
2. 一个桶可以容纳多个元素，而且桶内的元素的ngx_hash_key(name,len)的值是相等的。
3. 桶的末端有一个结束标志，所以桶的长度＋1。
4. nginx的hash是静态的，创建之后不能插入和删除，只能查找

**图解**：
![ngx_hash_t](/assets/image/ngx_hash_t.png)

**定义的结构**：   

	ngx_hash_elt_t;
	ngx_hash_t;
	ngx_hash_init_t;
	ngx_hash_key_t;
	
	//上面的结构就可以满足了基本hash的实现

	ngx_hash_keys_arrays_t;
	ngx_table_elt_t;
	
**定义**：

	typedef struct {
    	void             *value;
    	u_short           len;
    	u_char            name[1];
	} ngx_hash_elt_t;
	
	typedef struct {
    	ngx_hash_elt_t  **buckets;
    	ngx_uint_t        size;
	} ngx_hash_t;


	typedef struct {
    	ngx_hash_t       *hash;
    	ngx_hash_key_pt   key;

    	ngx_uint_t        max_size;
    	ngx_uint_t        bucket_size;

    	char             *name;
    	ngx_pool_t       *pool;
    	ngx_pool_t       *temp_pool;
	} ngx_hash_init_t;
	
	typedef struct {
    	ngx_str_t         key;
    	ngx_uint_t        key_hash;
    	void             *value;
	} ngx_hash_key_t;
	
	//上面的结构就可以满足了基本hash的实现，但nginx也实现了支持通配符的hash，所以有如下结构
	
	typedef struct {
    	ngx_hash_t        hash;
    	void             *value;
	} ngx_hash_wildcard_t;
	
	typedef struct {
    	ngx_hash_t            hash;
    	ngx_hash_wildcard_t  *wc_head;
    	ngx_hash_wildcard_t  *wc_tail;
	} ngx_hash_combined_t;
	
	typedef struct {
    	ngx_uint_t        hsize;

    	ngx_pool_t       *pool;
    	ngx_pool_t       *temp_pool;

    	ngx_array_t       keys;
    	ngx_array_t      *keys_hash;  //检查是否冲突

    	ngx_array_t       dns_wc_head;
    	ngx_array_t      *dns_wc_head_hash;   //检查是否冲突

    	ngx_array_t       dns_wc_tail;
    	ngx_array_t      *dns_wc_tail_hash;   //检查是否冲突
	} ngx_hash_keys_arrays_t;


	typedef struct {
    	ngx_uint_t        hash;
    	ngx_str_t         key;
    	ngx_str_t         value;
    	u_char           *lowcase_key;
	} ngx_table_elt_t;
	

**定义的函数**

	//hash查找函数
	void *ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len);
	void *ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
	void *ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
	void *ngx_hash_find_combined(ngx_hash_combined_t *hash, ngx_uint_t key, u_char *name, size_t len);
	
	//hash初始化函数
	ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts);
	ngx_int_t ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts);

	//获得字符串hash值函数
	#define ngx_hash(key, c)   ((ngx_uint_t) key * 31 + c)
	
	ngx_uint_t ngx_hash_key(u_char *data, size_t len);
	ngx_uint_t ngx_hash_key_lc(u_char *data, size_t len);
	ngx_uint_t ngx_hash_strlow(u_char *dst, u_char *src, size_t n);


	ngx_int_t ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type);
	ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key, void *value, ngx_uint_t flags);

**解释**

1.  NGX_HASH_ELT_SIZE(name) （获取元素空间大小）

		#define NGX_HASH_ELT_SIZE(name)                                               \
    				(sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))

2. ngx_hash_init（基本hash初始化）

		ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts){
    		u_char          *elts;
    		size_t           len;
    		u_short         *test;
    		ngx_uint_t       i, n, key, size, start, bucket_size;
    		ngx_hash_elt_t  *elt, **buckets;

    		//检查bucket_size是否太小，＋sizeof(void *)是因为每个桶有一个值为空的结束标志项
    		for (n = 0; n < nelts; n++) {
        		if (hinit->bucket_size < NGX_HASH_ELT_SIZE(&names[n]) + sizeof(void *)){
            		ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          		"could not build the %s, you should "
                          		"increase %s_bucket_size: %i",
                          		hinit->name, hinit->name, hinit->bucket_size);
            		return NGX_ERROR;
        		}
    		}

    		//test作为中间临时变量，所以不再内存池中申请
    		test = ngx_alloc(hinit->max_size * sizeof(u_short), hinit->pool->log);
    		if (test == NULL) {
        		return NGX_ERROR;
    		}

    		/*
    		 * 估计桶数量:
    		 * 一个桶可容纳hash_key值相等的多个元素 
    		 * elem_size = (2 * sizeof(void *)) 为一个元素内存大小的最小值
    		 * num_elem_per_bucket = bucket_size / elem_size  每个桶能装多少个元素
    		 * start = nelts /num_elem_per_bucket 需要最少桶数
    		 *
    		 */
    		bucket_size = hinit->bucket_size - sizeof(void *);

    		start = nelts / (bucket_size / (2 * sizeof(void *)));
    		start = start ? start : 1;
			
			//估计是在指定桶数量太大的时候采取的保守估计
    		if (hinit->max_size > 10000 && nelts && hinit->max_size / nelts < 100) {
        		start = hinit->max_size - 1000;
    		}

    		//查找适合的桶大小size
    		for (size = start; size <= hinit->max_size; size++) {

        		ngx_memzero(test, size * sizeof(u_short));

        		for (n = 0; n < nelts; n++) {
            		if (names[n].key.data == NULL) {
                		continue;
            		}

            		//test[key]记录了第key个桶的长度，如果长度大于bucket_size,说明桶的数量太小，应该增大
            		key = names[n].key_hash % size;
            		test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));

		#if 0
            	ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: %ui %ui \"%V\"",
                          size, key, test[key], &names[n].key);
		#endif

            		if (test[key] > (u_short) bucket_size) {
                		goto next;
            		}		
        		}

        		goto found;

    		next:

        		continue;
        	}

    		ngx_log_error(NGX_LOG_WARN, hinit->pool->log, 0,
                  		"could not build optimal %s, you should increase "
                  		"either %s_max_size: %i or %s_bucket_size: %i; "
                  		"ignoring %s_bucket_size",
                  		hinit->name, hinit->name, hinit->max_size,
                  		hinit->name, hinit->bucket_size, hinit->name);

			//找到了桶的数量为size，下面就要确认每一个桶的大小（桶的最后面需要添加一个值为null的多余项，作为桶的结束标志）
		found:

    		//每个桶至少要有一个值为空的结束标志项
    		for (i = 0; i < size; i++) {
        		test[i] = sizeof(void *);
    		}

    		for (n = 0; n < nelts; n++) {
        		if (names[n].key.data == NULL) {
            		continue;
        		}

        		key = names[n].key_hash % size;
        		test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    		}

    		len = 0;
    		//统计所有非空的桶的总长度
    		for (i = 0; i < size; i++) {
        		//空桶不计入总长度
        		if (test[i] == sizeof(void *)) {
            		continue;
        		}

        		//第i个桶在内存对齐之后的长度
        		test[i] = (u_short) (ngx_align(test[i], ngx_cacheline_size));

        		len += test[i];
    		}

    		//为hash的buckets分配指针内存空间
    		if (hinit->hash == NULL) {
        		hinit->hash = ngx_pcalloc(hinit->pool, sizeof(ngx_hash_wildcard_t)
                                             + size * sizeof(ngx_hash_elt_t *));
        		if (hinit->hash == NULL) {
            		ngx_free(test);
            		return NGX_ERROR;
        		}

        		buckets = (ngx_hash_elt_t **)
                      ((u_char *) hinit->hash + sizeof(ngx_hash_wildcard_t));

    		} else {
        		buckets = ngx_pcalloc(hinit->pool, size * sizeof(ngx_hash_elt_t *));
        		if (buckets == NULL) {
            		ngx_free(test);
            		return NGX_ERROR;
        		}
    		}


    		//为每个桶分配空间，总长度为每个桶长度之和
    		elts = ngx_palloc(hinit->pool, len + ngx_cacheline_size);
    		if (elts == NULL) {
        		ngx_free(test);
        		return NGX_ERROR;
    		}

    		elts = ngx_align_ptr(elts, ngx_cacheline_size);

    		//指定每个桶的首地址
    		for (i = 0; i < size; i++) {
        		if (test[i] == sizeof(void *)) {
            		continue;
        		}

        		buckets[i] = (ngx_hash_elt_t *) elts;
        		elts += test[i];  //test[i]的值为第i个桶的长度

    		}

    		//test数组置零，用于在插入值的时候，记录每个桶的偏移量
    		for (i = 0; i < size; i++) {
        		test[i] = 0;
    		}

    		for (n = 0; n < nelts; n++) {
        		if (names[n].key.data == NULL) {
            		continue;
        		}

        		key = names[n].key_hash % size;
        		elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);

        		elt->value = names[n].value;
        		elt->len = (u_short) names[n].key.len;

        		ngx_strlow(elt->name, names[n].key.data, names[n].key.len);

        		test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    		}


    		//在每个非空的桶的最后插入一个值为null的多余记录，用来作为桶的结束标志。
    		for (i = 0; i < size; i++) {
        		if (buckets[i] == NULL) {
            		continue;
        		}

        		elt = (ngx_hash_elt_t *) ((u_char *) buckets[i] + test[i]);

        		elt->value = NULL;
    		}

    		ngx_free(test);

    		hinit->hash->buckets = buckets;
    		hinit->hash->size = size;

		#if 0

    		for (i = 0; i < size; i++) {
        		ngx_str_t   val;
        		ngx_uint_t  key;

        		elt = buckets[i];

        		if (elt == NULL) {
            		ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          		"%ui: NULL", i);
            		continue;
        		}

        		while (elt->value) {
            		val.len = elt->len;
            		val.data = &elt->name[0];

            		key = hinit->key(val.data, val.len);

            		ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          		"%ui: %p \"%V\" %ui", i, elt, &val, key);

            		elt = (ngx_hash_elt_t *) ngx_align_ptr(&elt->name[0] + elt->len,
                                                   sizeof(void *));
        		}
    		}

		#endif

    		return NGX_OK;
		}

3. ngx_hash_add_key（添加key-value对到初始化数组中）
	
	图解：![ngx_hash_add_key](/assets/image/ngx_hash_add_key.png)
	
	
		ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key, void *value, ngx_uint_t flags){
   			
   			size_t           len;
    		u_char          *p;
    		ngx_str_t       *name;
    		ngx_uint_t       i, k, n, skip, last;
    		ngx_array_t     *keys, *hwc;
    		ngx_hash_key_t  *hk;

    		last = key->len;

    		//支持通配符的hash，检查key（str）的格式
    		if (flags & NGX_HASH_WILDCARD_KEY) {

        		/*
         		* supported wildcards:
         		*     "*.example.com", ".example.com", and "www.example.*"
         		*/

       			n = 0;

        		for (i = 0; i < key->len; i++) {

            		//key中只能在头或者尾出现`*`字符，且只能出现一次
            		if (key->data[i] == '*') {
                		if (++n > 1) {
                    		return NGX_DECLINED;
                		}
            		}
            		//key中不能出现连续的`.`字符
            		if (key->data[i] == '.' && key->data[i + 1] == '.') {
                		return NGX_DECLINED;
            		}
        		}

        		//".example.com"类型key
        		if (key->len > 1 && key->data[0] == '.') {
            		skip = 1;
            		goto wildcard;
        		}

        
        		if (key->len > 2) {

            		//"*.example.com"类型key
            		if (key->data[0] == '*' && key->data[1] == '.') {
                		skip = 2;
                		goto wildcard;
            		}

            		//"www.example.*"类型key
            		if (key->data[i - 2] == '.' && key->data[i - 1] == '*') {
                		skip = 0;
                		last -= 2;
                		goto wildcard;
            		}
        		}

        		//确保只支持 "*.example.com", ".example.com", and "www.example.*"这三种通配符，不满足则报错
        		if (n) {
            		return NGX_DECLINED;
        		}
    		}

    		/* exact hash */

    		//取得key的hash key值，如果设置了 “NGX_HASH_READONLY_KEY”标志不转换大小写，否则一致小写

    		k = 0;

    		for (i = 0; i < last; i++) {
        		if (!(flags & NGX_HASH_READONLY_KEY)) {
           	 		key->data[i] = ngx_tolower(key->data[i]);
        		}
        		k = ngx_hash(k, key->data[i]);
    		}

    		k %= ha->hsize;

    		/* check conflicts in exact hash */

    		// 新添加的key会按hash key值装入ha->keys_hash数组的对应位置，所以要检查是否已经有相同的key
    		// ha->keys_hash是二维数组( ngx_array_t   *keys_hash;)

    		name = ha->keys_hash[k].elts;

    		if (name) {
        		for (i = 0; i < ha->keys_hash[k].nelts; i++) {
            		if (last != name[i].len) {
                		continue;
            		}

            		if (ngx_strncmp(key->data, name[i].data, last) == 0) {
                		return NGX_BUSY;
            		}
        		}

    		} else {
        		if (ngx_array_init(&ha->keys_hash[k], ha->temp_pool, 4,sizeof(ngx_str_t))!= NGX_OK){
            		return NGX_ERROR;
        		}
    		}

    		name = ngx_array_push(&ha->keys_hash[k]);
    		if (name == NULL) {
        		return NGX_ERROR;
    		}

    		*name = *key;


    		/* 将新来的key添加入ha.keys数组中
     		 * 所有的key都会装入ha.keys数组中，由于上面的处理，所以不可能再出现相同的key
     		 */

    		hk = ngx_array_push(&ha->keys);
    		if (hk == NULL) {
        		return NGX_ERROR;
    		}

    		hk->key = *key;
    		hk->key_hash = ngx_hash_key(key->data, last);
    		hk->value = value;

    		return NGX_OK;


		wildcard:

    		/* wildcard hash */

    		k = ngx_hash_strlow(&key->data[skip], &key->data[skip], last - skip);

    		k %= ha->hsize;

    		if (skip == 1) {

        		/* check conflicts in exact hash for ".example.com" */

        		// 将".example.com"样式的key装入 ha->keys_hash数组中

        		name = ha->keys_hash[k].elts;

        		if (name) {
            		len = last - skip;

            		for (i = 0; i < ha->keys_hash[k].nelts; i++) {
                		if (len != name[i].len) {
                    		continue;
                		}

                		if (ngx_strncmp(&key->data[1], name[i].data, len) == 0) {
                    		return NGX_BUSY;
                		}
            		}

        		} else {
            		if (ngx_array_init(&ha->keys_hash[k], ha->temp_pool, 4, sizeof(ngx_str_t))!= NGX_OK){
                		return NGX_ERROR;
            		}
        		}

        		name = ngx_array_push(&ha->keys_hash[k]);
        		if (name == NULL) {
            		return NGX_ERROR;
        		}

        		name->len = last - 1;
        		name->data = ngx_pnalloc(ha->temp_pool, name->len);
        		if (name->data == NULL) {
            		return NGX_ERROR;
        		}

        		ngx_memcpy(name->data, &key->data[1], name->len);
    		}


    		if (skip) {

        		/*
         		 * convert "*.example.com" to "com.example.\0"
         		 *      and ".example.com" to "com.example\0"
         		 */

        		p = ngx_pnalloc(ha->temp_pool, last);
        		if (p == NULL) {
            		return NGX_ERROR;
        		}

        		len = 0;
        		n = 0;

        		for (i = last - 1; i; i--) {
            		if (key->data[i] == '.') {
                		ngx_memcpy(&p[n], &key->data[i + 1], len);
                		n += len;
                		p[n++] = '.';
                		len = 0;
                		continue;
            		}

            		len++;
        		}

        		if (len) {
            		ngx_memcpy(&p[n], &key->data[1], len);
            		n += len;
        		}

        		p[n] = '\0';

        		hwc = &ha->dns_wc_head;
        		keys = &ha->dns_wc_head_hash[k];

    		} else {

        		/* convert "www.example.*" to "www.example\0" */

        		last++;

        		p = ngx_pnalloc(ha->temp_pool, last);
        		if (p == NULL) {
            		return NGX_ERROR;
        		}

        		ngx_cpystrn(p, key->data, last);

        		hwc = &ha->dns_wc_tail;
        		keys = &ha->dns_wc_tail_hash[k];
    		}


    		/* check conflicts in wildcard hash */

    		name = keys->elts;

    		if (name) {
        		len = last - skip;

        		for (i = 0; i < keys->nelts; i++) {
            		if (len != name[i].len) {
                		continue;
            		}

            		if (ngx_strncmp(key->data + skip, name[i].data, len) == 0) {
                		return NGX_BUSY;
            		}
        		}

    		} else {
        		if (ngx_array_init(keys, ha->temp_pool, 4, sizeof(ngx_str_t)) != NGX_OK){
            		return NGX_ERROR;
        		}
    		}

    		name = ngx_array_push(keys);
    		if (name == NULL) {
        		return NGX_ERROR;
    		}

    		name->len = last - skip;
    		name->data = ngx_pnalloc(ha->temp_pool, name->len);
    		if (name->data == NULL) {
        		return NGX_ERROR;
    		}

    		ngx_memcpy(name->data, key->data + skip, name->len);


    		/* add to wildcard hash */

    		hk = ngx_array_push(hwc);
    		if (hk == NULL) {
        		return NGX_ERROR;
    		}

    		hk->key.len = last - 1;
    		hk->key.data = p;
    		hk->key_hash = 0;
    		hk->value = value;

    		return NGX_OK;
		}

	
####ngx_buf_t--缓存

**定义所在路径**:
`/src/core/ngx_buf.h`、`/src/core/ngx_buf.c`

**中心思想**：
buf数据结构用于缓存数据，可以通过ngx_chain_t数据结构将多个buf串联起来形成链表。值得注意的是每一个ngx_chain_t在内存池中申请之后，如若释放则会加进到ngx_pool_t结构的chain域内.


**定义的结构**：   

	 ngx_buf_t
	 ngx_chain_s
	 ngx_bufs_t


**定义**：

	struct ngx_buf_s {
    	u_char          *pos;   //有效数据的头部位置
    	u_char          *last;  //有效数据的尾部位置
    	off_t            file_pos;   //buf的数据位于文件中，有效数据在文件中的头部位置
    	off_t            file_last;  //参考上面

    	u_char          *start;         /* start of buffer */   //buf 内存空间的头部
    	u_char          *end;           /* end of buffer */     //buf 内存空间的尾部
    	ngx_buf_tag_t    tag;
    	ngx_file_t      *file;    //buf的数据位于文件中，指示该文件位置
    	ngx_buf_t       *shadow;


    	/* the buf's content could be changed */
    	unsigned         temporary:1;    //临时可修改buf标志

    	/*
     	* the buf's content is in a memory cache or in a read only memory
     	* and must not be changed
     	*/
    	unsigned         memory:1;       //数据位于内存缓存或者只读内存标志，不可修改内容

    	/* the buf's content is mmap()ed and must not be changed */
    	unsigned         mmap:1;   // 数据通过mmap函数获得，不可修改标志

    	unsigned         recycled:1;
    	unsigned         in_file:1;   //数据位于文件中
    	unsigned         flush:1;
    	unsigned         sync:1;
    	unsigned         last_buf:1;   //该buf是数据的最后一个buf标志，意味着读完该buf就读完整块数据
    	unsigned         last_in_chain:1;  //位于buf链表的最后一个节点标志

    	unsigned         last_shadow:1;
    	unsigned         temp_file:1;

    	/* STUB */ int   num;
	};


	struct ngx_chain_s {
    	ngx_buf_t    *buf;
    	ngx_chain_t  *next;
	};


	typedef struct {
    	ngx_int_t    num;
    	size_t       size;
	} ngx_bufs_t;


**定义的函数(宏)**
	
	//申请一个buf
	#define ngx_alloc_buf(pool)  ngx_palloc(pool, sizeof(ngx_buf_t))
	#define ngx_calloc_buf(pool) ngx_pcalloc(pool, sizeof(ngx_buf_t))
	
	//释放一个chain
	#define ngx_free_chain(pool, cl)   cl->next = pool->chain; \
    									pool->chain = cl
	//申请一个临时buf
	ngx_buf_t *ngx_create_temp_buf(ngx_pool_t *pool, size_t size);
	
	//申请一个chain，并指定buf的个数和buf大小
	ngx_chain_t *ngx_create_chain_of_bufs(ngx_pool_t *pool, ngx_bufs_t *bufs);
	
	//申请一个chain
	ngx_chain_t *ngx_alloc_chain_link(ngx_pool_t *pool)
		

####ngx_file_t--文件

**定义所在路径**:
`/src/core/ngx_file.h`、`/src/core/ngx_file.c`、`/src/os/ngx_files.h`、`/src/os/ngx_files.c`

**中心思想**：
nginx在`/src/os/ngx_files.c`中封装了文件的操作，并且定义了用于处理文件的数据结构。


**定义的结构**：   

	 ngx_file_s
	 ngx_path_t

**定义**

	///src/os/ngx_files.c 对文件属性的封装
	typedef int                      ngx_fd_t;
	typedef struct stat              ngx_file_info_t;
	
	
	#define ngx_open_file_n          "open()"

	#define NGX_FILE_RDONLY          O_RDONLY
	#define NGX_FILE_WRONLY          O_WRONLY
	#define NGX_FILE_RDWR            O_RDWR
	#define NGX_FILE_CREATE_OR_OPEN  O_CREAT
	#define NGX_FILE_OPEN            0
	#define NGX_FILE_TRUNCATE        (O_CREAT|O_TRUNC)
	#define NGX_FILE_APPEND          (O_WRONLY|O_APPEND)
	#define NGX_FILE_NONBLOCK        O_NONBLOCK
	
	struct ngx_file_s {
   		ngx_fd_t                   fd;   //文件描述符
    	ngx_str_t                  name;  //文件名
    	ngx_file_info_t            info;   //文件的stat域，可获取关于文件的属性

    	off_t                      offset;   //文件偏移量,
    	off_t                      sys_offset;

    	ngx_log_t                 *log;

	#if (NGX_HAVE_FILE_AIO)
    	ngx_event_aio_t           *aio;
	#endif

    	unsigned                   valid_info:1;
    	unsigned                   directio:1;
	};
	
	typedef struct {
    	ngx_str_t                  name;     //路径名
    	size_t                     len;      //路径长度
    	size_t                     level[3];  //指示路径每个级的长度

    	ngx_path_manager_pt        manager;
    	ngx_path_loader_pt         loader;
    	void                      *data;

    	u_char                    *conf_file;
    	ngx_uint_t                 line;
	} ngx_path_t;
	






