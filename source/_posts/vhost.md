VRing的结构:

	typedef struct VRing
	{
	    unsigned int num;
	    unsigned int align;
	    hwaddr desc;
	    hwaddr avail;
	    hwaddr used;
	} VRing;
VirtQueue的结构:
	
	struct VirtQueue
	{
	    VRing vring;
	    hwaddr pa;
	    uint16_t last_avail_idx;
	    /* Last used index value we have signalled on */
	    uint16_t signalled_used;
	
	    /* Last used index value we have signalled on */
	    bool signalled_used_valid;
	
	    /* Notification enabled? */
	    bool notification;
	
	    uint16_t queue_index;
	
	    int inuse;
	
	    uint16_t vector;
	    void (*handle_output)(VirtIODevice *vdev, VirtQueue *vq);
	    VirtIODevice *vdev;
	    EventNotifier guest_notifier;
	    EventNotifier host_notifier;
	};

virtqueue_pop():从desc table中找到available ring中添加的buffers，映射内存  

	int virtqueue_pop(VirtQueue *vq, VirtQueueElement *elem)
	//这个函数首先获取了desc的物理地址
	hwaddr desc_pa = vq->vring.desc;
	//获取virtio设备
	VirtIODevice *vdev = vq->vdev;
随后，该函数提取出所有descriptors。  
在此之后，利用virtqueue_map_sg进行映射。这一步也就是把desc都映射到了内存中去，其地址和长度，保存在VirtQueueElement的两个iovec结构体数组in_sg和out_sg当中，在这里，iov_base的地址是Qemu中的虚拟地址，这个函数会使用cpu_physical_memory_map函数，从gpa得到对应的hva。  
	
	virtqueue_map_sg(elem->in_sg, elem->in_addr, elem->in_num, 1);
    virtqueue_map_sg(elem->out_sg, elem->out_addr, elem->out_num, 0);
 
### 入口:virtio_blk_handle_output   
QEMU后端，对于virtio-blk的请求处理入口是virtio_blk_handle_output。  
首先通过virtio_blk_get_request取得请求。  virtio_blk_get_request首先调用virtio_blk_alloc_request分配一个VirtIOReq对象，然后调用virtqueue_pop从vring中提取一个请求，并且传递到req的elem域中。  
在此之后，调用virtio_blk_handle_request，函数会根据不同情况具体进行处理。首先通过guest传递过来的type进行不同的处理，判断是毒还是写，这关系到使用out_sg还是in_sg。确定后根据sg设置req的QEMUIOVector对象。第二个参数mrb是一个MultiReqBuffer对象，如果它包含的请求数量达到上限，或者因为类型不同无法合并等，就调用virtio_blk_submit_multireq函数，先提交mrb中的请求，否则这个请求放在mrb中先缓存起来。  
virtio_blk_submit_multireq负责将mrb中缓存的请求提交出去。  

### virtio_blk_handle_Read
首先获取扇区号:  
sector = virtio_ldq_p(VIRTIO_DEVICE(req->dev), &req->out.sector);  
可见req->out.sector中已经包含了对应的扇区信息。  
从读取上来说，函数的调用链如下:  
blk_aio_readv->bdrv_aio_readv->bdrv_co_aio_rw_vector  

### bdrv_co_aio_rw_vector
sector_num:本次要处理的起始扇区  
nb_sectors:这次请求涉及到的扇区数目  
qiov:相应的io vector信息  
cb:请求完成的回调函数  
is_write:是否为写操作  
这个函数首先构造一个BlockAIOCoroutine对象acb，然后通过qemu的协程机制启动一个子协程，利用acb来执行bdrv_co_do_rw函数。  
### 读取
在bdrv_co_do_rw函数中，对于读操作，调用bdrv_co_do_readv，随后再继续调用bdrv_co_do_preadv。  
offset:从磁盘的哪个地方开始读写数据  
bytes:这次操作想要读取多少字节的数据  
qiov:和本次操作相关的io vector  
bdrv_co_do_preadv又进一步调用bdrv_aligned_preadv来处理对齐后的请求。  bdrv_aligned_preadv->bdrv_co_do_readv－>bdrv_co_do_preadv。  
bdrv_co_do_preadv进一步调用了bdrv_aligned_preadv。  
bdrv_aligned_preadv处理对齐后的请求:  
offset是对齐后的要访问的起始位置  
bytes是对齐后要访问的大小  
align是对齐的力度  
qiov是经过对齐处理后的io vector  
然后调用了bdrv_co_readv回调函数，对于raw格式来说，这个回调函数是bdrv_co_readv_em->bdrv_co_io_em，然后进一步调用回调函数bdrv_aio_readv，对于raw格式来说，它是raw_aio_readv，在这个函数中通过QEMU_AIO_READ调用了raw_aio_submit。它会把请求发给AIO去处理了。      


    