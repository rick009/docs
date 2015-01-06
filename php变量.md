# PHP变量

## 1. 变量结构体zval

    struct _zval_struct {
    	/* Variable information */
    	zvalue_value value;         //变量的值
    	zend_uint refcount__gc;     //变量的引用次数，用于GC
    	zend_uchar type;            //实际的类型
    	zend_uchar is_ref__gc;      //是否是引用
    };
    此结构体的长度为：zvalue_value16字节+zend_uint4字节+zend_uchar1字节*2=22字节，内存需要对齐，所以占用24字节
    
## 2. 变量值结构体zvalue_value

    typedef union _zvalue_value {
    	long lval;					//整数类型的值
    	double dval;				//浮点数类型的值
    	struct {                    //字符串类型的值
    		char *val;
    		int len;
    	} str;
    	HashTable *ht;				//数组类型的值（哈希表）
    	zend_object_value obj;      //对象类型的值
    	zend_ast *ast;              //????
    } zvalue_value;
    此联合体的最大长度为字符串的长度（指针8字节+int4字节=12字节），由于内存需要对齐，所以占用16字节
    
## 3. zval的封装zend_gc_info

    typedef struct _zval_gc_info {
    	zval z;
    	union {
    		gc_root_buffer       *buffered;
    		struct _zval_gc_info *next;
    	} u;
    } zval_gc_info;
    该结构体只是增加了一个联合体在其中，该联合体的最大长度为指针的长度8字节，所以占用24+8=32字节
    
## 4. 各个元素占用的大小

                                 |  64 bit   | 32 bit
    ---------------------------------------------------
    zval                         |  24 bytes | 16 bytes
    + cyclic GC info             |   8 bytes |  4 bytes
    + allocation header          |  16 bytes |  8 bytes
    
    ===================================================
    zval (value) total           |  48 bytes | 28 bytes
    
    ===================================================
    bucket                       |  72 bytes | 36 bytes
    + allocation header          |  16 bytes |  8 bytes
    + pointer                    |   8 bytes |  4 bytes
    
    ===================================================
    bucket (array element) total |  96 bytes | 48 bytes
    
    ===================================================
    total total                  | 144 bytes | 76 bytes
    