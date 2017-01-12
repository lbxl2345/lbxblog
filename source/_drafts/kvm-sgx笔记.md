虚拟机中的epc，使用sgx_vm_epc_buffer结构来管理的。其数据结构如下：  
  
  	struct sgx_vm_epc_buffer {
		struct list_head buf_list;
		struct list_head page_list;
		unsigned int nr_pages;
		unsigned long userspace_addr;
		__u32 handle;
	};

Kai Huang添加了sgx_vm.c。

	__alloc_epc_buf:申请一个新的epc buffer，在这里对每一个页，都申请了一个iso_page。它调用sgx_alloc_vm_epc_page来完成具体的申请。这个函数定义在sgx_alloc_epc_page当中，实际上调用的就是sgx_alloc_epc_page。
	
	__get_next_epc_buf_handle():每一个epc buffer都有一个handle，这个handle相当于一个标识，利用handle来找到对应的epc buffer，这个函数用来在申请一个epc_buf的时候，获取它的handle。  
	
	__free_epc_buf():释放epc buffer，这里同样也要完成对应的iso_pages的释放。  
	
	__sgx_map_vm_epc_buffer():

  	