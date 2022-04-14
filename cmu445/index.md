# CMU-15-445

> ## Project 1     
### LRUReplacer    
&emsp;&emsp;**理解Pin与Unpin**: 参考《数据库系统概念》，当有线程在读取一个Page时，这个Page是不能被淘汰的，因此需要Pin操作将它移出LRUReplacer。同时Page类中会有计数器记录有几个线程正在读取它的内容，当计数器降为0时，用Unpin操作将其添加进LRUReplacer。             
&emsp;&emsp;实现方式是采用**双端队列与哈希表**，思路与[分布式缓存](https://taofengliu.github.io/%E5%88%86%E5%B8%83%E5%BC%8F%E7%BC%93%E5%AD%98/)中的相同。           
### BufferPoolManager         
&emsp;&emsp;**我的刷盘策略**: 懒刷盘，只有当一个页面被驱逐或者被上层显式调用刷盘时才进行刷盘。              
&emsp;&emsp;**frame_id和获得页框的方法**: frame_id就是```pages_```数组的下标，代表一个内存“页框”，用于存放物理页。一开始所有frame_id均在```free_list_```中，当占用一个frame_id时，将它从```free_list_```取出，并在```page_table_```建立映射，注意此时并不将其放入```replacer_```，因为此时上层调用者会使用返回的Page，```pin_count_```不为零，只有可以进行驱逐的frame_id才会放入```replacer_```。当后期```free_list_```为空时，获得一个新页框的方法是从```replacer_```中驱逐一个页面，得到它对应的页框，此时应注意对驱逐的页面刷盘。        
&emsp;&emsp;**易错点**:              
&emsp;&emsp;&emsp;&emsp;1.```UnpinPgImp()```操作中，```page_table_```找不到要Unpin的页面时，应当返回true。                 
&emsp;&emsp;&emsp;&emsp;2.```DeletePgImp()```中如果成功删除掉一个页，应该Pin一下对应的frame_id。保证一个frame_id只能出现在```free_list_```或```replacer_```中的一个地方，当然也有可能均不出现(此时这个frame_id是正在被使用的，```pin_count_```大于零，无法被驱逐)。              
### ParallelBufferPoolManager     
&emsp;&emsp;用```std::vector<BufferPoolManagerInstance*> bpms_```记录管理的BufferPoolManagerInstance，用```size_t next_instance_```记录下一个页面应当分配在哪一个BufferPoolManagerInstance。注意只有```NewPgImp()```操作需要加锁，这个ParallelBufferPoolManager就是为了提高并发度的。    
### 得分情况   
&emsp;&emsp;因BufferPoolManager中的易错点一开始只有42分，修改完成后达到满分。       
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![得分](/cmu445/score1.jpg)


