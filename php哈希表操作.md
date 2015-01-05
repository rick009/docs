shTable操作函数

## 初始化操作

	#PHP中的初始化数组
	$array = array();

	#C中的初始化函数
    ZEND_API int _zend_hash_init(HashTable *ht, uint nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
    {
    	uint i = 3;//初始化时创建的HashTable最小的长度为8，如果初始化时传递的nSize长度小于8，则将向上取整到8
    
    	SET_INCONSISTENT(HT_OK);//开启debug时设置哈希表的状态为HT_OK，正常状态
    
    	if (nSize >= 0x80000000) {//数组的最大长度是十进制2147483648
    		/* prevent overflow */
    		ht->nTableSize = 0x80000000;//不能超过无符号整数的最大值
    	} else {
    		while ((1U << i) < nSize) {
    			i++;
    		}
    		ht->nTableSize = 1 << i;//将哈希表的长度调整为不小于nSize的最小的2的倍数
    	}
    
    	ht->nTableMask = 0;	//初始化长度为0，即arBuckets为空，未初始化
    	ht->pDestructor = pDestructor;//析构函数
    	ht->arBuckets = (Bucket**)&uninitialized_bucket;//初始化为uninitialized_bucket=NULL
    	ht->pListHead = NULL;//链表头为null
    	ht->pListTail = NULL;//链表尾也为null
    	ht->nNumOfElements = 0;//哈希表元素个数为0
    	ht->nNextFreeElement = 0;//下一个空闲元素地址为0
    	ht->pInternalPointer = NULL;//内部指针为null
    	ht->persistent = persistent;
    	ht->nApplyCount = 0;
    	ht->bApplyProtection = 1;
    	return SUCCESS;
    }
    
    ZEND_API int _zend_hash_init_ex(HashTable *ht, uint nSize, dtor_func_t pDestructor, zend_bool persistent, zend_bool bApplyProtection ZEND_FILE_LINE_DC)
    {
    	int retval = _zend_hash_init(ht, nSize, pDestructor, persistent ZEND_FILE_LINE_CC);
    
    	ht->bApplyProtection = bApplyProtection;//是否开启嵌套遍历的保护
    	return retval;
    }
    
## 添加或更新

    ZEND_API int _zend_hash_add_or_update(HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest, int flag ZEND_FILE_LINE_DC)
    {
    	ulong h;
    	uint nIndex;
    	Bucket *p;
    #ifdef ZEND_SIGNALS
    	TSRMLS_FETCH();
    #endif
    
    	IS_CONSISTENT(ht);
    
    	ZEND_ASSERT(nKeyLength != 0);
    
    	CHECK_INIT(ht);
    
    	h = zend_inline_hash_func(arKey, nKeyLength);//将关联数组的下标通过time33算法转为ulong类型
    	nIndex = h & ht->nTableMask;
    
    	p = ht->arBuckets[nIndex];
    	while (p != NULL) {
    		if (p->arKey == arKey ||
    			((p->h == h) && (p->nKeyLength == nKeyLength) && !memcmp(p->arKey, arKey, nKeyLength))) {
    				if (flag & HASH_ADD) {
    					return FAILURE;
    				}
    				ZEND_ASSERT(p->pData != pData);
    				HANDLE_BLOCK_INTERRUPTIONS();
    				if (ht->pDestructor) {
    					ht->pDestructor(p->pData);
    				}
    				UPDATE_DATA(ht, p, pData, nDataSize);
    				if (pDest) {
    					*pDest = p->pData;
    				}
    				HANDLE_UNBLOCK_INTERRUPTIONS();
    				return SUCCESS;
    		}
    		p = p->pNext;
    	}
    	
    	if (IS_INTERNED(arKey)) {
    		p = (Bucket *) pemalloc(sizeof(Bucket), ht->persistent);
    		p->arKey = arKey;
    	} else {
    		p = (Bucket *) pemalloc(sizeof(Bucket) + nKeyLength, ht->persistent);
    		p->arKey = (const char*)(p + 1);
    		memcpy((char*)p->arKey, arKey, nKeyLength);
    	}
    	p->nKeyLength = nKeyLength;
    	INIT_DATA(ht, p, pData, nDataSize);
    	p->h = h;
    	CONNECT_TO_BUCKET_DLLIST(p, ht->arBuckets[nIndex]);
    	if (pDest) {
    		*pDest = p->pData;
    	}
    
    	HANDLE_BLOCK_INTERRUPTIONS();
    	CONNECT_TO_GLOBAL_DLLIST(p, ht);
    	ht->arBuckets[nIndex] = p;
    	HANDLE_UNBLOCK_INTERRUPTIONS();
    
    	ht->nNumOfElements++;
    	ZEND_HASH_IF_FULL_DO_RESIZE(ht);		/* If the Hash table is full, resize it */
    	return SUCCESS;
    }
