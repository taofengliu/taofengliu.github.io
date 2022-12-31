# 虚拟文件系统

> ## 虚拟文件系统的作用
&emsp;&emsp;虚拟文件系统（Virtual Filesystem）也可以称为虚拟文件系统转换（Virtual Filesystem Switch，VFS），是一个内核软件层，用来处理与Unix标准文件系统相关的所有系统调用。通过虚拟文件系统，Linux可以支持多种不同类型的文件系统。对用户调用的每个读、写或其它函数，内核都会通过虚拟文件系统将它们转化成实际文件系统所支持的函数。
![虚拟文件系统](/posts/虚拟文件系统/vfs.jpg "虚拟文件系统")

&emsp;&emsp;例如，当用户输入以下shell命令：```$ cp /floppy/TEST /temp/test```。其中 /floppy 是MS-DOS磁盘的一个安装点，而 /temp 是一个标准的第二扩展文件系统（Second Extended Filesystem，Ext2）的目录。此时 cp 程序不需要知道 /floppy/TEST 和 /temp/test 分别是什么文件系统类型，它直接与VFS交互便能实现它的功能。

&emsp;&emsp;VFS支持的文件系统主要有以下三种：

&emsp;&emsp;&emsp;**1.磁盘文件系统**&emsp;磁盘文件系统管理在本地磁盘分区中可用的存储空间和其他可以起到磁盘作用的设备（比如一个USB闪存）。实际上，大多数文件系统都由此演变而来。比如，一些众所周知的文件系统，包括Ext2/3、Reiserfs、FAT和iso9660。所有这些文件系统都使用面向块的介质，必须解决以下问题：如何将文件内容和结构信息存储在目录层次结构上。    
&emsp;&emsp;&emsp;**2.网络文件系统**&emsp;这种文件系统允许访问另一台计算机上的数据，该计算机通过网络连接到本地计算机。在这种情况下，数据实际上存储在一个不同系统的硬件设备上。这意味着内核无需关注文件存取、数据组织和硬件通信的细节，这些由远程计算机的内核处理。对此类文件系统中文件的操作都通过网络连接进行。在进程向文件写数据时，数据使用特定的协议（由具体的网络文件系统决定）发送到远程计算机。接下来远程计算机负责存储传输的数据并通知发送者数据已经到达。尽管如此，即使在内核处理网络文件系统时，仍然需要文件长度、文件在目录层次中的位置以及文件的其他重要信息。它必须也提供函数，使得用户进程能够执行通常的文件相关操作，如打开、读、删除等。由于VFS抽象层的存在，用户空间进程不会看到本地文件系统与网络文件系统之间的区别。    
&emsp;&emsp;&emsp;**3.特殊文件系统**&emsp;最常见的特殊文件系统包括用于进程间通信的pipefs（管道）、sockfs（套接字）、mqueue（POSIX消息队列）。

&emsp;&emsp;在Linux系统中，根目录包含在根文件系统中，这个根文件系统通常就是Ext2或Ext3，其他的文件系统都可以被安装在根文件系统的子目录中。

&emsp;&emsp;除了为所有文件系统实现一个通用的接口之外，VFS还负责控制磁盘高速缓存，例如目录项高速缓存与索引节点高速缓存、页高速缓存。

> ## 通用文件模型
&emsp;&emsp;VFS所隐含的主要思想在于引入了一个通用文件模型，这个文件模型能够表示所有支持的文件系统，囊括了任何文件系统常用的功能集，这使得Linux可以支持差异很大的各种文件系统。要支持一个具体文件系统，内核会将其物理组织结构转换为虚拟文件系统的通用文件模型。

&emsp;&emsp;通用文件模型包含一系列对象（没错就是面向对象的设计思路），其中定义了数据结构和操作数据结构的方法，这些方法其实就是一些函数指针，指向了具体文件系统的实现函数。

> ## VFS的数据结构
&emsp;&emsp;通用文件模型由下列对象组成：

&emsp;**1.超级块对象(Superblock Object)**&emsp;超级块对象存放已安装文件系统的相关信息。

&emsp;**2.索引节点对象(Inode Object)**&emsp;索引节点对象存放具体文件的一般信息，每个索引节点都有一个索引节点号，这个节点号唯一地标识一个文件。

&emsp;**3.文件对象(File Object)**&emsp;文件对象存放打开的文件与进程进行交互的有关信息，这些信息仅仅在进程访问文件时才存在于内核中。

&emsp;**4.目录项对象(Dentry Object)**&emsp;一个目录项是一个路径的组成部分。
### 超级块对象
&emsp;&emsp;超级块对象由```super_block```结构表示，定义在文件```<linux/fs.h>```中，其中包含的主要字段有如下表。
|类型|字段|说明|
|---|-----|----|
|```struct list_head```|```s_list```|指向超级块链表的指针|
|```unsigned long```|```s_blocksize```|以字节为单位的块大小|
|```struct file_system_type *```|```s_type```|文件系统类型|
|```struct super_operations *```|```s_op```|超级块方法|
|```int```|```s_count```|引用计数器|
|```struct list_head```|```s_inodes```|所有索引节点链表|
|```struct list_head```|```s_files```|文件对象链表|
|```void *```|```s_fs_info```|指向特定文件系统的超级块信息的指针|
|```struct semaphore```|```s_lock```|超级块信号量|

&emsp;&emsp;超级块对象中最重要的字段就是```s_op```，它指向超级块的操作函数表。超级块操作函数表由```super_operations```结构体表示，定义在```<linux/fs.h>```中。该结构体的每一项都是一个指向超级块操作函数的指针，超级块操作函数执行文件系统和索引节点的底层操作。当文件系统需要对其超级块进行操作时，首先要在超级块对象中寻找需要的操作方法。常见的超级块操作函数如下表。
|操作函数|说明|
|--------|----|
|```struct inode *alloc_inode(stuct super_block *sb)```|在给定的超级块下创建和初始化一个新的索引节点对象|
|```void destroy_inode(struct inode *inode)```|释放给定的索引节点|
|```void delete_inode(struct inode *inode)```|从磁盘上删除给定的索引节点|
|```void write_inode(struct inode *inode, int wait)```|用于将给定索引节点写入磁盘|

&emsp;&emsp;一部分超级块操作函数是可选的，文件系统可以将不需要的函数指针设为NULL。

### 索引节点对象
&emsp;&emsp;索引节点对象包含了内核在操作文件或目录时的全部信息。对于Unix风格的文件系统来说，这些信息可以从磁盘索引节点直接读入，如果一个文件系统没有索引节点，那么需要从磁盘中提取这些信息。无论采用哪种方式，索引节点对象必须在内存中创建，以便于文件系统使用。

&emsp;&emsp;索引节点对象由```inode```结构体表示，定义在文件```<linux/fs.h>```中，主要字段如下表。
|类型|字段|说明|
|---|-----|----|
|```struct hlist_node```|```i_hash```|用于散列表的指针|
|```struct list_head```|```i_list```|索引节点链表|
|```struct list_head```|```i_sb_list```|超级块链表|
|```struct list_head```|```i_dentry```|目录项链表|
|```unsigned long```|```i_ino```|节点号|
|```atomic_t```|```i_count```|引用计数|
|```unsigned int```|```i_nlink```|硬链接计数|
|```loff_t```|```i_size```|以字节为单位的文件大小|
|```umode_t```|```i_mode```|访问权限|
|```struct file_operations *```|```i_op```|索引节点操作表|

&emsp;&emsp;每个索引节点总是出现在以下三个双向循环链表中：未使用的索引节点链表、正在使用的索引节点链表、脏索引节点链表。所有情况下，指向相邻元素的指针存放在```i_list```字段中。此外，所有索引节点对象也包含在超级块对象的```s_inodes```字段中，索引节点对象的```i_sb_list```字段存放了指向链表相邻元素的指针。

&emsp;&emsp;索引节点对象还存放在一个成为```inode_hashtable```的散列表中。散列表加快了对索引节点的搜索，前提是内核知道了索引节点号及文件所在文件系统对应的超级块对象地址。该散列表采用拉链法解决冲突，冲突链表的前后相邻元素指针由```i_hash```字段保存。

&emsp;&emsp;常见的索引节点操作函数见下表。
|操作函数|说明|
|--------|----|
|```int link(struct dentry *old_dentry, struct inode *dir, struct dentry *dentry)```|该函数被系统调用```link()```调用，用来创建硬链接,硬链接名称由```dentry```参数指定，连接对象是```dir```目录中```old_dentry```目录项所代表的文件|
|```int unlink(struct inode *dir, struct dentry *dentry)```|该函数被系统调用```unlink()```调用，从目录```dir```中删除由目录项```dentry```指定的索引节点对象（硬链接计数减一）|
|```int mkdir(sturct inode *dir, struct dentry *dentry, int mode)```|该函数被系统调用```mkdir()```调用,创建一个新目录|
|```int rmdir(sturct inode *dir, struct dentry *dentry)```|该函数被系统调用```rmdir()```调用,删除```dir```目录中```dentry```目录项代表的文件|

### 目录项对象
&emsp;&emsp;VFS把目录当作文件对待，所以在路径 /bin/vi 中，bin和vi都属于文件，路径中的每个部分都对应有一个索引节点对象表示。路径名查找需要解析路径中的每一个组成部分，不但要确保它有效，还需要进一步寻找路径中的下一个部分。为了方便查找操作，VFS引入目录项的概念。每一个dentry代表路径中的一个特定部分。对于上一个例子来说，/、bin和vi都属于目录项对象，前两个是目录，最后一个是普通文件。

&emsp;&emsp;目录项对象由```dentry```结构体表示，定义在文件```<linux/dcache.h>```中，主要的字段见下表。
|类型|字段|说明|
|---|-----|----|
|```struct list_head```|```d_subdirs```|子目录链表|
|```struct dentry *```|```d_parent```|父目录项|
|```struct dentry_operations *```|```d_op```|目录项操作函数表|
|```struct hlist_node```|```d_hash```|散列表|

&emsp;&emsp;如果遍历路径名中的所有元素并将它们逐个解析为目录项对象，还要到达最深层目录，这是一个很费时的工作，所以内核将目录项对象缓存在目录项缓存中。

### 文件对象
&emsp;&emsp;文件对象是已打开的文件在内存中的表示。该对象由```open()```系统调用创建，由```close()```系统调用撤销。因为一个文件可以被多个进程打开，所以一个文件可能对应多个文件对象。

&emsp;&emsp;文件对象由```file```结构体表示，定义在```<linux/fs.h>```中，主要的字段如下。
|类型|字段|说明|
|---|-----|----|
|```struct file_operations *```|```f_op```|文件操作函数表|
|```mode_t```|```f_mode```|访问模式|
|```loff_t```|```f_pos```|文件当前偏移量|
|```struct file_ra_state```|```f_ra```|预读状态|
|```struct address_space *```|```f_mapping```|页缓存映射|

&emsp;&emsp;文件对象的操作函数是块I/O系统调用的基础，常见的函数如下。
|操作函数|说明|
|--------|----|
|```ssize_t read(struct file *file, char *buf, size_t count, loff_t *offset)```|从给定文件的```offset```偏移处读取```count```字节数据到```buf```中，系统调用```read()```调用它|
|```ssize_t write(struct file *file, char *buf, size_t count, loff_t *offset)```|从```buf```中取出```count```字节数据，写入指定文件的```offset```偏移处，系统调用```write()```调用它|
|```int lock(struct file *file, int cmd, struct file_lock *lock)```|给指定文件上锁|

