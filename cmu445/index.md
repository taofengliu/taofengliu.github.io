# CMU-15-445

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
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![得分](/cmu445/score1.jpg)

> ## Project 2     
### 参考资料         
&emsp;&emsp;[Extendible Hash Table](http://www.mathcs.emory.edu/~cheung/Courses/554/Syllabus/3-index/extensible-hashing.html)          
&emsp;&emsp;[Extendible Hashing (Dynamic approach to DBMS)](https://www.geeksforgeeks.org/extendible-hashing-dynamic-approach-to-dbms/)          
&emsp;&emsp;**一定要仔细看Project提供的已有代码！**
### 组织结构说明     
&emsp;&emsp;一个Page的大小在```config.h```中给出，为 4096 byte，那么如何确定```BUCKET_ARRAY_SIZE```呢？```BUCKET_ARRAY_SIZE```是一页中可以存放的键值对数，每一个键值对还需要两 bit 的 readable 和 occupied 标志，所以可以通过算式```PAGE_SIZE/(sizeof (MappingType) + 0.25)```分子分母同乘4得到```hash_table_page_defs.h```中的```#define BUCKET_ARRAY_SIZE (4 * PAGE_SIZE / (4 * sizeof(MappingType) + 1))```。          
&emsp;&emsp;可以注意到```Page```类中```data_```的大小就是一个 PAGE_SIZE ，同时也注意到```HashTableBucketPage```没有构造函数和析构函数并且存放数据的```array_```长度为零，那如何产生一个```HashTableBucketPage```对象？在课程官网有提到```reinterpret_cast```，分配一个```HashTableBucketPage```对象时调用```auto bucket_page = reinterpret_cast<HashTableBucketPage<int, int, IntComparator> *>(bpm->NewPage(&bucket_page_id, nullptr)->GetData());```，这时会按照```Page```类中```GetData()```返回的大小为 PAGE_SIZE 的数据分配内存，然后被```HashTableBucketPage```指针指向，```HashTableBucketPage```指针指向的内存大小就是一个 PAGE_SIZE ，因为```array_```变量在类的尾部声明，就可以使用```array_```来访问页中数据，即使```array_```的长度被声明为 0 。```HashTableDirectoryPage```的内存分配同理。     
&emsp;&emsp;页中的键值对是以```std::pair<KeyType, ValueType>```的形式存放的，```KeyType```是一个模板类，内部是一个长度为```KeySize```的定长 char 数组。```ValueType```是```RID```(Record ID)类，存放长度为 32 bit 的```slot_num_```与 32 bit 的```page_id_```，其实就是对应数据页的 id 和该 Record 在数据页内的偏移，和 Project 3 有关，现在可以不管。           
&emsp;&emsp;当一个键值对加入 bucket 后，如果要删除它，是将它对应的 readable 标志置为 0 ，意味着此时这个位置是一个 tombstone 。          
&emsp;&emsp;```DIRECTORY_ARRAY_SIZE```为 512 ，所以```GlobalDepth```最大为 9 。        
&emsp;&emsp;```HashTableDirectoryPage```与```ExtendibleHashTable```共用一把```table_latch_```。         
&emsp;&emsp;截止当前 Project ，可见的项目结构如图：            
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![项目结构](/cmu445/act.jpg)          
### 易错点         
&emsp;&emsp;1.千万记住 Unpin 不再使用的 Page ，包括```HashTableDirectoryPage```。       
&emsp;&emsp;2.课程要求使用 least-significant bits 作为 index ，这和很多参考资料上画的是相反的，要注意。      
&emsp;&emsp;3.我给```Insert()```加的是读锁，多个线程同时执行```Insert()```，全部 insert 到一个满的桶上都会到```SplitInsert()```扩容，为了避免重复扩容，```SplitInsert()```先检查是否可以插入成功并且不扩容。至于为什么```Insert()```加读锁，是因为大部分insert操作不改变 HashTable 的全局状态，加读锁性能明显更好。```Merge()```同理。         
&emsp;&emsp;4.桶分裂时当前桶可能存在于```bucket_page_ids_```中的多个位置，注意全部设成正确的新值。      
&emsp;&emsp;5.对目录进行 Shrink 之后应该再对所有空桶进行检查是否能合并，因为 Shrink 之后刚刚合并的桶的 SplitBucket 发生了改变，可能继续与空桶合并。     
### 神奇的位运算    
&emsp;&emsp;```HashTableDirectoryPage```和```HashTableBucketPage```中有很多函数用到了几个比较妙的位运算，并且这几个函数对于```ExtendibleHashTable```中的几个骚操作很有帮助。根据参考材料和课程提供的注释使用这些函数即可。下面是其中几个比较有意思的实现。     
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
> ## Project 3       
### 参考资料    
&emsp;&emsp;[课程PPT](https://15445.courses.cs.cmu.edu/fall2021/slides/11-queryexecution1.pdf)重点看 Processing Models 和 Expression Evaluation 。    
&emsp;&emsp;[CMU15-445 数据库实验全满分通过笔记](https://blog.csdn.net/twentyonepilots/article/details/120868216)     
&emsp;&emsp;[Hash Join](https://zhuanlan.zhihu.com/p/94065716)    
   
### 组织结构说明     
##### 执行相关:    
&emsp;&emsp;```ExecutionEngine```是一个执行计划被执行的地方，它接收上层的执行计划并执行后返回结果集，执行时会调用```ExecuteFactory```的静态方法```CreateExecutor(ExecutorContext *, const AbstractPlanNode *)```新建一个```std::unique_ptr<AbstractExecutor>```用以进行执行操作。它的成员变量如下，其中的```Catalog *catalog_```是用于进行 table creation, table lookup, index creation, and index lookup 的。(```[[maybe_unused]]```是一个 C++17 新特性，详见[Attributes](https://en.cppreference.com/w/cpp/language/attributes))    
```cpp
private:
  /** The buffer pool manager used during query execution */
  [[maybe_unused]] BufferPoolManager *bpm_;
  /** The transaction manager used during query execution */
  [[maybe_unused]] TransactionManager *txn_mgr_;
  /** The catalog used during query execution */
  [[maybe_unused]] Catalog *catalog_;
```
&emsp;&emsp;```ExecutorContext```保存了一个语句的执行过程中可能用到的上下文信息,```ExecuteFactory```创建 Executor 的时候会将```ExecutorContext```传递给```Executor```。成员变量如下。
```cpp
private:
  /** The transaction context associated with this executor context */
  Transaction *transaction_;
  /** The datbase catalog associated with this executor context */
  Catalog *catalog_;
  /** The buffer pool manager associated with this executor context */
  BufferPoolManager *bpm_;
  /** The transaction manager associated with this executor context */
  TransactionManager *txn_mgr_;
  /** The lock manager associated with this executor context */
  LockManager *lock_mgr_;
```
&emsp;&emsp;每个执行都会有一个执行计划类，```ExecuteFactory```创建 Executor 的时候也会把对应的执行计划类传递给```Executor```。    
##### 索引和表相关:    
&emsp;&emsp;```Tuple```可以看作是一个数据行，有着唯一的 RID(Record ID) ，即 Project 2 中哈希表的```ValueType```，```RID```类，存放长度为 32 bit 的```slot_num_```与 32 bit 的```page_id_```，其实就是对应```TablePage```的 id 和该 Record 在数据页内的偏移。哈希表中查到 RID 之后就可以根据 RID 获取对应的数据行。    
&emsp;&emsp;```Schema schema_```保存了一张表中的列信息，主要包含```std::vector<Column> cols```，```Column```即记录一个列的信息，包括这一列的名字、数据类型和在```Tuple```中的偏移量，并且保存了创建一个列的表达式```const AbstractExpression *expr_```。    
&emsp;&emsp;```TableInfo```保存一张表的信息，其中主要包含```Schema schema_```、```std::string name```、```std::unique_ptr<TableHeap> table_```。```TableHeap```代表了一个存在于磁盘的表，用于对表进行真正的修改操作。```TablePage```是表的存储单元，保存了```Tuple```，```TableHeap```中有```TablePage```的双向链表，表内容修改操作发生于```TablePage```。        
&emsp;&emsp;```TablePage```中的注释很好地说明了数据是如何保存在表中的。    
```cpp
/**
 * Slotted page format:
 *  ---------------------------------------------------------
 *  | HEADER | ... FREE SPACE ... | ... INSERTED TUPLES ... |
 *  ---------------------------------------------------------
 *                                ^
 *                                free space pointer
 *
 *  Header format (size in bytes):
 *  ----------------------------------------------------------------------------
 *  | PageId (4)| LSN (4)| PrevPageId (4)| NextPageId (4)| FreeSpacePointer(4) |
 *  ----------------------------------------------------------------------------
 *  ----------------------------------------------------------------
 *  | TupleCount (4) | Tuple_1 offset (4) | Tuple_1 size (4) | ... |
 *  ----------------------------------------------------------------
 *
 */
 ```
&emsp;&emsp;```IndexInfo```记录了一个索引的信息，成员如下，其中```std::unique_ptr<Index> index_```使用的就是 Project 2 实现的 ExtendibleHashTable 。    
```cpp
  /** The schema for the index key */
  Schema key_schema_;
  /** The name of the index */
  std::string name_;
  /** An owning pointer to the index */
  std::unique_ptr<Index> index_;
  /** The unique OID for the index */
  index_oid_t index_oid_;
  /** The name of the table on which the index is created */
  std::string table_name_;
  /** The size of the index key, in bytes */
  const size_t key_size_;
```
&emsp;&emsp;```Catalog```使用```TableInfo```与```IndexInfo```两个结构来对表和索引进行操作。     
### Nested Loop Join    
&emsp;&emsp;Nested Loop Join 写起来感觉怪怪的，贴一下我的代码吧，不知道逻辑对不对，测试是过了。    
```cpp
void NestedLoopJoinExecutor::Init() {
    left_executor_->Init();
    right_executor_->Init();
    first_ = true;
}

bool NestedLoopJoinExecutor::Next(Tuple *tuple, RID *rid) {
    if (first_ && !left_executor_->Next(&left_tuple_, &left_rid_)) {
        return false;
    }
    first_ = false;

    Tuple right_tuple;
    RID right_rid;

    while (true) {
        while (right_executor_->Next(&right_tuple, &right_rid)) {
            if (plan_->Predicate() == nullptr || plan_->Predicate()
                    ->EvaluateJoin(&left_tuple_, left_executor_->GetOutputSchema(),
                                   &right_tuple, right_executor_->GetOutputSchema())
                    .GetAs<bool>()) {
                std::vector<Value> output;
                for (const auto &col : GetOutputSchema()->GetColumns()) {
                    output.push_back(col.GetExpr()->EvaluateJoin(&left_tuple_, left_executor_->GetOutputSchema(), &right_tuple,
                                                                 right_executor_->GetOutputSchema()));
                }
                *tuple = Tuple(output, GetOutputSchema());
                return true;
            }
        }
        if (!left_executor_->Next(&left_tuple_, &left_rid_)) {
            break;
        }
        right_executor_->Init();
    }

    return false;
}
```
### Hash Join     
&emsp;&emsp;这个自己按照我的 Nested Loop Join 的方法写出来线上测试过不了，后来按照[CMU15-445 数据库实验全满分通过笔记](https://blog.csdn.net/twentyonepilots/article/details/120868216)的修改，代码如下。    
```cpp
void HashJoinExecutor::Init() {
    left_child_->Init();
    right_child_->Init();

    // 初始化哈希表阶段
    Tuple left_tuple;
    RID left_rid;
    while (left_child_->Next(&left_tuple, &left_rid)) {
        HashJoinKey hash_key{plan_->LeftJoinKeyExpression()->Evaluate(&left_tuple, left_child_->GetOutputSchema())};
        if (map_.count(hash_key) != 0) {
            map_[hash_key].emplace_back(left_tuple);
        } else {
            map_[hash_key] = std::vector{left_tuple};
        }
    }

    left_index_ = -1;
}

bool HashJoinExecutor::Next(Tuple *tuple, RID *rid) {
    // 探测阶段
    HashJoinKey cur_key;
    if (left_index_ != -1) {
        cur_key.key_ = plan_->RightJoinKeyExpression()->Evaluate(&right_tuple_, right_child_->GetOutputSchema());
    }
    if (left_index_ == -1  // 判断是否第一次执行Next()
        || map_.find(cur_key) == map_.end()  // 判断当前是否已经指向一个能join的右表tuple
        || left_index_ == static_cast<int>(map_[cur_key].size()) // 判断这个能join的右表tuple是否还能跟左表tuple相结合
            ) {
        while (true) {
            if (right_child_->Next(&right_tuple_, &right_rid_)) {
                cur_key.key_ = plan_->RightJoinKeyExpression()->Evaluate(&right_tuple_, right_child_->GetOutputSchema());
                if (map_.find(cur_key) != map_.end()) {
                    left_index_ = 0;
                    break;
                }
            } else {
                return false;
            }
        }
    }

    auto cur_vector = map_.find(cur_key)->second;
    auto cur_left_tuple = cur_vector[left_index_];
    std::vector<Value> output;
    for (const auto &col : GetOutputSchema()->GetColumns()) {
        output.push_back(col.GetExpr()->EvaluateJoin(&cur_left_tuple, left_child_->GetOutputSchema(),
                                                     &right_tuple_, right_child_->GetOutputSchema()));
    }
    *tuple = Tuple(output, GetOutputSchema());
    ++left_index_;
    return true;
}
```
### 得分情况    
&emsp;&emsp;这个 Project 的线上测试因为测试框架的更新导致编译失败，详见[Issue #227](https://github.com/cmu-db/bustub/issues/227#issuecomment-1078992923)。向提交的压缩包中加入 /src/include/storage/page/tmp_tuple_page.h 后即可编译成功。第一次提交因为 Hash Join 超时得到 330 分，修改后达到满分，Time 3.87 ，排名 30 。    
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![得分](/cmu445/score3.jpg)    
> ## Project 4    
### 参考资料    
&emsp;&emsp;《数据库系统概念》    
&emsp;&emsp;[CMU15-445 数据库实验全满分通过笔记](https://blog.csdn.net/twentyonepilots/article/details/120868216)      
&emsp;&emsp;[2021 CMU 15-445 实验笔记](https://www.inlighting.org/archives/cmu-15-445-notes/)     
### 三种隔离级别    
&emsp;&emsp;READ_UNCOMMITED: 读取不需要获得共享锁，写入需要获得排他锁，用完直接放锁。    
&emsp;&emsp;READ_COMMITTED: 要解决脏读的问题，解决方案就是读时上读锁，读完解读锁；写时上写锁，但等到commit时才解写锁；读时上读锁，读完解读锁。这样，永远不会读到未commit的数据，因为上面有写锁。    
&emsp;&emsp;REPEATABLE_READ: 读取和写入均需要锁，需要遵守强两阶段锁规则。    
### 事务状态     
&emsp;&emsp;**注意:** Any failed lock operation should lead to an ABORTED transaction state (implicit abort) and throw an exception. The transaction manager would further catch this exception and rollback write operations executed by the transaction.    
```cpp
 * Transaction states for 2PL:
 *     _________________________
 *    |                         v
 * GROWING -> SHRINKING -> COMMITTED   ABORTED
 *    |__________|________________________^
 *
 * Transaction states for Non-2PL:
 *     __________
 *    |          v
 * GROWING  -> COMMITTED     ABORTED
 *    |_________________________^
 *
```
### 加锁解锁逻辑    
&emsp;&emsp;按照《数据库系统概念》18.1.4 的描述和[2021 CMU 15-445 实验笔记](https://www.inlighting.org/archives/cmu-15-445-notes/)的思路实现即可。下面是课程官网的提示。    
+ ```LockShared(Transaction, RID)```: Transaction txn tries to take a shared lock on record id rid. This should be blocked on waiting and should return true when granted. Return false if transaction is rolled back (aborts).
+ ```LockExclusive(Transaction, RID)```: Transaction txn tries to take an exclusive lock on record id rid. This should be blocked on waiting and should return true when granted. Return false if transaction is rolled back (aborts).
+ ```LockUpgrade(Transaction, RID)```: Transaction txn tries to upgrade a shared to exclusive lock on record id rid. This should be blocked on waiting and should return true when granted. Return false if transaction is rolled back (aborts). This should also abort the transaction and return false if another transaction is already waiting to upgrade their lock.
+ ```Unlock(Transaction, RID)```: Unlock the record identified by the given record id that is held by the transaction.    
### 得分情况和疑惑    
&emsp;&emsp;这个 Project 真写麻了，是四个 Project 里最痛苦的，一开始只有 55 分，和死锁预防的有关的测试全部超时，后来在[CMU15-445 数据库实验全满分通过笔记](https://blog.csdn.net/twentyonepilots/article/details/120868216)看到“如果老事务想上锁，队列中如果有只是在等待并没有获得锁的新事务请求，也要abort它。如果老事务请求继续等待，测试就会超时”，但是这个和《数据库系统概念》讲的又不一样了。虽然最后达到了满分，但是很多实现和教材的有冲突并且我的很多代码逻辑并不完善。Project 4 无排名。    
> ## 总结    
&emsp;&emsp;简单了解C++加上做完 CMU 15-445 共计 13 天，对数据库底层原理加深了理解，也知道了《数据库系统概念》这一本很好的教材，还需要仔细看看。共计 44 次提交，感谢连续十多天熬夜 debug 的自己。    
![提交记录](/cmu445/record.jpg)

