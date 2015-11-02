---
layout: post
title: How much memory does PHP array take?
category: [tech]
tags: [php]
---
I've read an interesting article talking about how much memory does PHP array take.
Here is the summary.

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

Here are the details of each components:

### zval
zval is the internal representation of PHP variables.

{% highlight c %}
struct _zval_struct {
    zvalue_value value;     // The value
    zend_uint refcount__gc; // The number of references to this value (for GC)
    zend_uchar type;        // The type
    zend_uchar is_ref__gc;  // Whether this value is a reference (&)
};
{% endhighlight %}

and zvalue_value looks like this:

{% highlight c %}
typedef union _zvalue_value {
    long lval;                // For integers and booleans
    double dval;              // For floats (doubles)
    struct {                  // For strings
        char *val;            //     consisting of the string itself
        int len;              //     and its length
    } str;
    HashTable *ht;            // For arrays (hash tables)
    zend_object_value obj;    // For objects
} zvalue_value
{% endhighlight %}

The size of zvalue_value is the max of each member, which is str member that takes 12 bytes. Take padding into account, it gives us 16 bytes. So the sum of zval will be 16 + 4 + 1 + 1, which is 22 bytes and padded to 24 bytes in total.

### cyclic GC info
Starts at PHP5.3, it introduced garbage collector for cyclic reference. So it wraps zval into this:
{% highlight c %}
typedef struct _zval_gc_info {
    zval z;
    union {
        gc_root_buffer       *buffered;
        struct _zval_gc_info *next;
    } u;
} zval_gc_info;
{% endhighlight %}
So the sum will be 24 + 8 = 32 bytes.

### allocation header
Here is the unexpected part: In order to keep track of allocated memories, PHP add allocation header to every allocation done throught it:

{% highlight c %}
typedef struct _zend_mm_block {
    zend_mm_block_info info;
#if ZEND_DEBUG
    unsigned int magic;
# ifdef ZTS
    THREAD_T thread_id;
# endif
    zend_mm_debug_info debug;
#elif ZEND_MM_HEAP_PROTECTION
    zend_mm_debug_info debug;
#endif
} zend_mm_block;

typedef struct _zend_mm_block_info {
#if ZEND_MM_COOKIES
    size_t _cookie;
#endif
    size_t _size; // size of the allocation
    size_t _prev; // previous block (not sure what exactly this is)
} zend_mm_block_info;
{% endhighlight %}

I can't understand above well, but when all the options turned off, the size of above structure is 16 bytes.
Now we get 48 bytes for each varibale.

### Buckets
As we know, the Array in PHP is in fact Hash table. Every element is in a bucket. The bucket is:

{% highlight c %}
typedef struct bucket {
    ulong h;                  // The hash (or for int keys the key)
    uint nKeyLength;          // The length of the key (for string keys)
    void *pData;              // The actual data
    void *pDataPtr;           // ??? What's this ???
    struct bucket *pListNext; // PHP arrays are ordered. This gives the next element in that order
    struct bucket *pListLast; // and this gives the previous element
    struct bucket *pNext;     // The next element in this (doubly) linked list
    struct bucket *pLast;     // The previous element in this (doubly) linked list
    const char *arKey;        // The key (for string keys)
} Bucket;
{% endhighlight %}
The size of above structure is 68 bytes, padding to 72 bytes. Every bucket itself is dynamically allocated, adding addtional 16 bytes from allocation header, sums up to 88 bytes. Also, the pointer to each buckets cost 8 bytes, so the final results is 96 bytes.

Here we come at the results that: each element in array costs: 96 bytes for bucket and 48 bytes for zval = 144 bytes total.
10000 element array will costs: 13.73MB.

### Array size allocation
Like vector in STL, size of contailer is not increased one by one. Instead, when size reaches pre-allocated limit, 2 times of the existing memory will allocated. As a result, the actual array size will be larger than the sum of its elements.

### Conclusion
PHP is the BEST language. HAHA..
    
