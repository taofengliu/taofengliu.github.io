# CMU-15-445

> ## Project 1     
### LRUReplacer    
&emsp;&emsp;**理解Pin与Unpin**: 参考《数据库系统概念》，当有线程在读取一个 Page 时，这个 Page 是不能被淘汰的，因此需要Pin操作将它移出 LRUReplacer 。同时 Page 类中会有计数器记录有几个线程正在读取它的内容，当计数器降为 0 时，用 Unpin 操作将其添加进 LRUReplacer 。             
&emsp;&emsp;实现方式是采用**双端队列与哈希表**，思路与[分布式缓存](https://taofengliu.github.io/%E5%88%86%E5%B8%83%E5%BC%8F%E7%BC%93%E5%AD%98/)中的相同。           
### BufferPoolManager         
&emsp;&emsp;**我的刷盘策略**: 懒刷盘，只有当一个页面被驱逐或者被上层显式调用刷盘时才进行刷盘。              
&emsp;&emsp;**frame_id和获得页框的方法**: frame_id 就是```pages_```数组的下标，代表一个内存“页框”，用于存放物理页。一开始所有frame_id均在```free_list_```中，当占用一个 frame_id 时，将它从```free_list_```取出，并在```page_table_```建立映射，注意此时并不将其放入```replacer_```，因为此时上层调用者会使用返回的Page，```pin_count_```不为零，只有可以进行驱逐的frame_id才会放入```replacer_```。当后期```free_list_```为空时，获得一个新页框的方法是从```replacer_```中驱逐一个页面，得到它对应的页框，此时应注意对驱逐的页面刷盘。        
&emsp;&emsp;**易错点**:              
&emsp;&emsp;&emsp;&emsp;1.```UnpinPgImp()```操作中，```page_table_```找不到要Unpin的页面时，应当返回 true 。                 
&emsp;&emsp;&emsp;&emsp;2.```DeletePgImp()```中如果成功删除掉一个页，应该 Pin 一下对应的 frame_id 。保证一个 frame_id 只能出现在```free_list_```或```replacer_```中的一个地方，当然也有可能均不出现(此时这个 frame_id 是正在被使用的，```pin_count_```大于零，无法被驱逐)。              
### ParallelBufferPoolManager     
&emsp;&emsp;用```std::vector<BufferPoolManagerInstance*> bpms_```记录管理的 BufferPoolManagerInstance ，用```size_t next_instance_```记录下一个页面应当分配在哪一个 BufferPoolManagerInstance 。注意只有```NewPgImp()```操作需要加锁，这个 ParallelBufferPoolManager 就是为了提高并发度的。    
### 得分情况与优化     
&emsp;&emsp;因 BufferPoolManager 中的易错点一开始只有 42 分，修改完成后达到满分，Time 13.01 ，排名 212 。后进行优化，将双向队列中的智能指针改为原始指针，并在```FlushPgImp()```和```FlushAllPgsImp()```中将 Page 的```is_dirty_```设为 false ，避免重复刷盘。优化后 Time 9.63 ，排名 84 。       
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![得分](/cmu445/score1.jpg)

> ## Project 2     
### 参考资料         
&emsp;&emsp;[Extendible Hash Table](http://www.mathcs.emory.edu/~cheung/Courses/554/Syllabus/3-index/extensible-hashing.html)          
&emsp;&emsp;[Extendible Hashing (Dynamic approach to DBMS)](https://www.geeksforgeeks.org/extendible-hashing-dynamic-approach-to-dbms/)          
&emsp;&emsp;**一定要仔细看Project提供的已有代码！**
### 组织结构说明     
&emsp;&emsp;一个Page的大小在```config.h```中给出，为 4096 byte，那么如何确定```BUCKET_ARRAY_SIZE```呢？```BUCKET_ARRAY_SIZE```是一页中可以存放的键值对数，每一个键值对还需要两bit的readable和occupied标志，所以可以通过算式```PAGE_SIZE/(sizeof (MappingType) + 0.25)```分子分母同乘4得到```hash_table_page_defs.h```中的```#define BUCKET_ARRAY_SIZE (4 * PAGE_SIZE / (4 * sizeof(MappingType) + 1))```。          
&emsp;&emsp;可以注意到```Page```类中```data_```的大小就是一个 PAGE_SIZE ，同时也注意到```HashTableBucketPage```没有构造函数和析构函数并且存放数据的```array_```长度为零，那如何产生一个```HashTableBucketPage```对象？在课程官网有提到```reinterpret_cast```，分配一个```HashTableBucketPage```对象时调用```auto bucket_page = reinterpret_cast<HashTableBucketPage<int, int, IntComparator> *>(bpm->NewPage(&bucket_page_id, nullptr)->GetData());```，这时会按照```Page```类中```GetData()```返回的大小为 PAGE_SIZE 的数据分配内存，然后被```HashTableBucketPage```指针指向，```HashTableBucketPage```指针指向的内存大小就是一个 PAGE_SIZE ，因为```array_```变量在类的尾部声明，就可以使用```array_```来访问页中数据，即使```array_```的长度被声明为 0 。```HashTableDirectoryPage```的内存分配同理。     
&emsp;&emsp;页中的键值对是以```std::pair<KeyType, ValueType>```的形式存放的，```KeyType```是一个模板类，内部是一个长度为```KeySize```的定长 char 数组。```ValueType```是```Rid```类，存放长度为 64 bit 的信息。           
&emsp;&emsp;当一个键值对加入 bucket 后，如果要删除它，是将它对应的 readable 标志置为 0 ，意味着此时这个位置是一个 tombstone 。          
&emsp;&emsp;```DIRECTORY_ARRAY_SIZE```为 512 ，所以```GlobalDepth```最大为 9 。        
&emsp;&emsp;```HashTableDirectoryPage```与```ExtendibleHashTable```共用一把```table_latch_```。         
&emsp;&emsp;截止当前 Project ，可见的项目结构如图：            
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![项目结构](/cmu445/act.jpg)          
### 易错点         
&emsp;&emsp;1.千万记住 Unpin 不再使用的 Page ，包括```HashTableDirectoryPage```。       
&emsp;&emsp;2.课程要求使用 least-significant bits 作为 index ，这和很多参考资料上画的是相反的，要注意。      
&emsp;&emsp;3.我给```Insert()```加的是读锁，多个线程同时执行```Insert()```，全部insert到一个满的桶上都会到```SplitInsert()```扩容，为了避免重复扩容，```SplitInsert()```先检查是否可以插入成功并且不扩容。至于为什么Insert加读锁，是因为大部分insert操作不改变HashTable的全局状态，加读锁性能明显更好。```Merge()```同理。         
&emsp;&emsp;4.桶分裂时当前桶可能存在于```bucket_page_ids_```中的多个位置，注意全部设成正确的新值。      
&emsp;&emsp;5.对目录进行 Shrink 之后应该再对所有空桶进行检查是否能合并，因为 Shrink 之后刚刚合并的桶的 SplitBucket 发生了改变，可能继续与空桶合并。     
### 神奇的位运算    
&emsp;&emsp;```HashTableDirectoryPage```和```HashTableBucketPage```中有很多函数用到了几个比较妙的位运算，并且这几个函数对于```ExtendibleHashTable```中的几个骚操作很有帮助。根据参考材料和课程提供的注释使用这些函数即可。    
```cpp
uint32_t HashTableDirectoryPage::GetSplitImageIndex(uint32_t bucket_idx) {
  return bucket_idx ^ (1 << (local_depths_[bucket_idx] - 1));
}
```
```cpp
template <typename KeyType, typename ValueType, typename KeyComparator>
void HASH_TABLE_BUCKET_TYPE::RemoveAt(uint32_t bucket_idx) {
  readable_[bucket_idx >> 3] &= ~(1 << (bucket_idx & 7));
}

template <typename KeyType, typename ValueType, typename KeyComparator>
bool HASH_TABLE_BUCKET_TYPE::IsOccupied(uint32_t bucket_idx) const {
  return occupied_[bucket_idx >> 3] & (1 << (bucket_idx & 7));
}

template <typename KeyType, typename ValueType, typename KeyComparator>
void HASH_TABLE_BUCKET_TYPE::SetOccupied(uint32_t bucket_idx) {
  occupied_[bucket_idx >> 3] |= 1 << (bucket_idx & 7);
}

template <typename KeyType, typename ValueType, typename KeyComparator>
bool HASH_TABLE_BUCKET_TYPE::IsReadable(uint32_t bucket_idx) const {
  return readable_[bucket_idx >> 3] & (1 << (bucket_idx & 7));
}

template <typename KeyType, typename ValueType, typename KeyComparator>
void HASH_TABLE_BUCKET_TYPE::SetReadable(uint32_t bucket_idx) {
  readable_[bucket_idx >> 3] |= 1 << (bucket_idx & 7);
}
```
### 得分情况      
&emsp;&emsp;一共两次有效提交，第一次因为没有注意到易错点 5 只有 75 分，改正后达到满分，Time 32.96 ，排名 38 ，还算是比较靠前的，有时间再优化一下，很多逻辑都还可以简化，但是重构比较麻烦。
