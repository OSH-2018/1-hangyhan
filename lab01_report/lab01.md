**姓名：杭逸哲**
**学号：PB16021135**

# 一、实验环境
&emsp;&emsp;Ubuntu 16.04LTS
&emsp;&emsp;QEMU 2.5.0
&emsp;&emsp;gdb 8.1

# 二、准备工作

  1. 在http://www.kernel.org上下载Linux最新内核

  2. 输入`mv linux-4.16.tar.xz /usr/src`，将内核源代码移动至/usr/src

  3. 输入tar -xvf linux-4.16.tar.xz，将内核代码解压

  4. 输入`cp /boot/config-4.13.0-36-generic .config`使用原来的配置文件

  5. 输入`make menuconfig`进行手动配置
    此时出现错误

  >  Unable to find the ncurses libraries or the required header files.
  
  >  /bin/sh: 1: bison: not found


    这是因为没有安装ncurses-dev和bison
    输入 apt-get install ncurses-dev(bison)安装 ncurses libraries和bison。
    安装后重试即可进行手动配置
     
    进行以下两项配置，以保证可以正常设置断点
![1](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/2.PNG)

![2](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/3.PNG)

 6. 输入`make`进行编译
    出现错误 

    >fatal error: openssl/opensslv.h: No such file or directory              

    输入`apt-get install libssl-dev`可解决问题

 7. 输入`apt-get install qemu`安装QEMU

 8. 输入' qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -initrd rootfs.img -append "console=tty1 root=/dev/ram rdinit=/sbin/init nokaslr" -S -s'运行已编译的内核。
    但是在启动内核时，出现错误

    ![](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/4.PNG)

    提示缺少rootfs文件，并且虚拟机停在`Booting from ROM`。因此需要制作rootfs文件

 9. 下载busybox    
    执行以下命令从网上下载git；然后下载busybox源码，配置并安装busybox。在配置时注意选
    择Build static binary (no shared libs)。
    >apt-get install git 
    
    >git clone git://busybox.net/busybox.git
    
    >make menuconfig
    
    >make
    
    >make install

    ![](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/5.PNG)

  10.编辑rcS文件并创建rootfs.img
    利用命令`touch _install/etc/initd.d/rcS`在该路径下创建.sh文件rcS
    输入"`vi rcS.sh`"并按下"Esc+:+i"进入编辑模式
    在编辑模式下输入
    `#!/bin/sh`
    
    `mount -t proc none /proc`
    
    `mount -t sysfs none /sys sbin/mdev -s`
    
    按下"Esc + : + wq"保存并推出
    
    输入`chmod +x rcS`使得rcS.sh为可执行文件
    
    输入`cd _install`切换至_install下，执行`find . | cpio -o --format=newc > ../rootfs.img`
    
    输入`mv rootfs.img /usr/src/ubuntu-4.16`将生成的rootfs.img镜像文件移至内核内
    

  11.测试能否正常调试
    打开一个终端，输入`cd /usr/src/linux-4.16`切换到内核目录下
    输入`qemu-system-x86_64 -kernel arch/x86_64/boot/bzImage -initrd rootfs.img -append "console=tty1 root=/dev/ram rdinit=/sbin/init nokaslr" -S -s`启动QEMU
    
    打开另一个终端，同样输入'cd /usr/src/linux-4.16'切换至内核目录下
    
    输入`gdb -tui`
    
    输入'file vmlinux'加载符号表
    
    输入`target remote:1234`建立远程连接
    
    输入`break start_point`设置断点
    
    输入`c`执行
    
    此时内核能在断点停下，但是会出现异常：

![](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/6.PNG)

上网寻找解决方法后，可知：先中断远程调试，然后输入`set arch i386:x86-64:intel`再重新建立连接，便能正常在断点处中止

![](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/7.PNG)

但是在关闭gdb和QEMU后，再次尝试进行远程调试时，该异常再次出现。说明上述方法只是临时的补救措施。
根据网上的查阅相关资料，需要修改gdb/remote.c文件中的static void。但是并没有在并没有在电脑中找到gdb下的remote.c。所以只能暂时采用上面的办法了。

# 三、调试内核启动过程

  1. 分析start_kernel函数
     打开/usr/src/linux-4.16/init/main.c，查看start_kernel函数的代码。可以发现start_kernel中调用的函数有：

> set_task_stack_end_magic(&init_stack);

> smp_setup_procesor_id();

> debug_objects_early_init();

> cgroup_init_early();

> local_irq_disable();

> boot_cpu_init();

> page_address_init();

> pr_notice("%s", linux_banner);

> setup_arch(&command_line);

> add_latent_entropy();

> add_device_randomness(command_line, strlen(command_line));

> boot_init_stack_canary();

> mm_init_cpumask(&init_mm);

> setup_command_line(command_line);

> setup_nr_cpu_ids();

> setup_per_cpu_areas();

> boot_cpu_state_init();

> smp_prepare_boot_cpu();

> build_all_zonelists(NULL);

> page_alloc_init();

> pr_notice("Kernel command line: %s\n", boot_command_line);

> parse_early_param();

> parse_args("Booting kernel",static_command_line, __start___param, __stop___param - __start___param,-1, -1, NULL, &unknown_bootoption);

> IS_ERR_OR_NULL(after_dashes)

> parse_args("Setting init args", after_dashes, NULL, 0, -1, -1, NULL, set_init_arg);

> jump_label_init();

> setup_log_buf(0);

> vfs_caches_init_early();

> sort_main_extable();

> trap_init();

> mm_init();

> ftrace_init();

> early_trace_init();

> sched_init();

> preempt_disable();

> WARN(!irqs_disabled(),"Interrupts were enabled *very* early, fixing it\n"))

> local_irq_disable();

> radix_tree_init();

> housekeeping_init();

> workqueue_init_early();

> rcu_init();

> trace_init();

> context_tracking_init();

> early_irq_init();

> init_IRQ();

> tick_init();

> rcu_init_nohz();

> init_timers();


> hrtimers_init();

> softirq_init();

> timekeeping_init();

> time_init();

> sched_clock_postinit();

> printk_safe_init();

> perf_event_init();

> profile_init();

> call_function_init();

> WARN(!irqs_disabled(), "Interrupts were enabled early\n");

> early_boot_irqs_disabled = false;

> local_irq_enable();


> kmem_cache_init_late();


> console_init();


> panic("Too many boot %s vars at `%s'", panic_later,panic_param);

> lockdep_info();

> locking_selftest();

> mem_encrypt_init();

> page_to_pfn(virt_to_page((void *)initrd_start)) 


> pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n", page_to_pfn(virt_to_page((void *)initrd_start)),min_low_pfn);

> page_ext_init();

> kmemleak_init();

> debug_objects_mem_init();

> setup_per_cpu_pageset();

> numa_policy_init();

> acpi_early_init();


> late_time_init();

> calibrate_delay();

> pid_idr_init();

> anon_vma_init();

> efi_enabled(EFI_RUNTIME_SERVICES)

> efi_enter_virtual_mode();

> thread_stack_cache_init();

> cred_init();

> fork_init();

> proc_caches_init();

> buffer_init();

> key_init();

> security_init();

> dbg_late_init();

> vfs_caches_init();

> pagecache_init();

> signals_init();

> proc_root_init();

> nsfs_init();

> cpuset_init();


> cgroup_init();

> taskstats_init_early();

> delayacct_init();

> check_bugs();

> acpi_subsystem_init();

> arch_post_acpi_subsys_init();

> sfi_init_late();

> efi_enabled(EFI_RUNTIME_SERVICES)

> efi_free_boot_services();

> rest_init();

> 共调用了一百多个个函数，下面只挑选几个函数进行分析。


2.set_task_stack_end_magic()函数
在gdb中使用单步执行step命令进入set_task_stack_end_magic()函数.
![](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/9.PNG)
该函数的代码为：


> ```c++
> void set_task_stack_end_magic(struct task_struct *tsk)
> {
>    unsigned long *stakend;
>    stakend = end_of_stack(tsk);
>    *stackend = STACK_END_MAGIC ;
> }
> ```
>
> 

在main()函数中调用set_task_stack_end_magic()时，向后者传递了变量inti_task。而init_task是在函数init_task()中完成的初始化，相关代码是：

> INIT_THREAD_INFO(init_task);

函数set_task_stack_end_magic()的主要功能就是获取init_task的栈边界地址，并将宏STACK_END_MAGIC的值设为该值，从而STACK_END_MAGIC之后可用作栈边界的标识以防止溢出。

3.cgroup_init_early()函数

利用break cgoup_init_early()在cgroup_init_early()处设置断点，可在内核运行到该函数时中断。

该函数的代码为：

```c++
int __init cgroup_init_early(void)
{
    init_cgroup_root(&cgrp_dfl_root, &opts);
    cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;

    RCU_INIT_POINTER(init_task.cgroups, &init_css_set);

    for_each_subsys(ss, i) {
        WARN(!ss->css_alloc || !ss->css_free || ss->name || ss->id,
             "invalid cgroup_subsys %d:%s css_alloc=%p css_free=%p id:name=%d:%s\n",
             i, cgroup_subsys_name[i], ss->css_alloc, ss->css_free,
             ss->id, ss->name);
        WARN(strlen(cgroup_subsys_name[i]) > MAX_CGROUP_TYPE_NAMELEN,
             "cgroup_subsys_name %s too long\n", cgroup_subsys_name[i]);

        ss->id = i;
        ss->name = cgroup_subsys_name[i];
        if (!ss->legacy_name)
            ss->legacy_name = cgroup_subsys_name[i];

        if (ss->early_init)
            cgroup_init_subsys(ss, true);
    }
    return 0;
}
```

要了解这个函数的功能首先得先了解cgroup的概念。

 cgroup是LInux内核提供的一种可以限制单个进程或者多个进程所使用资源的机制，开发者可以利用cgroup提供的精细化控制能力限制某一个或者一组进程对资源的使用。

![](https://github.com/OSH-2018/1-hangyhan/blob/master/lab01_report/pictures/8.png)

上图描述了cgroup(control group)、资源/子系统和进程的关系。

每一个cgroup层级结构中是一颗树形结构，树的每个结点就是一个cgroup结构体；每一个cgroup层级结构可以被多个资源/子系统所限制。如图中的cgroup层级A，/cgrp 1和/cgrp2共同对cpu和cpuacct资源进行限制。

在相同的资源上受到限制的进程（图中的P）会关联到同一个辅助数据结构css_set(cgroups subsystem set)，而一个辅助结构可以与不同cgroup层级中的多个cgroup结构体相关联，从而表示该css_set下的进程都将受到多个资源上的限制。

查阅相关资料，与cgroup概念相关的内核代码有：

- task_struct

  task_struct是存储进程信息的结构体，其中与cgroup相关的是

  ```c++
  struct css_set __rcu *cgroups; //cgroups指向该进程所属于的那个css_set
  struct list_head cg_list; // cg_list是所有与该进程同属一个css_set的进程链表表头
  ```

- css_set

  ```c++
  struct css_set {
      atomic_t refcount;  // 记录该css_set所限制的资源数
      struct hlist_node hlist;
      struct list_head tasks; //task是该css_set所关联的进程的链表表头
      struct list_head mg_tasks;
      struct list_head cgrp_links; // cgrp_links是与该css_set相关的cgroup结构体的链表的表头
      struct cgroup *dfl_cgrp;
      struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT]; //css_set的状态数组
      struct list_head mg_preload_node;
      struct list_head mg_node;
      struct cgroup *mg_src_cgrp;
      struct cgroup *mg_dst_cgrp;
      struct css_set *mg_dst_cset;
      struct list_head e_cset_node[CGROUP_SUBSYS_COUNT];
      struct list_head task_iters;
      bool dead;
      struct rcu_head rcu_head;
  };
  ```

- cgroup

  ```c++
  struct cgroup {
      struct cgroup_subsys_state self; //self与cgroup相关的css_set
      unsigned long flags;
      int id;
      // 这个cgroup所在层级中，当前cgroup的深度
      int level; //该cgroup在该cgroup层级中的深度
      int populated_cnt;
      struct kernfs_node *kn;        /* cgroup kernfs entry */
      struct cgroup_file procs_file;    /* handle for "cgroup.procs" */
      struct cgroup_file events_file;    /* handle for "cgroup.events" */

      u16 subtree_control;
      u16 subtree_ss_mask;
      u16 old_subtree_control;
      u16 old_subtree_ss_mask;

      struct cgroup_subsys_state __rcu *subsys[CGROUP_SUBSYS_COUNT]; //记录与该cgroup相关联的css_set

      struct cgroup_root *root; //根croup
      struct list_head cset_links;
      struct list_head e_csets[CGROUP_SUBSYS_COUNT];
      struct list_head pidlists;
      struct mutex pidlist_mutex;
      wait_queue_head_t offline_waitq;
      struct work_struct release_agent_work;
      int ancestor_ids[]; //当前group的祖先
  };
  ```

下面回到cgroup_init_early，可以看出该函数的主要功能就是初始化一个cgroup层级结构的根节点cgroup_root。

4.之后内核会继续执行start_kernel中的其他函数从而完成内核的初始化

# 四、实验总结

在本次实验中，掌握了Qemu,gdb的基本使用方法。熟悉了Linux命令行的常用命令。阅读了linux内核启动的部分关键代码，对Linux启动时做的一些初始化操作有了初步的了解。

本次实验过程中，在搭建实验环境、编译内核的实验准备部分遇到的比较大的麻烦，途中尝试了多种方法都没有进展。最后通过在网上查询一些博客了解到了相似问题的处理方法，解决了问题；这段过程让我以后再遇到这样棘手的问题时不会再感到无所适从。



​      



