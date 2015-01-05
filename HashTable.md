# HashTable结构

## HashTable结构定义

    typedef struct _hashtable {
        uint nTableSize;//哈希表的实际分配长度
        uint nTableMask;//计数器，实际上值为nTableSize-1
        uint nNumOfElements;//存储的元素个数
        ulong nNextFreeElement;//指向下一个空元素的位置
        Bucket *pInternalPointer;//foreach循环时，用于记录当前元素的位置
        Bucket *pListHead;//哈希表队列的头
        Bucket *pListTail;//哈希表队列的尾
        Bucket **arBuckets;//存储的元素数组
        dtor_func_t pDestructor;//析构函数
        zend_bool persistent;
        unsigned char nApplyCount;
        zend_bool bApplyProtection;
    #if ZEND_DEBUG
        int inconsistent;
    #endif
    } HashTable;
    
## HashTable字段说明

* nTableSize：哈希表的实际分配长度，不代表实际的元素的个数，一般有预留的空间，该值一直设置为2的指数倍，如果初始时给定的值不为2的指数倍，
则分配空间时自动调整为2的指数倍
* nTableMask：哈希表的掩码，用于计算存放在哪个bucket的位置，初始化值为nTableSize-1
* nNumOfElements：哈希表实际有多少个元素
* nNextFreeElement：下一个可用的元素索引
* *pInternalPointer：遍历哈希表时使用的内部索引
* *pListHead：哈希表的队列头部
* *pListTail：哈希表的队列尾部
* **arBuckets：哈希表实际元素的指针
* pDestructor：析构函数的指针
* persistent：是否持续,用作pmalloc的persistent参数
* nApplyCount：zend_hash_apply的次数，这个用来限制嵌套遍历的层数，源码里面是限制为3层，例如foreach里面又套了foreach
* bApplyProtection：是否开启嵌套遍历的保护
* inconsistent：debug时用来记录哈希表的状态，HT_OK,HT_IS_DESTROYING,HT_DESTROYED,HT_CLEANING
    
## Bucket结构定义

    typedef struct bucket {
    	ulong h;//数组的数字索引
    	uint nKeyLength;//字符串索引的长度
    	void *pData;//实际数据的存储地址
    	void *pDataPtr;
    	struct bucket *pListNext;
    	struct bucket *pListLast;
    	struct bucket *pNext;
    	struct bucket *pLast;
    	const char *arKey;
    } Bucket;
    
## Bucket字段说

* h：哈希值，索引数组的时候会用到这个值
* nKeyLength：key的长度，如果是关联数组，这个就是那个key串的长度，如果是索引数组，这个长度设置为0，因此可以根据这个来判断是关联数组还是索引数组
* pData：元素的实际内容块的指针，这个与pDataPtr结合使用
* pDataPtr：存进哈希表的元素是一个指针的话，那指针直接放这里，pData指向pDataPtr即可。这样就防止存进一个指针类型数据还去emalloc产生一些内存碎片。
* pListNext：指向哈希链表的下一个元素
* pListLast：指向哈希链表的上一个元素
* pNext：相同哈希值的下一个元素（哈希冲突用） 
* pLast：相同哈希值的上一个元素（哈希冲突用）
* arKey：key串，关联数组用