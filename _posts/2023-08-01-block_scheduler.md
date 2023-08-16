---
layout: post
title: 【vfs】二、bio调用栈
date: 2023-07-18
tags:  vfs
---

## open系统调用

<div align="center">
<img src="/images/out/storage/open_syscall.png">  
</div> 

#### allow_merge

.bio_merge()
    | -> blk_mq_sched_try_merge
    | -> blk_attempt_bio_merge
    |    | -> blk_mq_sched_allow_merge
    |            | -> e->type->ops.allow_merge(q, rq, bio);


elv_iosched_allow_bio_merge
    | -> e->type->ops.allow_merge(q, rq, bio);

#### bio_merge

__submit_bio_noacct_mq
__submit_bio
|    | -> blk_mq_submit_bio
|        | -> blk_mq_sched_bio_merge
|                | -> __blk_mq_sched_bio_merge
|                    | -> e->type->ops.bio_merge(q, bio, nr_segs)

#### request_merge

.bio_merge()
    | ->blk_mq_sched_try_merge
        | -> e->type->ops.request_merge(q, req, bio);

#### request_merged

.bio_merge()
    | -> blk_mq_sched_try_merge
        | -> elv_merged_request
                | -> e->type->ops.request_merged(q, rq, type);

#### requests_merged

.insert_requests()
    | -> blk_mq_sched_try_insert_merge
        | -> elv_attempt_insert_merge
                | -> blk_attempt_req_merge
                    | -> attempt_merge
                        | -> elv_merge_requests
                            | -> e->type->ops.requests_merged(q, rq, type);
        
.bio_merge()
    -> blk_mq_sched_try_merge
        | -> attempt_front_merge
        | -> attempt_back_merge
        |    | -> attempt_merge
        |        | -> elv_merge_requests
        |            | -> e->type->ops.requests_merged(q, rq, type);

#### limit_depth

blk_get_request
     | -> blk_mq_alloc_request
__submit_bio
__submit_bio_noacct_mq
|    | -> blk_mq_submit_bio
|    |    | -> __blk_mq_alloc_request
|    |            | -> e->type->ops.limit_depth(data->cmd_flags, data);


#### prepare_request

         | -> nvme_alloc_request
             | -> blk_mq_alloc_request_hctx
__submit_bio
    | -> blk_mq_submit_bio
blk_get_request
    | -> blk_mq_alloc_request
    |    | -> __blk_mq_alloc_request
    |        |    | -> blk_mq_rq_ctx_init
    |        |        | -> e->type->ops.prepare_request(rq);


#### finish_request

blk_mq_free_request
    | -> e->type->ops.finish_request(rq);

#### insert_requests
    | -> blk_poll
__submit_bio
    | -> blk_mq_submit_bio
    | -> blk_finish_plug
        |    | -> blk_flush_plug_list
        |        | -> blk_mq_flush_plug_list
        |                | -> blk_mq_sched_insert_requests
        |                        | -> e->type->ops.insert_requests(hctx, &list, at_head);


blk_execute_rq_nowait
blk_mq_requeue_work
__blk_mq_try_issue_directly
blk_mq_submit_bio
    | -> blk_mq_sched_insert_request
        | -> e->type->ops.insert_requests(hctx, &list, at_head);

#### dispatch_request

blk_mq_alloc_and_init_hctx
    | -> blk_mq_alloc_hctx
        | -> blk_mq_run_work_fn
blk_mq_delay_run_hw_queues
blk_mq_dispatch_rq_list
blk_mq_do_dispatch_sched
    | -> blk_mq_delay_run_hw_queue
kyber_domain_wake
blk_mq_hctx_notify_dead
blk_mq_submit_bio
blk_mq_request_bypass_insert
blk_mq_start_stopped_hw_queue
blk_mq_start_hw_queue
blk_mq_run_hw_queues
blk_mq_dispatch_rq_list
blk_mq_dispatch_wake
blk_mq_get_tag
blk_mq_sched_insert_requests
blk_mq_sched_insert_request
blk_mq_sched_dispatch_requests
blk_mq_sched_restart
    | -> blk_mq_run_hw_queue
        | -> __blk_mq_delay_run_hw_queue
        |    | -> __blk_mq_run_hw_queue
        |            | -> blk_mq_sched_dispatch_requests
        |                    | -> __blk_mq_sched_dispatch_requests
        |                            | -> blk_mq_do_dispatch_sched
        |                                    | -> __blk_mq_do_dispatch_sched
        |                                            | -> e->type->ops.dispatch_request(hctx);

#### has_work

blk_mq_run_hw_queue
    | -> blk_mq_hctx_has_pending
        | -> blk_mq_sched_has_work
        | -> __blk_mq_do_dispatch_sched
        |    | -> e->type->ops.has_work(hctx)

#### completed_request

blk_mq_dispatch_rq_list
    | -> blk_mq_end_request
        | -> __blk_mq_end_request
                | -> blk_mq_sched_completed_request
                        | -> e->type->ops.completed_request(rq, now);

#### requeue_request

blk_mq_requeue_request
    | -> blk_mq_sched_requeue_request
        | -> e->type->ops.requeue_request(rq);

#### former_request

attempt_front_merge
    | -> elv_former_request
        | -> e->type->ops.former_request(q, rq)


#### former_request

attempt_back_merge
    | -> elv_latter_request
        | -> e->type->ops.next_request(q, rq)

#### init_icq

blk_mq_rq_ctx_init
    | -> blk_mq_sched_assign_ioc
        | -> ioc_create_icq
                | -> et->ops.init_icq(icq);

blk_mq_do_dispatch_ctx
    | -> copy_process
    | -> do_exit
        |    | -> exit_io_context
        |        | -> put_io_context_active
elevator_switch_mq
blk_exit_queue
|    | -> ioc_clear_queue
|       | __ioc_clear_queue
    create_task_io_context
        | -> ioc_release_fn
        |        | -> ioc_destroy_icq
        |        |    | -> ioc_exit_icq
        |        |            | -> et->ops.exit_icq(icq);

#### blk_mq_dispatch_rq_list


__blk_mq_sched_dispatch_requests
    | -> blk_mq_do_dispatch_ctx
        | -> blk_mq_dispatch_rq_list


__blk_mq_do_dispatch_sched
__blk_mq_sched_dispatch_requests

blk_mq_run_work_fn
    | -> __blk_mq_run_hw_queue
        | -> blk_mq_sched_dispatch_requests
            | -> __blk_mq_sched_dispatch_requests
                | -> blk_mq_do_dispatch_sched
                    | -> __blk_mq_do_dispatch_sched
                        | -> blk_mq_dispatch_hctx_list
                            | -> blk_mq_dispatch_rq_list


blk_mq_realloc_hw_ctxs
    | -> blk_mq_alloc_and_init_hctx
        | -> blk_mq_alloc_hctx
            | -> blk_mq_run_work_fn

blk_mq_init_queue
    | -> blk_mq_init_queue_data
        | -> blk_mq_init_allocated_queue
            | -> blk_mq_realloc_hw_ctxs

__blk_mq_update_nr_hw_queues
    | -> blk_mq_realloc_hw_ctxs

<br>
转载请注明： [【vfs】二、block_scheduler](https://dracoding.github.io/2023/08/block_scheduler/) 
