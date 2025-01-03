---
layout:     post
title:      Stress-ng IO
subtitle:   IO压测详解
date:       2025-01-03
author:     Friden.zhang
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Stress ng
---

## 前言

stress-ng 是一个 Linux 系统的压力测试工具，可以用来模拟各种系统负载，包括内存、CPU、磁盘、网络等。stress-ng 主要用于测试系统的性能、稳定性、容错能力、并发处理能力等。

本文将介绍 stress-ng 的 IO 压测方法。

## 正文

### 使用方法
stress-ng 的 IO 压测方法如下：

``` bash
stress-ng --iomix-bytes 1024 --timeout 60s
```
- --iomix-bytes 1024 表示每个 IO worker 线程写 1024 字节。
- --timeout 60s 表示运行 60 秒后自动停止。

### 代码分析

``` c
static const stress_opt_t opts[] = {
	{OPT_iomix_bytes, "iomix-bytes", TYPE_ID_OFF_T, MIN_IOMIX_BYTES, MAX_IOMIX_BYTES, NULL},
	END_OPT,
};
``` 
- iomix-bytes 参数，表示每个 IO worker 线程写 N 字节。


### IO压测函数

``` c

static stress_iomix_func iomix_funcs[] = {
	stress_iomix_wr_seq_bursts,
	stress_iomix_wr_rnd_bursts,
	stress_iomix_wr_seq_slow,
	stress_iomix_wr_seq_slow,
	stress_iomix_rd_seq_bursts,
	stress_iomix_rd_rnd_bursts,
	stress_iomix_rd_seq_slow,
	stress_iomix_rd_seq_slow,
	stress_iomix_sync,
#if defined(HAVE_POSIX_FADVISE)
	stress_iomix_bad_advise,
#endif
	stress_iomix_rd_wr_mmap,
	stress_iomix_wr_bytes,
	stress_iomix_wr_rev_bytes,
	stress_iomix_rd_bytes,
#if defined(__linux__)
	stress_iomix_inode_flags,
#endif
#if defined(__linux__)
	stress_iomix_drop_caches,
#endif
#if defined(HAVE_COPY_FILE_RANGE)
	stress_iomix_copy_file_range,
#endif
#if defined(HAVE_SYS_SENDFILE_H) && \
	defined(HAVE_SENDFILE)
	stress_iomix_sendfile,
#endif
#if defined(__linux__) && \
	defined(__NR_cachestat)
	stress_iomix_cachestat,
#endif
};
```

stress-ng 的 IO 压测函数主要分为以下几类：

- 写序列、随机、慢速：stress_iomix_wr_seq_bursts、stress_iomix_wr_rnd_bursts、stress_iomix_wr_seq_slow。
- 读序列、随机、慢速：stress_iomix_rd_seq_bursts、stress_iomix_rd_rnd_bursts、stress_iomix_rd_seq_slow。
- 同步：stress_iomix_sync。
- 坏的预读：stress_iomix_bad_advise。
- 内存映射：stress_iomix_rd_wr_mmap。
- 写字节、反向写字节：stress_iomix_wr_bytes、stress_iomix_wr_rev_bytes。
- 读字节：stress_iomix_rd_bytes。
- inode 标志：stress_iomix_inode_flags。
- 缓存：stress_iomix_drop_caches。
- 复制文件范围：stress_iomix_copy_file_range。
- 发送文件：stress_iomix_sendfile。
- 缓存统计：stress_iomix_cachestat。

### IO 压测流程

``` c
for (i = 0; i < MAX_IOMIX_PROCS; i++)
	{
		stress_sync_start_init(&s_pids[i]);

		s_pids[i].pid = fork();
		if (s_pids[i].pid < 0)
		{
			goto reap;
		}
		else if (s_pids[i].pid == 0)
		{
			s_pids[i].pid = getpid();
			stress_sync_start_wait_s_pid(&s_pids[i]);

			/* Child */
			(void)sched_settings_apply(true);
			iomix_funcs[i](args, fd, fs_type, iomix_bytes);
			_exit(EXIT_SUCCESS);
		}
		else
		{
			stress_sync_start_s_pid_list_add(&s_pids_head, &s_pids[i]);
		}
	}
```

- stress-ng 遍历 iomix_funcs 数组，为每个压测函数启动一个子进程。
- 每个子进程调用 iomix_funcs 函数，进行 IO 压测。
- 父进程等待子进程结束，并重新启动一个子进程。
- 重复上述步骤，直到所有子进程结束。

### 总结

stress-ng 的 IO 压测方法，通过启动多个 IO worker 线程，模拟多个 IO 并发操作，来模拟真实的 IO 压力。


## 参考

- [stress-ng](https://github.com/ColinIanKing/stress-ng)