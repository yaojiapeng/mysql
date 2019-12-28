# 1 前言

前段时间刚大致做完基于Taurus 2.0的 **Scalable Lock Manager**[2](http://http//3ms.huawei.com/hi/group/3288655/wiki_5319175.html)





1. 解决了Taurus 1.0**纯写最优配置**（thread_pool_size需要设置很大，例如1000）和**纯读最优配置**（thread_pool_size需要设置很小，例如100）不同的问题，如果用纯读最优配置跑纯写，性能下降25-30%，反之用纯写最优配置跑纯读，性能也下降25-30%，有Scalable Lock Manager + Asynchronous Commit机制后，不需要很多线程，thread_pool_size设置为100就行了，即统一用纯读最优配置即可；
2. 初步评估，新机制下的纯写性能（thread_pool_size设置为100） vs 旧机制下的纯写性能（thread_pool_size设置为1000），高并发下性能（>=256）还能提升 15-20%，具体提升程度随并发数变化有差异；



目前实验室能够测试到的纯写QPS能达到~40W左右（阿里计算方式），虽然比PolarDB的~25W左右已经高了一截，听起来很不错，但是：



1. 实验室用的都是物理机，网络是万M网卡，跟实际线上环境（MCS，网络类似C3NE）比起来，虽然说C3NE网络PPS能力也很强 [6]，但是时延还是要差一些；
2. 线上的DFV池是实例共享的，目前物理机的是独占的；





也就是说，实际上线后，估计跑不出实验室的性能峰值，会劣化一些（当然具体劣化多少得实际评估测试，暂时没有环境），退一万步说，就算线上也要跑出实验室的效果，从笔者个人的角度出发也希望能够纯写QPS往50W看齐，达到目前PolarDB的2倍左右（~25W）。



再来回顾一下笔者之前对纯写性能优化提出的几个方向：



1. **硬件优化**：更快的CPU，内存，网络，例如目前V5节点用的CPU变成了Intel(R) Xeon(R) Gold 6151 CPU @ 3.00GHz，跟V3比性能提升较为明显；
2. **DFV（包含SAL-SQL）优化**：首先是优化PLOG写入时延及提升IOPS，这样可以进一步降低单事务时延，其次是优化Slice写入时延及处理效率，让Slice不拖后腿（虽然Slice处理是异步的，但如果处理太慢不例如长时间运行）；
3. **SQL优化**：首先是要优化多线程并发处理能力，例如Scalable Lock Manager，ReadView优化等，其次是单线程处理能力，例如Plan Cache优化；





硬件优化，显然这个不是笔者能决定的事情，DFV优化嘛，已经提目标给兄弟团队了，我们能发力的就是SQL优化，目前看起来Scalable Lock Manager优化有一定效果，所以笔者希望再接再厉继续评估一些优化的效果，ReadView优化就是其中一个。



# 2 现有ReadView优化机制回顾

现有笔者已知的对ReadView的优化主要有2个：



1. **TecentSQL的优化**：笔者在 [1] 里面做了一些分析，主要是把 trx_sys->mutex 拆锁，rw ids一把锁，还有read view单独的锁；
2. **MariaDB的优化**：笔者在 [7] 里面做了实现分析，这个优化是 MariaDB 10.3 引入的，主要也是把 trx_sys->mutex 这把锁干掉了，换成了 Lock-Free 的实现方式，另外还有一些后续的小优化 [3 - 4]；





可见大家都认为 trx_sys->mutex 这把大锁是并发瓶颈点之一，都是要用不同的方式优化这把锁，虽然看起来TecentSQL的优化方案简单一点，但是代码不开源，所以笔者觉得还是参考MariaDB的代码实现来做Taurus自己的ReadView优化更靠谱。

# 3 Lock-Free ReadView实现原理

## 3.1 关于ReadView

InnoDB支持MVCC多版本 [[8](http://http//mysql.taobao.org/monthly/2018/03/01/)**RC****RR****trx_sys****活跃读写事务数组****可见性**

Read view中保存的trx_sys状态主要包括



RR隔离级别（除了Gap锁之外）和RC隔离级别的差别是创建snapshot时机不同：

- RR隔离级别是在事务开始时刻，确切地说是第一个读操作创建read view的；
- RC隔离级别是在语句开始时刻创建read view的。


创建/关闭read view需要持有trx_sys->mutex，会降低系统性能，5.7版本对此进行**优化**：

- 在事务提交时session会cache只读事务的read view。
- 下次创建read view，判断如果是只读事务并且系统的读写事务状态没有发生变化，即trx_sys的max_trx_id没有向前推进，而且没有新的读写事务产生，就可以重用上次的read view。


Read view创建之后，读数据时比较记录最后更新的trx_id和view的high/low water mark和读写事务数组即可判断可见性。

## 3.2 为什么需要trx_sys->mutex

本质上来说，ReadView就是**活跃读写事务数组****全局****trx_sys**

1. 写事务开始时需要把自己的事务ID加入活跃事务数组；
2. 写事务结束时需要将自己的事务ID从活跃事务数组移除；
3. 只读或写事务需要拷贝当前最新的活跃事务数组内容（获取快照）；
4. Purge线程（**innodb_purge_threads**配置线程数量，缺省为4）需要获取当前所有活跃事务（包括只读和写）的最小low_limit_no信息；
5. ...



更麻烦的是，从实现上来讲，MySQL的事务系统trx_sys把活跃事务相关信息（事务句柄trx_t，事务ID trx_id_t）分成了**3个事务句柄列表**和**1个事务ID列表**：

```
  volatile trx_id_t max_trx_id; /*!< The smallest number not yet
                                assigned as a transaction id or
                                transaction number. This is declared
                                volatile because it can be accessed
                                without holding any mutex during
                                AC-NL-RO view creation. */

  trx_ut_list_t serialisation_list;
  /*!< Ordered on trx_t::no of all the
  currenrtly active RW transactions */

  trx_ut_list_t rw_trx_list; /*!< List of active and committed in
                             memory read-write transactions, sorted
                             on trx id, biggest first. Recovered
                             transactions are always on this list. */

  trx_ut_list_t mysql_trx_list; /*!< List of transactions created
                                for MySQL. All user transactions are
                                on mysql_trx_list. The rw_trx_list
                                can contain system transactions and
                                recovered transactions that will not
                                be in the mysql_trx_list.
                                mysql_trx_list may additionally contain
                                transactions that have not yet been
                                started in InnoDB. */

  trx_ids_t rw_trx_ids; /*!< Array of Read write transaction IDs
                        for MVCC snapshot. A ReadView would take
                        a snapshot of these transactions whose
                        changes are not visible to it. We should
                        remove transactions from the list before
                        committing in memory and releasing locks
                        to ensure right order of removal and
                        consistent snapshot. */
```



这么多数据结构要做到并发访问，最简单当然是用一把大锁 trx_sys->mutex，Lock-Free并发访问？看起来很困难。

## 3.3 如何做到Lock Free

MySQL代码实现里面，事务相关的状态信息都保存在 **trx_t** 这个数据结构里，每个事务对应一个 trx_t 句柄，事务子系统 **trx_sys** 本质上就是对 **trx_t事务句柄** 的管理，当然了，每个InnoDB事务运行会涉及到一大堆状态，但其中跟ReadView有关系的主要是下面这3个状态（**id**/**no**/**state**）：

```
struct trx_t {
  ...
  trx_id_t id; /*!< transaction id */

  trx_id_t no; /*!< transaction serialization number:
               max trx id shortly before the
               transaction is moved to
               COMMITTED_IN_MEMORY state.
               Initially set to TRX_ID_MAX. */

  /** State of the trx from the point of view of concurrency control
  and the valid state transitions.

  Possible states:

  TRX_STATE_NOT_STARTED
  TRX_STATE_FORCED_ROLLBACK
  TRX_STATE_ACTIVE
  TRX_STATE_PREPARED
  TRX_STATE_COMMITTED_IN_MEMORY (alias below COMMITTED)

  Valid state transitions are:

  Regular transactions:
  * NOT_STARTED -> ACTIVE -> COMMITTED -> NOT_STARTED

  Auto-commit non-locking read-only:
  * NOT_STARTED -> ACTIVE -> NOT_STARTED

  XA (2PC):
  * NOT_STARTED -> ACTIVE -> PREPARED -> COMMITTED -> NOT_STARTED

  Recovered XA:
  * NOT_STARTED -> PREPARED -> COMMITTED -> (freed)

  XA (2PC) (shutdown or disconnect before ROLLBACK or COMMIT):
  * NOT_STARTED -> PREPARED -> (freed)

  Disconnected XA can become recovered:
  * ... -> ACTIVE -> PREPARED (connected) -> PREPARED (disconnected)
  Disconnected means from mysql e.g due to the mysql client disconnection.
  Latching and various transaction lists membership rules:

  XA (2PC) transactions are always treated as non-autocommit. */

  trx_state_t state;
  ...
};
```



上述这些事务状态可能会存在并发访问，主要的并发冲突在于**遍历**和**写入**，即有的线程正在更改当前事务状态，但有的线程正在遍历所有事务句柄获取状态（例如获取快照），当然主要是用 trx_sys->mutex 这把全局大锁来控制并发冲突的，显而易见这会是一个明显的锁竞争点。



针对这一点，可以引入了LOCK-FREE的HASH来管理事务句柄，**LF_HASH**无锁算法基于论文 [Split-Ordered Lists: Lock-Free Extensible Hash Tables](http://people.csail.mit.edu/shanir/publications/Split-Ordered_Lists.pdf) [10]，实现还比较复杂。 注：实际上LF_HASH很早就被应用于Performance Schema，算是比较成熟的代码模块。



关于LF_HASH的实现原理，有兴趣的可以阅读论文，从本文的角度知道基本原理就可以了。



# 4 Lock-Free ReadView核心代码实现

## 4.1 关于LF_HASH

MySQL 8.0代码（Taurus SQL基线代码）里面，**LF_HASH**

```
include/lf.h
mysys/lf_alloc-pin.cc
mysys/lf_dynarray.cc
mysys/lf_hash.cc
```



目前在SQL层的Metadata Lock子系统和Performance Schema里面有用到 [[11](http://http//http//mysql.taobao.org/monthly/2014/11/05/)]，例如：

```
/**
  A collection of all MDL locks. A singleton,
  there is only one instance of the map in the server.
*/

class MDL_map {
  ...
  /** LF_HASH with all locks in the server. */
  LF_HASH m_locks;
  ...
};
```



但MySQL 8.0原生的LF_HASH实现支持并发**insert**/**delete**/**find**，但缺乏遍历（**iterate**）功能，需要对 **my_lfind** 函数调用做一点点改动，并且增加 **lf_hash_iterate** 函数调用：

```
typedef bool lf_hash_walk_func(void *, void *);

/**
  Walk the list, searching for an element or invoking a callback.

  Search for hashnr/key/keylen in the list starting from 'head' and position the
  cursor. The list is ORDER by hashnr, key

  @param head         start walking the list from this node
  @param cs           charset for comparing keys, nullptr if callback is used
  @param hashnr       hash number to searching for
  @param key          key to search for OR data for the callback
  @param keylen       length of the key to compare, 0 if callback is used
  @param cursor       for returning the found element
  @param pins         see lf_alloc-pin.cc
  @param callback     callback action, invoked for every element

  @note
    cursor is positioned in either case
    pins[0..2] are used, they are not removed on return
    callback might see some elements twice (because of retries)

  @return
    if find: 0 - not found
             1 - found
    if callback:
             0 - ok
             1 - error (callback returned true)
*/
static int my_lfind(std::atomic<LF_SLIST *> *head, CHARSET_INFO *cs,
                    uint32 hashnr, const uchar *key, size_t keylen,
                    CURSOR *cursor, LF_PINS *pins,
                    lf_hash_walk_func *callback) {
  uint32 cur_hashnr;
  const uchar *cur_key;
  size_t cur_keylen;
  LF_SLIST *link;

  /* should not be set both */
  DBUG_ASSERT((cs == nullptr) || (callback == nullptr));
  /* should not be set both */
  DBUG_ASSERT((keylen == 0) || (callback == nullptr));

retry:
  cursor->prev = head;
  do /* PTR() isn't necessary below, head is a dummy node */
  {
    cursor->curr = (LF_SLIST *)(*cursor->prev);
    lf_pin(pins, 1, cursor->curr);
  } while (*cursor->prev != cursor->curr && LF_BACKOFF);
  for (;;) {
    if (unlikely(!cursor->curr)) {
      return 0; /* end of the list */
    }
    do {
      /* QQ: XXX or goto retry ? */
      link = cursor->curr->link.load();
      cursor->next = PTR(link);
      lf_pin(pins, 0, cursor->next);
    } while (link != cursor->curr->link && LF_BACKOFF);
    cur_hashnr = cursor->curr->hashnr;
    cur_key = cursor->curr->key;
    cur_keylen = cursor->curr->keylen;
    if (*cursor->prev != cursor->curr) {
      (void)LF_BACKOFF;
      goto retry;
    }
    if (!DELETED(link)) {
      if (unlikely(callback != nullptr)) {
        if ((cur_hashnr & 1) > 0 &&
            callback(cursor->curr + 1,
                     const_cast<void *>(static_cast<const void *>(key)))) {
          return 1;
        }
      } else if (cur_hashnr >= hashnr) {
        int r = 1;
        if (cur_hashnr > hashnr ||
            (r = my_strnncoll(cs, (uchar *)cur_key, cur_keylen, (uchar *)key,
                              keylen)) >= 0) {
          return !r;
        }
      }
      cursor->prev = &(cursor->curr->link);
      lf_pin(pins, 2, cursor->curr);
    } else {
      /*
        we found a deleted node - be nice, help the other thread
        and remove this deleted node
      */
      if (atomic_compare_exchange_strong(cursor->prev, &cursor->curr,
                                         cursor->next)) {
        lf_pinbox_free(pins, cursor->curr);
      } else {
        (void)LF_BACKOFF;
        goto retry;
      }
    }
    cursor->curr = cursor->next;
    lf_pin(pins, 1, cursor->curr);
  }
}

/**
  Iterate over all elements in hash and call function with the element.

  @note
  If one of 'callback' invocations returns true the iteration aborts.
  'action' might see some elements twice!

  @return 0 if ok, or 1 if error (action returned true)
*/
int lf_hash_iterate(LF_HASH *hash, LF_PINS *pins, lf_hash_walk_func *callback,
                    void *argument) {
  CURSOR cursor;
  uint bucket = 0;
  int res;
  std::atomic<LF_SLIST *> *el;

  el = static_cast<std::atomic<LF_SLIST *> *>(
      lf_dynarray_lvalue(&hash->array, bucket));
  if (unlikely(el == nullptr)) {
    /* if there's no bucket==0, the hash is empty */
    return 0;
  }
  if (*el == nullptr && unlikely(initialize_bucket(hash, el, bucket, pins))) {
    /* if there's no bucket==0, the hash is empty */
    return 0;
  }

  res = my_lfind(el, nullptr, 0, static_cast<uchar *>(argument), 0, &cursor,
                 pins, callback);

  lf_unpin(pins, 2);
  lf_unpin(pins, 1);
  lf_unpin(pins, 0);

  return res;
}
```



值得注意的是，为了支持更加灵活的遍历，**lf_hash_iterate**支持用户自定义callback，callback函数可以根据所遍历节点的内容控制遍历进度，如果不指定callback（nullptr），那么**my_lfind**函数就退化至原有的查找功能。

## 4.2 LF_HASH表管理事务句柄

既然是LF_HASH，那自然要有**哈希表项**

```
struct rw_trx_hash_element_t {
  rw_trx_hash_element_t() : trx(nullptr) {
    mutex_create(LATCH_ID_RW_TRX_HASH_ELEMENT, &mutex);
  }

  ~rw_trx_hash_element_t() { mutex_free(&mutex); }

  /* lf_hash_init() relies on this to be first in the struct. */
  trx_id_t id;

  std::atomic<trx_id_t> no;

  trx_t *trx;

  ib_mutex_t mutex;
};
```



然后就是无锁哈希表的定义 **rw_trx_hash_t**：

```
/** Wrapper around LF_HASH to store set of in-memory read-write transactions. */
class rw_trx_hash_t {
  LF_HASH hash;

  /** Constructor callback for lock-free allocator.

  Object is just allocated and is not yet accessible via rw_trx_hash by
  concurrent threads. Object can be reused multiple times before it is freed.
  Every time object is being reused initialize() callback is called. */
  static void rw_trx_hash_constructor(uchar *arg);

  /** Destructor callback for lock-free allocator.

  Object is about to be freed and is not accessible via rw_trx_hash by
  concurrent threads. */
  static void rw_trx_hash_destructor(uchar *arg);

  /** Destructor callback for lock-free allocator.

  This destructor is used at shutdown. It frees remaining transaction objects.

  XA PREPARED transactions may remain if they haven't been committed or rolled
  back. ACTIVE transactions may remain if startup was interrupted or server is
  running in read-only mode or for certain srv_force_recovery levels. */
  static void rw_trx_hash_shutdown_destructor(uchar *arg);

  /** Initializer callback for lock-free hash.

  Object is not yet accessible via rw_trx_hash by concurrent threads, but is
  about to become such. Object id can be changed only by this callback and
  remains the same until all pins to this object are released.

  Object trx can be changed to 0 by erase() under object mutex protection,
  which indicates it is about to be removed from lock-free hash and become not
  accessible by concurrent threads. */
  static void rw_trx_hash_initialize(rw_trx_hash_element_t *element,
                                     trx_t *trx);

  /** Gets LF_HASH pins.

  Pins are used to protect object from being destroyed or reused. They are
  normally stored in trx object for quick access. If caller doesn't have trx
  available, we try to get it using current_trx(). If caller doesn't have trx at
  all, temporary pins are allocated. */
  LF_PINS *get_pins(trx_t *trx);

  struct eliminate_duplicates_arg {
    trx_ids_t ids;
    lf_hash_walk_func *action;
    void *argument;

    eliminate_duplicates_arg(size_t size, lf_hash_walk_func *act, void *arg)
        : action(act), argument(arg) {
      ids.reserve(size);
    }
  };

  static bool eliminate_duplicates(rw_trx_hash_element_t *element,
                                   eliminate_duplicates_arg *arg);

#ifdef UNIV_DEBUG
  static void validate_element(trx_t *trx);

  struct debug_iterator_arg {
    lf_hash_walk_func *action;
    void *argument;
  };

  static bool debug_iterator(rw_trx_hash_element_t *element,
                             debug_iterator_arg *arg);
#endif /* UNIV_DEBUG */

 public:
  void init();

  void destroy();

  /** Releases LF_HASH pins.

  Must be called by thread that owns trx_t object when the later is being
  "detached" from thread (e.g. released to the pool by trx_free()). Can be
  called earlier if thread is expected not to use rw_trx_hash.

  Since pins are not allowed to be transferred to another thread,
  initialisation thread calls this for recovered transactions. */
  void put_pins(trx_t *trx);

  /** Finds trx object in lock-free hash with given id.

  Only ACTIVE or PREPARED trx objects may participate in hash. Nevertheless the
  transaction may get committed before this method returns.

  With do_ref_count == false the caller may dereference returned trx pointer
  only if lock_sys.mutex was acquired before calling find().

  With do_ref_count == true caller dereferemce trx even if it is not holding
  lock_sys.mutex. Caller is responsible for calling trx->release_reference()
  when it is done playing with trx.

  Ideally this method should get caller rw_trx_hash_pins along with trx object
  as a parameter, similar to insert() and erase(). However most callers lose trx
  early in their call chains and it is not that easy to pass them through.

  So we take more expensive approach: get trx through current_thd()->ha_data.
  Some threads don't have trx attached to THD, and at least server
  initialisation thread, fts_optimize_thread, srv_master_thread,
  dict_stats_thread, srv_monitor_thread, btr_defragment_thread don't even have
  THD at all. For such cases we allocate pins only for duration of search and
  free them immediately.

  This has negative performance impact and should be fixed eventually (by
  passing caller_trx as a parameter). Still stream of DML is more or less Ok.

  @return pointer to trx or nullptr if not found */
  trx_t *find(trx_t *caller_trx, trx_id_t trx_id, bool do_ref_count);

  /** Inserts trx to lock-free hash.

  Object becomes accessible via rw_trx_hash. */
  void insert(trx_t *trx);

  /** Removes trx from lock-free hash.

  Object becomes not accessible via rw_trx_hash. But it still can be pinned by
  concurrent find(), which is supposed to release it immediately after it sees
  object trx is nullptr. */
  void erase(trx_t *trx);

  /** Returns the number of elements in the hash.

  The number is exact only if hash is protected against concurrent modifications
  (e.g., single threaded startup or hash is protected by some mutex). Otherwise
  the number maybe used as a hint only, because it may change even before this
  method returns. */
  uint32_t size();

  /** Iterates the hash.

  @param caller_trx used to get/set pins
  @param action     called for every element in hash
  @param argument   opque argument passed to action

  May return the same element multiple times if hash is under contention. If
  caller doesn't like to see the same transaction multiple times, it has to call
  iterate_no_dups() instead.

  May return element with committed transaction. If caller doesn't like to see
  committed transactions, it has to skip those under element mutex:

    mutex_enter(&element->mutex);
    trx_t *trx = element->trx;
    if (trx != nullptr) {
      // trx is protected against commit in this branch
    }
    mutex_exit(&element->mutex);

  May miss concurrently inserted transactions.

  @return 0 if iteration completed successfuly, or 1 if iteration was
  interrupted (action returned true) */
  int iterate(trx_t *caller_trx, lf_hash_walk_func *action, void *argument);

  int iterate(lf_hash_walk_func *action, void *argument);

  /** Iterates the hash and eliminates duplicate elements.

  @sa iterate() */
  int iterate_no_dups(trx_t *caller_trx, lf_hash_walk_func *action,
                      void *argument);

  int iterate_no_dups(lf_hash_walk_func *action, void *argument);
};
```



除了支持 **初始化(init)**/**销毁(destroy)**/**查找(find)**/**插入(insert)**/**删除(erase)** 功能外，rw_trx_hash_t 还支持**遍历(iterate)**功能，但需要特别提到的是遍历由于并发冲突的原因可能会遍历统一的哈希项不止一次，所以回调函数需要考虑这一点，如果想过滤重复项，需要调用 **iterate_no_dups** 接口，原理就是把遍历过的哈希项的事务ID记下来来查找是否有重复（性能差一些）：

```
bool rw_trx_hash_t::eliminate_duplicates(rw_trx_hash_element_t *element,
                                         eliminate_duplicates_arg *arg) {
  for (auto id : arg->ids) {
    if (id == element->id) {
      return false;
    }
  }

  arg->ids.push_back(element->id);
  return arg->action(element, arg->argument);
}

/** Iterates the hash and eliminates duplicate elements.

@sa iterate() */
int rw_trx_hash_t::iterate_no_dups(trx_t *caller_trx, lf_hash_walk_func *action,
                                   void *argument) {
  eliminate_duplicates_arg arg(size() + 32, action, argument);
  return iterate(caller_trx,
                 reinterpret_cast<lf_hash_walk_func *>(eliminate_duplicates),
                 &arg);
}
```

## 4.3 事务管理器trx_sys_t相关接口

基本上所有的事务管理器 的接口都是和**rw_trx_hash** ，这里指的Lock Free主要是说对这个哈希表的操作是无锁的：

```
/** The transaction system central memory data structure. */
struct trx_sys_t {
 private:
  /** To avoid false sharing */
  char pad1[64];
  /** The smallest number not yet assigned as a transaction id or transaction
  number. Accessed and updated with atomic operations. */
  std::atomic<trx_id_t> max_trx_id;

  /** To avoid false sharing */
  char pad2[64];
  /** Solves race conditions between register_rw() and snapshot_ids() as well as
  race condition between assign_new_trx_no() and snapshot_ids().

  @sa register_rw()
  @sa assign_new_trx_no()
  @sa snapshot_ids() */
  std::atomic<trx_id_t> rw_trx_hash_version;

 public:
  /** To avoid false sharing */
  char pad3[64];
  /** Mutex protecting trx list. */
  mutable TrxSysMutex mutex;

  /** To avoid false sharing */
  char pad4[64];
  /** List of all transactions. */
  trx_ut_list_t trx_list;

  /** To avoid false sharing */
  char pad5[64];
  /** Lock-free hash of in-memory read-write transactions. Works faster when
  it's on it's own cache line (tested). */
  rw_trx_hash_t rw_trx_hash;

  char pad6[64]; /*!< To avoid false sharing */

  Rsegs rsegs; /*!< Vector of pointers to rollback
               segments. These rsegs are iterated
               and added to the end under a read
               lock. They are deleted under a write
               lock while the vector is adjusted.
               They are created and destroyed in
               single-threaded mode. */

  Rsegs tmp_rsegs; /*!< Vector of pointers to rollback
                   segments within the temp tablespace;
                   This vector is created and destroyed
                   in single-threaded mode so it is not
                   protected by any mutex because it is
                   read-only during multi-threaded
                   operation. */

  std::atomic<ulint> rseg_history_len;
  /*!< Length of the TRX_RSEG_HISTORY
  list (update undo logs for committed
  transactions), protected by
  rseg->mutex */

  /** To avoid false sharing */
  char pad7[64];
  /*!< For replica only, visible_lsn is the max persist LSN of
  all slices */
  atomic_lsn_t visible_lsn;
  /* For replica only, it's used when preparing a view. */
  std::atomic<trx_id_t> purge_trx_no;

  /** Returns the minimum trx id in rw trx list.

  This is the smallest id for which the trx can possibly be active. (But, you
  must look at trx->state to find out if the minimum trx id transaction itself
  is active, or already committed.

  @return the minimum trx id, or max_trx_id if the trx list is empty */
  trx_id_t get_min_trx_id() {
    trx_id_t id = get_max_trx_id();
    rw_trx_hash.iterate(
        reinterpret_cast<lf_hash_walk_func *>(get_min_trx_id_callback), &id);
    return id;
  }

  /** Determines the maximum transaction id.

  @return maximum currently allocated trx id; will be stale after the next call
  to trx_sys.assign_new_trx_no */
  trx_id_t get_max_trx_id() {
    return max_trx_id.load(std::memory_order_relaxed);
  }

  /** Allocates and assigns new transaction serialisation number.

  There's a gap between max_trx_id increment and transaction serialisation
  number becoming visible through rw_trx_hash. While we're in this gap
  concurrent thread may come and do MVCC snapshot without seeing allocated but
  not yet assigned serialisation number. Then at some point purge thread may
  clone this view. As a result it won't see newly allocated serialisation number
  and may remove "unnecessary" history data of this transaction from rollback
  segments.

  rw_trx_hash_version is intended to solve this problem. MVCC snapshot has to
  wait until max_trx_id == rw_trx_hash_version, which effectively means that all
  transaction serialisation numbers up to max_trx_id are available through
  rw_trx_hash.

  We rely on refresh_rw_trx_hash_version() to issue RELEASE memory barrier so
  that rw_trx_hash_version increment happens after trx->rw_trx_hash_element->no
  becomes available visible through rw_trx_hash.

  @param trx transaction */
  void assign_new_trx_no(trx_t *trx) {
    trx->no = get_new_trx_id_no_refresh();
    trx->rw_trx_hash_element->no.store(trx->no, std::memory_order_relaxed);
    refresh_rw_trx_hash_version();
  }

  /** Takes MVCC snapshot.

  To reduce malloc probability we reserve rw_trx_hash.size() + 32 elements in
  ids.

  For details about get_rw_trx_hash_version() != get_max_trx_id() spin
  @sa register_rw() and @sa assign_new_trx_no().

  We rely on get_rw_trx_hash_version() to issue ACQUIRE memory barrier so that
  loading of rw_trx_hash_version happens before accessing rw_trx_hash.

  To optimise snapshot creation rw_trx_hash.iterate is being used instead of
  rw_trx_hash.iterate_no_dups(). It means that some transaction identifiers may
  appear multiple times in ids.

  @param[in,out] caller_trx used to get access to rw_trx_hash_pins
  @param[out]    ids        array to store registered transaction identifiers
  @param[out]    max_trx_id variable to store max_trx_id value
  @param[out]    mix_trx_no variable to store min(trx->no) value */
  void snapshot_ids(trx_t *caller_trx, ReadView::ids_t *ids,
                    trx_id_t *max_trx_id, trx_id_t *min_trx_no) {
    ut_ad(!mutex_own(&mutex));
    snapshot_ids_arg arg(ids);

    while ((arg.id = get_rw_trx_hash_version()) != get_max_trx_id()) {
      ut_delay(1);
    }
    arg.no = arg.id;

    ids->clear();
    ids->reserve(rw_trx_hash.size() + 32);
    rw_trx_hash.iterate(
        caller_trx, reinterpret_cast<lf_hash_walk_func *>(copy_one_id), &arg);

    *max_trx_id = arg.id;
    *min_trx_no = arg.no;
  }

  /** Initialiser for max_trx_id and rw_trx_hash_version. */
  void init_max_trx_id(trx_id_t value) {
    max_trx_id = rw_trx_hash_version = value;
  }

  /** @return total number of active (non-prepared) transactions */
  ulint any_active_transactions();

  /** Registers read-write transaction.

  Transaction becomes visible to MVCC.

  There's a gap between max_trx_id increment and transaction becoming visible
  through rw_trx_hash. While we're in this gap concurrent thread may come and do
  MVCC snapshot. As a result concurrent readview will be able to observe records
  owned by this transaction even before it is committed.

  rw_trx_hash_version is intendded to solve this problem. MVCC snapshot has to
  wait until max_trx_id == rw_trx_hash_version, which effectively means that all
  transactions up to max_trx_id are available through rw_trx_hash.

  We rely on refresh_rw_trx_hash_version() to issue RELEASE memory barrier so
  that rw_trx_hash_version increment happens after transaction becomes visible
  through rw_trx_hash. */
  void register_rw(trx_t *trx) {
    trx->id = get_new_trx_id_no_refresh();
    rw_trx_hash.insert(trx);
    refresh_rw_trx_hash_version();
  }

  /** For replica only, registers a faked read-write transaction. */
  void register_rw_replica(trx_t *trx) {
    /* trx->id and trx_sys->max_trx_id were already set. */
    rw_trx_hash.insert(trx);
    rw_trx_hash_version.store(get_max_trx_id(), std::memory_order_release);
  }

  /** Deregisters read-write transaction.

  Transaction is removed from rw_trx_hash, which releases all implicit locks.
  MVCC snapshot won't see this transaction anymore. */
  void deregister_rw(trx_t *trx) { rw_trx_hash.erase(trx); }

  bool is_registered(trx_t *caller_trx, trx_id_t id) {
    return (id > 0) && (find(caller_trx, id, false) != nullptr);
  }

  trx_t *find(trx_t *caller_trx, trx_id_t id, bool do_ref_count = true) {
    return rw_trx_hash.find(caller_trx, id, do_ref_count);
  }

  /** Registers transaction in trx_sys.

  @param trx transaction */
  void register_trx(trx_t *trx) {
    mutex_enter(&mutex);
    UT_LIST_ADD_FIRST(trx_list, trx);
    mutex_exit(&mutex);
  }

  /** Deregisters transaction in trx_sys.

  @param trx transaction */
  void deregister_trx(trx_t *trx) {
    mutex_enter(&mutex);
    UT_LIST_REMOVE(trx_list, trx);
    mutex_exit(&mutex);
  }

  /** Clones the oldest view and stores it in view.

  No need to call ReadView::close(). The caller owns the view that is passed in.
  This function is called by purge thread to determine whether it should purge
  the delete marked record or not. */
  void clone_oldest_view();

  /** Clones the oldest view of the cluster. Just call clone_oldest_view() if
  there is no replica. */
  void clone_oldest_view_for_cluster();

  /** @return the number of active views. */
  size_t view_count() const {
    size_t count = 0;

    mutex_enter(&mutex);
    for (const trx_t *trx = UT_LIST_GET_FIRST(trx_list); trx != nullptr;
         trx = UT_LIST_GET_NEXT(trx_list, trx)) {
      if (trx->read_view.get_state() == READ_VIEW_STATE_OPEN) {
        ++count;
      }
    }
    mutex_exit(&mutex);
    return count;
  }

  /** @return true if found prepared transaction(s). */
  bool found_prepared_trx();

 private:
  static bool get_min_trx_id_callback(rw_trx_hash_element_t *element,
                                      trx_id_t *id) {
    if (element->id < *id) {
      mutex_enter(&element->mutex);
      /* We don't care about read-only transactions here. */
      if (element->trx != nullptr &&
          element->trx->rsegs.m_redo.rseg != nullptr) {
        *id = element->id;
      }
      mutex_exit(&element->mutex);
    }
    return false;
  }

  struct snapshot_ids_arg {
    snapshot_ids_arg(ReadView::ids_t *_ids) : ids(_ids) {}

    ReadView::ids_t *ids;
    trx_id_t id;
    trx_id_t no;
  };

  static bool copy_one_id(rw_trx_hash_element_t *element,
                          snapshot_ids_arg *arg) {
    if (element->id < arg->id) {
      trx_id_t no = element->no.load(std::memory_order_relaxed);
      arg->ids->push_back(element->id);
      if (no < arg->no) {
        arg->no = no;
      }
    }
    return false;
  }

  /** Get for rw_trx_hash_version, must issue ACQUIRE memory barrier. */
  trx_id_t get_rw_trx_hash_version() {
    return rw_trx_hash_version.load(std::memory_order_acquire);
  }

  /** Increments rw_trx_hash_version, must issue RELEASE memory barrier. */
  void refresh_rw_trx_hash_version() {
    rw_trx_hash_version.fetch_add(1, std::memory_order_release);
  }

  /** Allocates new transaction id without refreshing rw_trx_hash_version.

  This method is extracted for exclusive use by register_rw() and
  assign_new_trx_no() where new id must be allocated atomically with payload
  of these methods from MVCC snapshot point of view.

  @sa assign_new_trx_no()

  @return new transaction id */
  trx_id_t get_new_trx_id_no_refresh() {
    return max_trx_id.fetch_add(1, std::memory_order_relaxed);
  }
};
```

## 4.4 事务管理器trx_list及mutex

出于实现原因，**trx_sys_t** 仍然保留了一把全局锁（**mutex**），也保留了 **trx_list**（可以简单粗暴的理解成取代了原来的3个列表，即 **serialisation_list**/**rw_trx_list**/**mysql_trx_list**），并且从上一节（4.3）可以看出，只有2个接口（**register_trx**/**deregister_trx**）是用来操作 trx_sys->trx_list 的，理论上来说，这个列表的功能完全可以被 trx_sys->rw_trx_hash 取代，但这个列表存在的积极意义应该主要是为了减少对 trx_sys->rw_trx_hash 的遍历操作，毕竟好钢用在刀刃上，ReadView相关操作需要频繁访问LF_HASH，但其它操作就完全没有必要了，比如遍历事务列表非 ReadView状态项（即**id**/**no**/**state**） 的相关操作）。

访问 trx_sys->mutex

此外，trx_sys->mutex 还用来局部保证 purge_sys 在遍历 trx_sys->trx_list 时能够获取到一致性的快照信息：

```
/** Creates a snapshot where exactly the transaction serialized before this
point in time are seen in the view.

@param[in,out] trx transaction */
void ReadView::snapshot(trx_t *trx) {
  trx_sys->snapshot_ids(trx, &m_ids, &m_low_limit_id, &m_low_limit_no);

  ids_t::value_type *p = m_ids.data();
  std::sort(p, p + m_ids.size());

#ifdef UNIV_DEBUG
  m_up_limit_id = m_ids.empty() ? m_low_limit_id : m_ids.front();
  ut_ad(m_up_limit_id <= m_low_limit_id);
#endif /* UNIV_DEBUG */
}

/** Open a read view where exactly the transactions serialized before this point
in time are seen in the view.

View becomes visible to purge thread.

@param[in,out] trx transaction*/
void ReadView::open(trx_t *trx) {
  ut_ad(this == &trx->read_view);
  switch (m_state.load(std::memory_order_relaxed)) {
    case READ_VIEW_STATE_OPEN:
      ut_ad(!srv_read_only_mode);
      return;
    case READ_VIEW_STATE_CLOSED:
      if (srv_read_only_mode) {
        return;
      }
      /* Reuse closed view if there were no read-write transactions since (and
      at) its creation time.

      Original comment states: there is an inherent race between purge and this
      thread.

      To avoid this race we should've checked trx_sys->get_max_trx_id() and set
      state to READ_VIEW_STATE_OPEN atomically under trx_sys->mutex protection.
      But we'are cutting edges to achieve greate scalability.

      There's at least two types of concurrent threads interested in this value:
      purge coordinator thread (sees trx_sys_t::clone_oldest_view()) and InnoDB
      monitor thread (see lock_trx_print_wait_and_mvcc_state()).

      What bad things can happen because we allow this race?

      Speculative execution may reorder state change before get_max_trx_id(). In
      this case purge thread has short gap to clone outdated view. Which is
      probably not that bad: it just won't be able to purge things that it was
      actually allowed to purge for a short while.

      This thread may as well get suspended after trx_sys->get_max_trx_id() and
      before state is set to READ_VIEW_STATE_OPEN. New read-write transaction
      may get started, committed and purged meanwhile. It is acceptable as well,
      since this view doesn't see it. */
      if (trx_is_autocommit_non_locking(trx) && m_ids.empty() &&
          m_low_limit_id == trx_sys->get_max_trx_id()) {
        goto reopen;
      }

      /* Can't reuse view, take new snapshot.

      Alas this empty critical section is simplest way to make sure concurrent
      purge thread completed snapshot copy. Of course purge thread may come
      again and try to copy once again after we release this mutex, but in this
      case it is guaranteed to see READ_VIEW_STATE_REGISTERED and thus it'll
      skip this view.

      This critical section can be replaced with new state, which purge thread
      would set to inform us to wait until it completes snapshot. However it'd
      complicate m_state even further. */
      mutex_enter(&trx_sys->mutex);
      mutex_exit(&trx_sys->mutex);
      m_state.store(READ_VIEW_STATE_SNAPSHOT, std::memory_order_relaxed);
      break;
    default:
      ut_ad(false);
  }

  snapshot(trx);
reopen:
  m_creator_trx_id = trx->id;
  m_state.store(READ_VIEW_STATE_OPEN, std::memory_order_release);
}

/** Clones the oldest view and stores it in view.

No need to call ReadView::close(). The caller owns the view that is passed in.
This function is called by purge thread to determine whether it should purge the
delete marked record or not. */
void trx_sys_t::clone_oldest_view() {
  purge_sys->view.snapshot(nullptr);
  mutex_enter(&mutex);
  /* Find oldest view. */
  for (const trx_t *trx = UT_LIST_GET_FIRST(trx_list); trx != nullptr;
       trx = UT_LIST_GET_NEXT(trx_list, trx)) {
    int32_t state;
    while ((state = trx->read_view.get_state()) == READ_VIEW_STATE_SNAPSHOT) {
      ut_delay(1);
    }

    if (state == READ_VIEW_STATE_OPEN) {
      purge_sys->view.copy(trx->read_view);
    }
  }
  mutex_exit(&mutex);
}
```

# 5 Lock-Free ReadView调用接口变化

## 5.1 原trx_t->in_mysql_trx_list

这个意思是说判断事务是不是用户事务（有些事务可以是InnoDB的背景线程起的事务，而不是用户起的），新的判断方式为：

```
ut_ad(trx->mysql_thd != nullptr);
```

## 5.2 原trx_rw_is_active/trx_get_rw_trx_by_id

根据具体情况，原来的这2个调用可以被新接口 trx_sys->find 或 trx_sys->is_registered 取代。



## 5.3 原MVCC

原来的整个 MVCC read view manager 不需要了（ReadView列表   及**ReadView** 



1. MVCC::**view_open** -> ReadView::**open**
2. MVCC::**view_close** -> ReadView::**close**
3. MVCC::**is_view_active** -> ReadView::**is_open**
4. MVCC::**clone_oldest_view** -> trx_sys_t::**clone_oldest_view**
5. MVCC::**size** -> trx_sys_t::**view_count**

# 附：参考资料

[1] [Taurus SQL层基于锁优化读写性能的一些方式](http://3ms.huawei.com/hi/group/3288655/wiki_5266709.html)

[3] [MDEV-12288 Reset DB_TRX_ID when the history is removed, to speed up MVCC](https://github.com/MariaDB/server/commit/3c09f148f3)

[4] [MDEV-13536 DB_TRX_ID is not actually being reset when the history is removed](https://jira.mariadb.org/browse/MDEV-13536)

[5] [MDEV-15914 performance regression for mass update](https://jira.mariadb.org/browse/MDEV-15914)

[6] [c3ne型弹性云服务器网络性能评估](http://3ms.huawei.com/hi/group/3288655/wiki_5314655.html)

[7] [MariaDB事务快照性能优化方案分析](http://3ms.huawei.com/hi/group/3288655/wiki_5274503.html)

[8] [MySQL · 源码分析 · InnoDB的read view，回滚段和purge过程简介](http://3ms.huawei.com/mysql.taobao.org/monthly/2018/03/01/)

[9] [MySQL · 特性分析 · 执行计划缓存设计与实现](http://mysql.taobao.org/monthly/2016/09/04/)

[10] [Split-Ordered Lists: Lock-Free Extensible Hash Tables](http://people.csail.mit.edu/shanir/publications/Split-Ordered_Lists.pdf)

[11] [MySQL · 5.7优化 · Metadata Lock子系统的优化](http://mysql.taobao.org/monthly/2014/11/05/)

[12] [The World's Simplest Lock-Free Hash Table](https://preshing.com/20130605/the-worlds-simplest-lock-free-hash-table/)