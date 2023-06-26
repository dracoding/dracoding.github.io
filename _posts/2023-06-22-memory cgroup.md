---
layout: post
title: 【cgroup】一、memory cgroup
date: 2023-06-19
tags:  cgroup
---

## mem cgroup相关的结构体

<div align="center">
<img src="/images/out/cgroup/memory.png">  
</div> 


## memory资源限制值的设置

```
static ssize_t mem_cgroup_write(struct kernfs_open_file *of,
                                char *buf, size_t nbytes, loff_t off)
{
        struct mem_cgroup *memcg = mem_cgroup_from_css(of_css(of));  // 获取对应的mem_cgroup结构体
        unsigned long nr_pages;
        int ret;

        buf = strstrip(buf);
        ret = page_counter_memparse(buf, "-1", &nr_pages);
        if (ret)
                return ret;

        switch (MEMFILE_ATTR(of_cft(of)->private)) {
        case RES_LIMIT:
                if (mem_cgroup_is_root(memcg)) { /* Can't set limit on root */
                        ret = -EINVAL;
                        break;
                }
                switch (MEMFILE_TYPE(of_cft(of)->private)) {
                case _MEM:
                        ret = mem_cgroup_resize_max(memcg, nr_pages, false);
                        break;
                case _MEMSWAP:
                        ret = mem_cgroup_resize_max(memcg, nr_pages, true);
                        break;
                case _KMEM:
                        pr_warn_once("kmem.limit_in_bytes is deprecated and will be removed. "
                                     "Please report your usecase to linux-mm@kvack.org if you "
                                     "depend on this functionality.\n");
                        ret = memcg_update_kmem_max(memcg, nr_pages);
                        break;
                case _TCP:
                        ret = memcg_update_tcp_max(memcg, nr_pages);
                        break;
                }
                break;
        case RES_SOFT_LIMIT:
                memcg->soft_limit = nr_pages;
                ret = 0;
                break;
        }
        return ret ?: nbytes;
}
```

```
static int mem_cgroup_resize_max(struct mem_cgroup *memcg,
                                 unsigned long max, bool memsw)
{
        bool enlarge = false;
        bool drained = false;
        int ret;
        bool limits_invariant;
        struct page_counter *counter = memsw ? &memcg->memsw : &memcg->memory;

        do {
                if (signal_pending(current)) {
                        ret = -EINTR;
                        break;
                }

                mutex_lock(&memcg_max_mutex);
                /*
                 * Make sure that the new limit (memsw or memory limit) doesn't
                 * break our basic invariant rule memory.max <= memsw.max.
                 */
                limits_invariant = memsw ? max >= READ_ONCE(memcg->memory.max) :
                                           max <= memcg->memsw.max;
                if (!limits_invariant) {
                        mutex_unlock(&memcg_max_mutex);
                        ret = -EINVAL;
                        break;
                }
                if (max > counter->max)
                        enlarge = true;
                ret = page_counter_set_max(counter, max);      // 设置计数器max值
                mutex_unlock(&memcg_max_mutex);

                if (!ret)
                        break;

                if (!drained) {
                        drain_all_stock(memcg);
                        drained = true;
                        continue;
                }

                if (!try_to_free_mem_cgroup_pages(memcg, 1,
                                        GFP_KERNEL, !memsw)) {
                        ret = -EBUSY;
                        break;
                }
        } while (true);

        if (!ret && enlarge)
                memcg_oom_recover(memcg);

        return ret;
}
```

## 缺页处理函数do_cow_fault

```
static vm_fault_t do_cow_fault(struct vm_fault *vmf)
{
        struct vm_area_struct *vma = vmf->vma;
        vm_fault_t ret;

        if (unlikely(anon_vma_prepare(vma)))
                return VM_FAULT_OOM;

        vmf->cow_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vma, vmf->address);
        if (!vmf->cow_page)
                return VM_FAULT_OOM;

        if (mem_cgroup_charge(vmf->cow_page, vma->vm_mm, GFP_KERNEL)) {
                put_page(vmf->cow_page);
                return VM_FAULT_OOM;
        }
        cgroup_throttle_swaprate(vmf->cow_page, GFP_KERNEL);

        ret = __do_fault(vmf);
        if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
                goto uncharge_out;
        if (ret & VM_FAULT_DONE_COW)
                return ret;

        copy_user_highpage(vmf->cow_page, vmf->page, vmf->address, vma);
        __SetPageUptodate(vmf->cow_page);

        ret |= finish_fault(vmf);
        unlock_page(vmf->page);
        put_page(vmf->page);
        if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
                goto uncharge_out;
        return ret;
uncharge_out:
        put_page(vmf->cow_page);
        return ret;
}       
```


```
int mem_cgroup_charge(struct page *page, struct mm_struct *mm, gfp_t gfp_mask)
{
        unsigned int nr_pages = thp_nr_pages(page);
        struct mem_cgroup *memcg = NULL;
        int ret = 0;

        if (mem_cgroup_disabled())
                goto out;

        if (PageSwapCache(page)) {
                swp_entry_t ent = { .val = page_private(page), };
                unsigned short id;

                /* 
                 * Every swap fault against a single page tries to charge the
                 * page, bail as early as possible.  shmem_unuse() encounters
                 * already charged pages, too.  page and memcg binding is
                 * protected by the page lock, which serializes swap cache
                 * removal, which in turn serializes uncharging.
                 */
                VM_BUG_ON_PAGE(!PageLocked(page), page);
                if (page_memcg(compound_head(page)))
                        goto out;

                id = lookup_swap_cgroup_id(ent);
                rcu_read_lock();
                memcg = mem_cgroup_from_id(id); 
                if (memcg && !css_tryget_online(&memcg->css))
                        memcg = NULL;
                rcu_read_unlock();
        }
     
        if (!memcg)
                memcg = get_mem_cgroup_from_mm(mm);    // 获取对应的mem_cgroup结构体
   
        ret = try_charge(memcg, gfp_mask, nr_pages);
        if (ret)
                goto out_put;

        css_get(&memcg->css);
        commit_charge(page, memcg);            // page页设置对应的mem_cgroup地址
                
        local_irq_disable();
        mem_cgroup_charge_statistics(memcg, page, nr_pages);
        memcg_check_events(memcg, page);
        local_irq_enable();
        if (do_memsw_account() && PageSwapCache(page)) {
                swp_entry_t entry = { .val = page_private(page) };
                /*
                 * The swap entry might not get freed for a long time,
                 * let's not wait for it.  The page already received a
                 * memory+swap charge, drop the swap entry duplicate.
                 */
                mem_cgroup_uncharge_swap(entry, nr_pages);
        }

out_put:
        css_put(&memcg->css);
out:
        return ret;
}
```

try_charge

```
static int try_charge(struct mem_cgroup *memcg, gfp_t gfp_mask,
                      unsigned int nr_pages)
{
        unsigned int batch = max(MEMCG_CHARGE_BATCH, nr_pages);
        int nr_retries = MAX_RECLAIM_RETRIES;
        struct mem_cgroup *mem_over_limit;
        struct page_counter *counter;
        enum oom_status oom_status;
        unsigned long nr_reclaimed;
        bool passed_oom = false;
        bool may_swap = true;
        bool drained = false;
        unsigned long pflags;

        if (mem_cgroup_is_root(memcg))
                return 0;
retry:
        if (consume_stock(memcg, nr_pages))
                return 0;

        if (!do_memsw_account() ||
            page_counter_try_charge(&memcg->memsw, batch, &counter)) {
                if (page_counter_try_charge(&memcg->memory, batch, &counter))
                        goto done_restock;
                if (do_memsw_account())
                        page_counter_uncharge(&memcg->memsw, batch);
                mem_over_limit = mem_cgroup_from_counter(counter, memory);
        } else {
                mem_over_limit = mem_cgroup_from_counter(counter, memsw);
                may_swap = false;
        }

        if (batch > nr_pages) {
                batch = nr_pages;
                goto retry;
        }

        /*              
         * Memcg doesn't have a dedicated reserve for atomic
         * allocations. But like the global atomic pool, we need to
         * put the burden of reclaim on regular allocation requests
         * and let these go through as privileged allocations.
         */
        if (gfp_mask & __GFP_ATOMIC)
                goto force;
        if (unlikely(current->flags & PF_MEMALLOC))
                goto force;

        if (unlikely(task_in_memcg_oom(current)))
                goto nomem;

        if (!gfpflags_allow_blocking(gfp_mask))
                goto nomem;

        memcg_memory_event(mem_over_limit, MEMCG_MAX);    // 产生对应的cgroup时间

        psi_memstall_enter(&pflags);
        nr_reclaimed = try_to_free_mem_cgroup_pages(mem_over_limit, nr_pages,    // 执行页面的回收
                                                    gfp_mask, may_swap);
        psi_memstall_leave(&pflags);

        if (mem_cgroup_margin(mem_over_limit) >= nr_pages)    // 剩余可申请的量大于nr_pages
                goto retry;

        if (!drained) {
                drain_all_stock(mem_over_limit);
                drained = true;
                goto retry;
        }

        if (gfp_mask & __GFP_NORETRY)
                goto nomem;
        /*
         * Even though the limit is exceeded at this point, reclaim
         * may have been able to free some pages.  Retry the charge
         * before killing the task.
         *
         * Only for regular pages, though: huge pages are rather
         * unlikely to succeed so close to the limit, and we fall back
         * to regular pages anyway in case of failure.
         */
        if (nr_reclaimed && nr_pages <= (1 << PAGE_ALLOC_COSTLY_ORDER))
                goto retry;
        /*
         * At task move, charge accounts can be doubly counted. So, it's
         * better to wait until the end of task_move if something is going on.
         */
        if (mem_cgroup_wait_acct_move(mem_over_limit))
                goto retry;

        if (nr_retries--)
                goto retry;

        if (gfp_mask & __GFP_RETRY_MAYFAIL)
                goto nomem;
        /* Avoid endless loop for tasks bypassed by the oom killer */
        if (passed_oom && task_is_dying())
                goto nomem;

        /*
         * keep retrying as long as the memcg oom killer is able to make
         * a forward progress or bypass the charge if the oom killer
         * couldn't make any progress.
         */
        oom_status = mem_cgroup_oom(mem_over_limit, gfp_mask,
                       get_order(nr_pages * PAGE_SIZE));
        if (oom_status == OOM_SUCCESS) {
                passed_oom = true;
                nr_retries = MAX_RECLAIM_RETRIES;
                goto retry;
        }
nomem:
        if (!(gfp_mask & __GFP_NOFAIL))
                return -ENOMEM;
force:
        /*
         * The allocation either can't fail or will lead to more memory
         * being freed very soon.  Allow memory usage go over the limit
         * temporarily by force charging it.
         */
        page_counter_charge(&memcg->memory, nr_pages);
        if (do_memsw_account())
                page_counter_charge(&memcg->memsw, nr_pages);

        return 0;

done_restock:
        if (batch > nr_pages)
                refill_stock(memcg, batch - nr_pages);

        /*
         * If the hierarchy is above the normal consumption range, schedule
         * reclaim on returning to userland.  We can perform reclaim here
         * if __GFP_RECLAIM but let's always punt for simplicity and so that
         * GFP_KERNEL can consistently be used during reclaim.  @memcg is
         * not recorded as it most likely matches current's and won't
         * change in the meantime.  As high limit is checked again before
         * reclaim, the cost of mismatch is negligible.
         */
        do {
                bool mem_high, swap_high;

                mem_high = page_counter_read(&memcg->memory) >
                        READ_ONCE(memcg->memory.high);
                swap_high = page_counter_read(&memcg->swap) >
                        READ_ONCE(memcg->swap.high);

                /* Don't bother a random interrupted task */
                if (in_interrupt()) {
                        if (mem_high) {
                                schedule_work(&memcg->high_work);
                                break;
                        }
                        continue;
                }

                if (is_high_async_reclaim(memcg)) {
                        WRITE_ONCE(memcg->high_async_reclaim, true);
                        schedule_work(&memcg->high_work);
                }

                if (mem_high || swap_high) {
                        /*
                         * The allocating tasks in this cgroup will need to do
                         * reclaim or be throttled to prevent further growth
                         * of the memory or swap footprints.
                         *
                         * Target some best-effort fairness between the tasks,
                         * and distribute reclaim work and delay penalties
                         * based on how much each task is actually allocating.
                         */
                        current->memcg_nr_pages_over_high += batch;
                        set_notify_resume(current);
                        break;
                }
        } while ((memcg = parent_mem_cgroup(memcg)));

        return 0;
}
```


<br>
转载请注明： [【cgroup】一、memory cgroup](https://dracoding.github.io/2023/06/memory cgroup/) 
