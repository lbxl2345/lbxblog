### buffer cache/page cache
linux中存在有两个缓存，buffer cache是针对设备的缓存，而page cache是针对文件的缓存。对于一个ext4文件系统来说，每个文件都有一棵radix树管理文件的缓存页，这些缓存页就是page cache；而对于每个块设备来说，都有一棵radix树来管理数据的缓存块，这些缓存块被称为buffer cache。在常见的linux系统中，page cache通常以4kb为单位，而buffer cache的大小由块设备来决定，通常是512B。总的来说，page cache是对文件数据的缓存，而buffer cache是对设备数据的缓存。  
在linux 2.4之前，这两个cache是有区别的，但这明显会产生一些浪费。因此在2.4之后的内核版本中，这两个cache就被统一化了：使用page cache。如果一个缓存数据既代表文件又代表块，那么buffer cache就直接指向page cache。  
但是buffer cache依然是保留的。因为内核依然需要进行block的I/O。由于大部分block表示的是文件数据，因此它们都通过page cache的形式来缓存。但是剩下的小部分数据不是文件：它们是metadata活着原始的block I/O，这一部分依然由buffer cache来保存。  
linux当中，所有的文件I/O操作，都是通过page cache来实现的。写操作是通过将page cache中对应的页标记为脏页来实现的；读操作是通过从page cache中返回数据来实现的。如果数据还不在cache中，就先把它读到cache里面。  
如果只是研究一般文件的读写，那么就只需要在意page cache，不用去关心buffer cache。
### 页高速缓存
现在我们知道，在linux中，大部分文件都采用了page cache的形式来进行缓存。但是块设备的读写，却是以块的形式来进行的。前面有提到，page cache通常以4kb为单位，而buffer cache则是