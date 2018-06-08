# Hyperkernel: Push-Button Verification of an OS Kernel

## 简介

本文描述了一种设计、实现和正式验证操作系统内核功能正确性的方法（Hyperkernel），它具有高度的验证自动化和低验证负担。

Hyperkernel的界面设计在xv6上，一个类unix操作系统。

Hyperkernel引入了三个关键思想来实现证明自动化：

1. 它限定了内核接口以避免无限循环或递归
2. 它分离内核和用户地址空间以简化关于虚拟内存的推理
3. 它在LLVM中间表示级别执行验证以避免建模的复杂C语义

Hyperkernel使用Z3 SMT解算器来验证Hyperkernel的实施。实验过程中共检查了50个系统调用和相关陷阱处理，其可以避免类似在xv6中发现的错误，并且这种检测的消耗不大

## 主要内容

##### 主要贡献

- 构建了经过验证的操作系统内核的按钮式方法
- 构建了适用于SMT解决方案的内核接口设计

##### 主要研究挑战

- 界面设计，它需要在可用性和校验自动化之间取得平衡。一方面，内核维护着一套丰富的数据结构和不变式来管理进程，虚拟内存和设备等资源。另一方面，系统提供的接口必须以一种能够用SMT解算器完全自动验证这些属性的方式来实现

- 内核代码中的虚拟内存管理。两个虚拟地址可能映射到同一个物理地址。因此，关于内核数据结构的推理需要关于虚拟到物理映射的推理。这种推理任务由于内核代码可以在执行过程中改变虚拟物理映射的事实而变得更加复杂。如下图所示：

  ![1](pic/1.png)

- Hyperkernel也是一个以C语言编写的系统，而C语言是一种已知形式推理复杂化的编程语言。

##### Hyperkernel的解决方案

- 内核接口被设计为有限的：系统调用、异常和中断的所有处理程序都没有无线循环和递归。从而保证可以对他们编程和验证的时候使用SMT

- Hyperkernel使用用户空间的独立空间运行，使用内核的身份映射，这样简化了内核代码的推理。
  - 为了有效地实现这种分离，Hyperkernel使用Intel VTx和AMD-V提供的x86虚拟化支持：内核和用户进程分别使用单独的页表在根（主机）和非根（guest）模式下运行

- Hyperkernel在LLVM中间表示（IR）级别执行验证，语义比C更简单，同时保持足够高的级别以避免关于机器细节的推理。

  架构图如下：

  ![2](pic/2.png)

##### 其他

- 性能方面使用Z3 SMT解算器，在8核机器上约15分钟内完成验证

- 整个设计过程中，作者主要依赖两种规范来描述系统调用的正确性
  - 功能正确性：详细的状态及规范
  - 手工检查：高层次的声明式规范
- 选择LLVM的原因
  - 相比于C代码，LLVM的语义简单，而且未定义的行为更少。
  - 与汇编相比，LLVM保留高层的信息，比如类型，但是同时又不涉及过多的细节。



## 实验

1. 获得对应的docker

  ```bash
  docker pull mancanfly53373931/docker_hv6:hv6
  ```

2. 下载hv6

   ```
   git clone https://github.com/chyyuu/hv6.git
   ```

3. 安装qemu和llvm

   ```bash
   sudo apt install qemu
   sudo apt install llvm
   ```

4. 编译安装

   ```
   make -j
   ```

5. 基本功能测试

   ```bash
   $ sha1sum /etc/passwd
   f1daac1b0756095703a816a62d1b94daa118a86b  /etc/passwd
   $ wttr  
   * Trying wttr.in (5.9.243.177)
   GET /?0 HTTP/1.0
   Host: wttr.in
   User-Agent: curl
   
   HTTP/1.1 200 OK
   Server: nginx/1.9.14
   Date: Mon, 04 Jun 2018 11:39:56 GMT
   Content-Type: text/html; charset=utf-8
   Content-Length: 354
   Connection: close
   Strict-Transport-Security: max-age=63072000; includeSubdomains
   X-Frame-Options: DENY
   X-Content-Type-Options: nosniff
   
   Weather report: Guangzhou, China
   
         .-.      Light Rain Shower
        (   ).    28-33 °C       
       (___(__)   ↓ 11 km/h      
        ‘ ‘ ‘ ‘   8 km           
       ‘ ‘ ‘ ‘    0.0 mm 
   ```

   

6. hv6-verify测试

   ```bash
   root@0ca0071ca44d:~/hv6# make hv6-verify -- -v --failfast HV6.test_sys_set_runnable
        PY2      hv6-verify
   Using z3 v4.5.0.0
   test_sys_set_runnable (__main__.HV6) ... If(Not(ULE(63, 18446744073709551615 + pid.3)),
      If(@proc_table->struct.proc::ppid.0(0, pid.3) ==
         @current.0(0),
         If(@proc_table->struct.proc::state.0(0, pid.3) == 1,
            0,
            4294967274),
         4294967283),
      4294967293)
   procs_state.1(current.0)
   ok
   
   ----------------------------------------------------------------------
   Ran 1 test in 10.566s
   
   OK
   ```
   整体测试

   ```bash
   root@0ca0071ca44d:~/hv6#  make hv6-verify
        PY2      hv6-verify
   Using z3 v4.5.0.0
   If(If(ULE(63, 18446744073709551615 + pid.3),
         4294967293,
         If(@proc_table->struct.proc::state.0(0, pid.3) == 4,
            If(ULE(8192, inpn.0),
               4294967274,
               If(@page_desc_table->struct.page_desc::pid.0(0,
                                           inpn.0) ==
                  @current.0(0),
                  If(And(Extract(63, 13, size.0) == 0,
                         ULE(Extract(12, 0, size.0), 4096)),
                     If(ULE(8192, outpn.0),
                        4294967274,
                        If(@page_desc_table->struct.page_desc::pid.0(0,
                                           outpn.0) ==
                           @current.0(0),
                           If(@page_desc_table->struct.page_desc::type.0(0,
                                           outpn.0) ==
                              3,
                              If(ULE(16, outfd.0),
                                 If(Or(@proc_table->struct.proc::ipc_from.0(0,
                                           pid.3) ==
                                       0,
                                       @proc_table->struct.proc::ipc_from.0(0,
                                           pid.3) ==
                                       @current.0(0)),
                                    0,
                                    4294967283),
                                 If(ULE(127,
                                        18446744073709551615 +
                                        @proc_table.0(0,
                                           @current.0(0),
                                           SignExt(32, outfd.0))),
                                    If(Or(@proc_table->struct.proc::ipc_from.0(0,
                                           pid.3) ==
                                          0,
                                          @proc_table->struct.proc::ipc_from.0(0,
                                           pid.3) ==
                                          @current.0(0)),
                                       0,
                                       4294967283),
                                    4294967274)),
                              4294967274),
                           4294967283)),
                     4294967274),
                  4294967283)),
            4294967285)) ==
      0,
      0,
      .....
      
      
   ```

   



## 参考

1. [https://locore.cs.washington.edu/papers/nelson-hyperkernel.pdf](https://locore.cs.washington.edu/papers/nelson-hyperkernel.pdf)
2. [https://zhuanlan.zhihu.com/p/30105945](https://zhuanlan.zhihu.com/p/30105945)
3. [https://locore.cs.washington.edu/papers/nelson-hyperkernel-poster.pdf](https://locore.cs.washington.edu/papers/nelson-hyperkernel-poster.pdf)

   