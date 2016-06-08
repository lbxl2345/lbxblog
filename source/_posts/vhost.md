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

buffer_head.h中，buffer_head结构体，包含了sector_t b_blocknr，也即start block number。  
	
	//buffer.c,函数submit_bh_wbc中
	bio->bi_iter.bi_sector = bh->b_blocknr * (bh->b_size >>9 )

	//此处skip是sector编号，是由block * 8计算得到的
	sudo dd if=/dev/sda1 of=dumptest skip=31406544 bs=512 count=1
	cat dumptest
	
每个ext4文件，都有一棵radix tree用于维护这个文件内容的page cache，对块设备上的数据进行cache维护。如果不采用Direct_IO的方式，会先将数据写入page cache。  
直接IO，在用户缓冲区和磁盘之间，直接传输数据。  
缓冲IO，读入页面缓存，再将数据复制到用户空间缓冲区。  

### 从GPA到HVA
virtqueue_map_sg函数会将vring中scatterlist中提取的信息设置到host中的变量in_sg、out_sg、iov_base。  
sg是一个iovec结构体。对于vring中的每一个desc，获取desc_pa，这里desc_pa是virng->desc，也就是vring中desc的物理地址。
这里，放在iov_base中的地址，是Qemu中的虚拟地址，cpu_physical_memory_map将desc里面的gpa，转化到了hva。  
而in_addr，out_addr保存着的是片段的gpa，每一个VirtQueueElement中，都有两个数组，in_addr和out_addr，保存了in/out对应的片段的起始地址(gpa)，而in_sg和out_sg，则是一个iovec结构体数组。再cpu_physical_memory_map函数中，之前所保存的gpa被转换为了hva。

	virtqueue_map_sg(elem->in_sg, elem->in_addr, elem->in_num, 1);
	virtqueue_map_sg(elem->out_sg, elem->out_addr, elem->out_num, 0);
可以看到，elem->in_addr，elem->out_addr，这两个地址，被作为了参数，传入了virtqueue_map_sg中。不妨再看一看这个函数的内部。

	void virtqueue_map_sg(struct iovec *sg, hwaddr *addr,
    size_t num_sg, int is_write)
首先是其定义。iovec就是我们要赋值的iovec，其中长度已经在vring_desc_len中计算并赋值过了。iov_base则是在这个函数中具体赋值的。这里addr就是gpa，num_sg是sg的数量，is_write指明了这个io是写还是读。其中，具体对sg中iov_base赋值的是这么一句：

	sg[i].iov_base = cpu_physical_memory_map(addr[i], &len, is_write);
	
这里的len，是sg[i].iov_len，也即缓冲区的长度。addr[i]也就是前面的out/in_addr[i]。进入这个函数后：

	void *cpu_physical_memory_map(hwaddr addr,
                              hwaddr *plen,
                              int is_write)
	{
	    return address_space_map(&address_space_memory, addr, plen, is_write);
	}
这里，参数address_space_memory是全局的地址空间。

	void *address_space_map(AddressSpace *as,
                        hwaddr addr,
                        hwaddr *plen,
                        bool is_write)
这里，参数as是地址空间，也就是内存区。这个函数的注释是：将一个guest物理内存域映射到host的虚拟地址空间中。

###