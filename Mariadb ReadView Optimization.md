# 1 前言

笔者之前和 **卡尔文** **DB_TRX_ID****MariaDB** 10.3











笔者浏览了MariaDB 10.2/10.3/10.4 的代码，发现 trx_sys_t 这个数据结构 10.2 -> 10.3 变化很大，MariaDB 10.2这块的代码还是跟MySQL比较相似，10.3就已经改得面目全非了，**Marko** 邮件里面提到的 lock-free hash table应该指的下面这个数据结构：

```
  /**
    Lock-free hash of in memory read-write transactions.
    Works faster when it is on it's own cache line (tested).
  */

  MY_ALIGNED(CACHE_LINE_SIZE) rw_trx_hash_t rw_trx_hash;
```



笔者在本文里面会分析一下MariaDB这个优化的具体实现。



# 2 rw_trx_hash

## 2.1 初始化/销毁

**trx_sys_create** 改成了 trx_sys_t::**create**：

```
/*****************************************************************//**
Creates the trx_sys instance and initializes purge_queue and mutex. */
void
trx_sys_create(void)
/*================*/
{
	ut_ad(trx_sys == NULL);

	trx_sys = static_cast<trx_sys_t*>(ut_zalloc_nokey(sizeof(*trx_sys)));

	mutex_create(LATCH_ID_TRX_SYS, &trx_sys->mutex);

	UT_LIST_INIT(trx_sys->serialisation_list, &trx_t::no_list);
	UT_LIST_INIT(trx_sys->rw_trx_list, &trx_t::trx_list);
	UT_LIST_INIT(trx_sys->mysql_trx_list, &trx_t::mysql_trx_list);

	trx_sys->mvcc = UT_NEW_NOKEY(MVCC(1024));

	new(&trx_sys->rw_trx_ids) trx_ids_t(ut_allocator<trx_id_t>(
			mem_key_trx_sys_t_rw_trx_ids));

	new(&trx_sys->rw_trx_set) TrxIdSet();
}

/** Create the instance */
void
trx_sys_t::create()
{
	ut_ad(this == &trx_sys);
	ut_ad(!is_initialised());
	m_initialised = true;
	mutex_create(LATCH_ID_TRX_SYS, &mutex);
	UT_LIST_INIT(trx_list, &trx_t::trx_list);
	my_atomic_store32(&rseg_history_len, 0);

	rw_trx_hash.init();
}
```



从实现对比我们可以看出：



1. 10.2里面的 **serialisation_list**/**rw_trx_list**/**mysql_trx_list** 在10.3里面合并变成了 **trx_list**；
2. 10.2里面的 **mvcc**/**rw_trx_ids**/**rw_trx_set** 在10.3里面合并变成了 **rw_trx_hash**；





销毁这方面，**trx_sys_close** 改成了 trx_sys_t::**close**：

```
/*********************************************************************
Shutdown/Close the transaction system. */
void
trx_sys_close(void)
/*===============*/
{
	ut_ad(trx_sys != NULL);
	ut_ad(srv_shutdown_state == SRV_SHUTDOWN_EXIT_THREADS);

	if (ulint size = trx_sys->mvcc->size()) {
		ib::error() << "All read views were not closed before"
			" shutdown: " << size << " read views open";
	}

	/* Only prepared transactions may be left in the system. Free them. */
	ut_a(UT_LIST_GET_LEN(trx_sys->rw_trx_list) == trx_sys->n_prepared_trx
	     || !srv_was_started
	     || srv_read_only_mode
	     || srv_force_recovery >= SRV_FORCE_NO_TRX_UNDO);

	while (trx_t* trx = UT_LIST_GET_FIRST(trx_sys->rw_trx_list)) {
		UT_LIST_REMOVE(trx_sys->rw_trx_list, trx);
		trx_free_prepared(trx);
	}

	/* There can't be any active transactions. */

	for (ulint i = 0; i < TRX_SYS_N_RSEGS; ++i) {
		if (trx_rseg_t* rseg = trx_sys->rseg_array[i]) {
			trx_rseg_mem_free(rseg);
		}

		if (trx_rseg_t* rseg = trx_sys->temp_rsegs[i]) {
			trx_rseg_mem_free(rseg);
		}
	}

	UT_DELETE(trx_sys->mvcc);

	ut_a(UT_LIST_GET_LEN(trx_sys->rw_trx_list) == 0);
	ut_a(UT_LIST_GET_LEN(trx_sys->mysql_trx_list) == 0);
	ut_a(UT_LIST_GET_LEN(trx_sys->serialisation_list) == 0);

	/* We used placement new to create this mutex. Call the destructor. */
	mutex_free(&trx_sys->mutex);

	trx_sys->rw_trx_ids.~trx_ids_t();

	trx_sys->rw_trx_set.~TrxIdSet();

	ut_free(trx_sys);

	trx_sys = NULL;
}

/** Close the transaction system on shutdown */
void
trx_sys_t::close()
{
	ut_ad(srv_shutdown_state == SRV_SHUTDOWN_EXIT_THREADS);
	if (!is_initialised()) {
		return;
	}

	if (size_t size = view_count()) {
		ib::error() << "All read views were not closed before"
			" shutdown: " << size << " read views open";
	}

	rw_trx_hash.destroy();

	/* There can't be any active transactions. */

	for (ulint i = 0; i < TRX_SYS_N_RSEGS; ++i) {
		if (trx_rseg_t* rseg = rseg_array[i]) {
			trx_rseg_mem_free(rseg);
		}

		if (trx_rseg_t* rseg = temp_rsegs[i]) {
			trx_rseg_mem_free(rseg);
		}
	}

	ut_a(UT_LIST_GET_LEN(trx_list) == 0);
	mutex_free(&mutex);
	m_initialised = false;
}
```



## 2.2 Lock-Free Hash Table实现

基础数据结构在 include/lf.h**lf**



```
/*
  extendible hash, lf_hash.c
*/
#include <hash.h>

C_MODE_START

typedef struct st_lf_hash LF_HASH;
typedef void (*lf_hash_initializer)(LF_HASH *hash, void *dst, const void *src);

#define LF_HASH_UNIQUE 1

/* lf_hash overhead per element (that is, sizeof(LF_SLIST) */
extern const int LF_HASH_OVERHEAD;

struct st_lf_hash {
  LF_DYNARRAY array;                    /* hash itself */
  LF_ALLOCATOR alloc;                   /* allocator for elements */
  my_hash_get_key get_key;              /* see HASH */
  lf_hash_initializer initializer;      /* called when an element is inserted */
  my_hash_function hash_function;       /* see HASH */
  CHARSET_INFO *charset;                /* see HASH */
  uint key_offset, key_length;          /* see HASH */
  uint element_size;                    /* size of memcpy'ed area on insert */
  uint flags;                           /* LF_HASH_UNIQUE, etc */
  int32 volatile size;                  /* size of array */
  int32 volatile count;                 /* number of elements in the hash */
};

void lf_hash_init(LF_HASH *hash, uint element_size, uint flags,
                  uint key_offset, uint key_length, my_hash_get_key get_key,
                  CHARSET_INFO *charset);
void lf_hash_destroy(LF_HASH *hash);
int lf_hash_insert(LF_HASH *hash, LF_PINS *pins, const void *data);
void *lf_hash_search(LF_HASH *hash, LF_PINS *pins, const void *key, uint keylen);
void *lf_hash_search_using_hash_value(LF_HASH *hash, LF_PINS *pins,
                                      my_hash_value_type hash_value,
                                      const void *key, uint keylen);
int lf_hash_delete(LF_HASH *hash, LF_PINS *pins, const void *key, uint keylen);
int lf_hash_iterate(LF_HASH *hash, LF_PINS *pins,
                    my_hash_walk_action action, void *argument);
/*
  shortcut macros to access underlying pinbox functions from an LF_HASH
  see lf_pinbox_get_pins() and lf_pinbox_put_pins()
*/
#define lf_hash_get_pins(HASH)       lf_alloc_get_pins(&(HASH)->alloc)
#define lf_hash_put_pins(PINS)       lf_pinbox_put_pins(PINS)
#define lf_hash_search_unpin(PINS)   lf_unpin((PINS), 2)
/*
  cleanup
*/

C_MODE_END
```



然后基于 st_lf_hash 就实现了 rw_trx_hash，也就是Lock-Free Hash Table：

```
/**
  Wrapper around LF_HASH to store set of in memory read-write transactions.
*/
class rw_trx_hash_t
{
  LF_HASH hash;
  ...

public:
  void init()
  {
    lf_hash_init(&hash, sizeof(rw_trx_hash_element_t), LF_HASH_UNIQUE, 0,
                 sizeof(trx_id_t), 0, &my_charset_bin);
    hash.alloc.constructor= rw_trx_hash_constructor;
    hash.alloc.destructor= rw_trx_hash_destructor;
    hash.initializer=
      reinterpret_cast<lf_hash_initializer>(rw_trx_hash_initializer);
  }


  void destroy()
  {
    hash.alloc.destructor= rw_trx_hash_shutdown_destructor;
    lf_hash_destroy(&hash);
  }

  ...

  /**
    Inserts trx to lock-free hash.

    Object becomes accessible via rw_trx_hash.
  */

  void insert(trx_t *trx)
  {
    ut_d(validate_element(trx));
    int res= lf_hash_insert(&hash, get_pins(trx),
                            reinterpret_cast<void*>(trx));
    ut_a(res == 0);
  }


  /**
    Removes trx from lock-free hash.

    Object becomes not accessible via rw_trx_hash. But it still can be pinned
    by concurrent find(), which is supposed to release it immediately after
    it sees object trx is 0.
  */

  void erase(trx_t *trx)
  {
    ut_d(validate_element(trx));
    mutex_enter(&trx->rw_trx_hash_element->mutex);
    trx->rw_trx_hash_element->trx= 0;
    mutex_exit(&trx->rw_trx_hash_element->mutex);
    int res= lf_hash_delete(&hash, get_pins(trx),
                            reinterpret_cast<const void*>(&trx->id),
                            sizeof(trx_id_t));
    ut_a(res == 0);
  }


  /**
    Returns the number of elements in the hash.

    The number is exact only if hash is protected against concurrent
    modifications (e.g. single threaded startup or hash is protected
    by some mutex). Otherwise the number may be used as a hint only,
    because it may change even before this method returns.
  */

  uint32_t size()
  {
    return uint32_t(my_atomic_load32_explicit(&hash.count,
                                              MY_MEMORY_ORDER_RELAXED));
  }


  /**
    Iterates the hash.

    @param caller_trx  used to get/set pins
    @param action      called for every element in hash
    @param argument    opque argument passed to action

    May return the same element multiple times if hash is under contention.
    If caller doesn't like to see the same transaction multiple times, it has
    to call iterate_no_dups() instead.

    May return element with committed transaction. If caller doesn't like to
    see committed transactions, it has to skip those under element mutex:

      mutex_enter(&element->mutex);
      if (trx_t trx= element->trx)
      {
        // trx is protected against commit in this branch
      }
      mutex_exit(&element->mutex);

    May miss concurrently inserted transactions.

    @return
      @retval 0 iteration completed successfully
      @retval 1 iteration was interrupted (action returned 1)
  */

  int iterate(trx_t *caller_trx, my_hash_walk_action action, void *argument)
  {
    LF_PINS *pins= caller_trx ? get_pins(caller_trx) : lf_hash_get_pins(&hash);
    ut_a(pins);
#ifdef UNIV_DEBUG
    debug_iterator_arg debug_arg= { action, argument };
    action= reinterpret_cast<my_hash_walk_action>(debug_iterator);
    argument= &debug_arg;
#endif
    int res= lf_hash_iterate(&hash, pins, action, argument);
    if (!caller_trx)
      lf_hash_put_pins(pins);
    return res;
  }


  int iterate(my_hash_walk_action action, void *argument)
  {
    return iterate(current_trx(), action, argument);
  }


  /**
    Iterates the hash and eliminates duplicate elements.

    @sa iterate()
  */

  int iterate_no_dups(trx_t *caller_trx, my_hash_walk_action action,
                      void *argument)
  {
    eliminate_duplicates_arg arg(size() + 32, action, argument);
    return iterate(caller_trx, reinterpret_cast<my_hash_walk_action>
                   (eliminate_duplicates), &arg);
  }


  int iterate_no_dups(my_hash_walk_action action, void *argument)
  {
    return iterate_no_dups(current_trx(), action, argument);
  }
};
```

## 2.3 ReadView创建

优化前ReadView创建需要持有 trx_sys->**mutex** 

```
/**
Allocate and create a view.
@param view		view owned by this class created for the
			caller. Must be freed by calling view_close()
@param trx		transaction instance of caller */
void
MVCC::view_open(ReadView*& view, trx_t* trx)
{
	ut_ad(!srv_read_only_mode);

	/** If no new RW transaction has been started since the last view
	was created then reuse the the existing view. */
	if (view != NULL) {

		uintptr_t	p = reinterpret_cast<uintptr_t>(view);

		view = reinterpret_cast<ReadView*>(p & ~1);

		ut_ad(view->m_closed);

		/* NOTE: This can be optimised further, for now we only
		resuse the view iff there are no active RW transactions.

		There is an inherent race here between purge and this
		thread. Purge will skip views that are marked as closed.
		Therefore we must set the low limit id after we reset the
		closed status after the check. */

		if (trx_is_autocommit_non_locking(trx) && view->empty()) {

			view->m_closed = false;

			if (view->m_low_limit_id == trx_sys_get_max_trx_id()) {
				return;
			} else {
				view->m_closed = true;
			}
		}

		mutex_enter(&trx_sys->mutex);

		UT_LIST_REMOVE(m_views, view);

	} else {
		mutex_enter(&trx_sys->mutex);

		view = get_view();
	}

	if (view != NULL) {

		view->prepare(trx->id);

		view->complete();

		UT_LIST_ADD_FIRST(m_views, view);

		ut_ad(!view->is_closed());

		ut_ad(validate());
	}

	trx_sys_mutex_exit();
}
```



优化后去掉这把锁了：

```
/**
  Opens a read view where exactly the transactions serialized before this
  point in time are seen in the view.

  View becomes visible to purge thread.

  @param[in,out] trx transaction
*/
void ReadView::open(trx_t *trx)
{
  ut_ad(this == &trx->read_view);
  switch (m_state)
  {
  case READ_VIEW_STATE_OPEN:
    ut_ad(!srv_read_only_mode);
    return;
  case READ_VIEW_STATE_CLOSED:
    if (srv_read_only_mode)
      return;
    /*
      Reuse closed view if there were no read-write transactions since (and at)
      its creation time.

      Original comment states: there is an inherent race here between purge
      and this thread.

      To avoid this race we should've checked trx_sys.get_max_trx_id() and
      set state to READ_VIEW_STATE_OPEN atomically under trx_sys.mutex
      protection. But we're cutting edges to achieve great scalability.

      There're at least two types of concurrent threads interested in this
      value: purge coordinator thread (see trx_sys_t::clone_oldest_view()) and
      InnoDB monitor thread (see lock_trx_print_wait_and_mvcc_state()).

      What bad things can happen because we allow this race?

      Speculative execution may reorder state change before get_max_trx_id().
      In this case purge thread has short gap to clone outdated view. Which is
      probably not that bad: it just won't be able to purge things that it was
      actually allowed to purge for a short while.

      This thread may as well get suspended after trx_sys.get_max_trx_id() and
      before state is set to READ_VIEW_STATE_OPEN. New read-write transaction
      may get started, committed and purged meanwhile. It is acceptable as
      well, since this view doesn't see it.
    */
    if (trx_is_autocommit_non_locking(trx) && m_ids.empty() &&
        m_low_limit_id == trx_sys.get_max_trx_id())
      goto reopen;

    /*
      Can't reuse view, take new snapshot.

      Alas this empty critical section is simplest way to make sure concurrent
      purge thread completed snapshot copy. Of course purge thread may come
      again and try to copy once again after we release this mutex, but in
      this case it is guaranteed to see READ_VIEW_STATE_REGISTERED and thus
      it'll skip this view.

      This critical section can be replaced with new state, which purge thread
      would set to inform us to wait until it completes snapshot. However it'd
      complicate m_state even further.
    */
    mutex_enter(&trx_sys.mutex);
    mutex_exit(&trx_sys.mutex);
    my_atomic_store32_explicit(&m_state, READ_VIEW_STATE_SNAPSHOT,
                               MY_MEMORY_ORDER_RELAXED);
    break;
  default:
    ut_ad(0);
  }

  snapshot(trx);
reopen:
  m_creator_trx_id= trx->id;
  my_atomic_store32_explicit(&m_state, READ_VIEW_STATE_OPEN,
                             MY_MEMORY_ORDER_RELEASE);
}
```

# 附：参考资料

[1] [Taurus SQL层基于锁优化读写性能的一些方式](http://3ms.huawei.com/hi/group/3288655/wiki_5266709.html)

[2] [MDEV-12288 Reset DB_TRX_ID when the history is removed, to speed up MVCC](https://github.com/MariaDB/server/commit/3c09f148f3)

[3] [MDEV-13536: DB_TRX_ID is not actually being reset when the history is removed](https://jira.mariadb.org/browse/MDEV-13536)

[4] [MDEV-15914: performance regression for mass update](https://jira.mariadb.org/browse/MDEV-15914)