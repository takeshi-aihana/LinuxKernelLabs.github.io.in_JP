## はじめに


### この講義の目的

   * オペレーティング・システムの基本的な用語とその概念

   * Linux カーネルの概要


### オペレーティング・システムの基本的な用語と概念


#### 「ユーザ」 vs 「カーネル」

「カーネル」と「ユーザ」はオペレーティング・システムの概念の中でよく出てくる用語です。
これらの意味はとても簡単です：
カーネルはユーザよりも高い権限で実行するオペレーティング・システムの一部であり、それに対しユーザは通常、低い権限で実行されるアプリケーションを意味しています。

しかし、これらの用語はちょっと大げさな表現であり、ある状況ではかなり特殊な意味を持つ場合があります。

「ユーザ・モード」と「カーネル・モード」は CPU プロセッサの実行モードを指す用語です。
カーネル・モードで実行するコードは完全に[^hypervisor] CPU を制御できますが、ユーザ・モードで実行されるコードは幾つか制限があります。
例えば、ローカル CPU の割り込みはカーネル・モードで実行中の間にのみ無効にしたり有効にすることができます。
もし、そのような処理がユーザ・モードで実行中に行われると、例外が発生してカーネルがその例外を引き継ぎます。

[^hypervisor]:プロセッサの中には Kernel モードよりも更に高い特権を持つものがあります。
例えば「ハイパーバイザ・モード」はハイパーバイザ（仮想マシンを監視するシステム）で実行しているコードにだけアクセスが可能です。

「ユーザ空間」と「カーネル空間」という用語は、特にメモリ保護、あるいはカーネルまたはユーザのアプリケーションのいずれかに関連づけられている仮想アドレス空間を指す場合があります。

説明をかなり単純化すると、カーネル空間はカーネルのために予約されているメモリ領域であり、それに対してユーザ空間は特定のユーザ・プロセスのために予約されたメモリ領域です。
カーネル空間へのアクセスは保護されているので、ユーザのアプリケーションから直接アクセスすることはできませんが、ユーザ空間にはカーネル・モードで実行しているコードから直接アクセスすることができます。


### 一般的なオペレーティングシステムの基本概念

一般的なオペレーティングシステムのアーキテクチャ（下図を参照のこと）において、オペレーティングシステムのカーネルの仕事は複数のアプリケーションの間でハードウェアへのアクセスやリソースの共有を安全かつ公平に行えるようにすることです。

![](images/Fig1-OperationgSystemArchitecture.png)

カーネルは、一般に「システム・コール」と呼ばれるアプリケーションが発行する API 一式を提供します。
これらの API は、その実行モードがユーザ・モードからカーネル・モードに切り替わる境界線にあたるため、通常のライブラリが提供する API とは異なります。

アプリケーションに互換性を提供するため、システム・コールが変更されることはめったにありません。
Linux の場合は、特にこのルールに厳格です（必要に応じて変更が可能なカーネル API とは対照的です）。

カーネルのコードそのものは、論理的にカーネルのコア部とデバイス・ドライバにそれぞれ分離が可能です。
デバイス・ドライバのコードは特定のデバイスへのアクセスを担当し、カーネルのコア部のコードは汎用的なコードです。
カーネルのコア部はさらに論理的なサブシステム（例えばファイルへのアクセス、ネットワーク、プロセス管理など）に分離できます。

### モノリシック・カーネル（*Monolithic kernel*）

「モノリシック・カーネル」ではカーネル内のサブシステム同士のアクセスは保護されておらず、またいろいろなサブシステム同士でグローバル関数を直接呼び出すことができるカーネルです。

![](images/Fig2-MonolithicKernel.png)

但し、ほとんどのモノリシック・カーネルはサブシステム間で論理的な独立を強制します。特にカーネルのコア部とデバイスドライバの間は比較的に厳格な API を使用します（但し、必ずしもそれで固定されているという訳ではない）。この API は一つのサブシステムまたは複数のデバイス・ドライバによって提供されるサービスにアクセスする際に使用します。
もちろん、これはカーネルの実装とそのアーキテクチャによって異なります。


## マイクロ・カーネル（*Micro kernel*）

「マイクロ・カーネル」は、カーネルの大部分がお互いに保護され、通常はユーザ空間で複数のサービスを実行しているカーネルです。
そこでは、いくつかあるカーネルの重要な部分がユーザ・モードで実行されているため、カーネル・モードで実行される残りのコードは非常に小さいと言うことがマイクロ・カーネルと言う名前の由来となります。

![](images/Fig3-MicroKernel.png)

マイクロ・カーネルのアーキテクチャにおいて、カーネルは実行中のいろいろなプロセスの間でメッセージをやり取りを可能にする十分なコードが含まれます。
実際には、カーネルの中にスケジューラと IPC のメカニズムが実装されている他、アプリケーションとカーネルのサービスとの間の保護機能を設定するための基本的なメモリ管理が実装されています。


このアーキテクチャの利点の一つは、カーネルのサービスが独立しているため、一つのサービスのバグが他のサービスに影響を与ることはないと言うことです。

したがって、もしカーネルのサービスがクラッシュしたらシステム全体に影響を与えることなく、そのサービスを再起動することができます。
しかしながらサービスの再起動はそれに依存する全てのアプリケーションに影響を与える可能性があるので（例えばファイル・サーバがクラッシュしたら、ファイル・ディスクプリタ経由でオープンしていたファイルにアクセスした全てのアプリケーションでエラーが発生します）、実際にこれを実現するのは困難です。

このアーキテクチャのカーネルにはモジュール型のアプローチが必要で、サービス間のメモリ保護機能を提供しますが、パフォーマンスが犠牲になります。
反対に、モノリシック・カーネルでは二つのサービスの間の簡単な関数呼び出しでも IPC とスケジューラを使う必要があるのでパフォーマンスが低下します[^minix-vs-linux]。


[^minix-vs-linux]:https://lwn.net/Articles/220255/


## マイクロ・カーネル vs モノリシック・カーネル

よくマイクロ・カーネルの擁護者は、モジュール型の設計手法を採用しているマイクロ・カーネルの方が優れていると主張します。
しかし、モノリシック・カーネルもモジュール化に対応することが可能で、これに関しては最新のモノリシック・カーネルを使用すると言う解決方法がいくつかあります：

   * モジュール化するコンポーネントはコンパイル時に有効または無効にできる

   * 実行中にロード可能なカーネル・モジュールをサポートする

   * カーネルを論理的で独立したサブシステムとして扱う

   * インタフェースは厳密であるが、パフォーマンスのオーバーヘッドが少ないマクロやインライン関数、そして関数のポインタを使う

かってモノリシック・カーネルとマイクロ・カーネル（例えば Windows や Mac OS X）の間にハイブリッド・カーネルなるオペレーティング・システムの種類がありました。
しかし、これらのオペレーティング・システムでは典型的なモノリシック・サービスが全てカーネル・モードで動くので、モノリシック・カーネル以外にそれらのサービスを qualify するメリットは殆どありません。


多くのオペレーティング・システムとカーネルの専門家たちは、このレッテルには意味はなく、ただの商用向けの売り文句だとしてはねつけています。
この件について Linus Torvalds 氏は次のように語っています：

> 「ハイブリッド・カーネル」そのものは - ただのマーケティング用語です。
> 「そうそう、マイクロ・カーネルには優れた「広告塔」がありました。我々が作業しているカーネルでも優れた広告塔を持つにはどうすればよいだろう？ ああ、こんなのはどうだろうか。かっこいい名前を付けて、他のシステムが持つ広告塔よりも全て優れてますよと言うことを間接的に伝えてみるというのは。」といった感じです。


## アドレス空間

   * 物理アドレス空間

     * RAM と周辺機器のメモリ

   * 仮想アドレス空間

     * CPU がメモリを認識する方法 (プロテクト・モード / ページング・モード の時)

     * プロセスのアドレス空間

     * カーネルのアドレス空間

「アドレス空間」は、さまざまな文脈で異なる意味を持つ多重定義が可能な用語です。

「物理アドレス空間」はメモリ・バス上にある RAM とデバイスのメモリの見え方を反映しています。
例えば、32ビットの Intel アーキテクチャの場合、一般的に物理メモリの低位の空間に RAM がマップされるのに対し、グラフィクス・カードのメモリは物理メモリの高位の空間にマップされます。

「仮想メモリ空間」（または単に「アドレス空間」）は仮想メモリ・モジュールが動きだした時（「プロテクト・モードが有効になった」時、または「ページングが有効になった」時）に CPU から見たメモリの見え方を反映したものになっています。
カーネルは、仮想メモリ空間の中から任意のメモリ領域を確保して、特定の物理メモリ領域に投影する「マッピング」を担当しています。

仮想アドレス空間に関連して、よく使用する別の用語が二つあります： それはプロセス（アドレス）空間とカーネル（アドレス）空間です。

「プロセス空間」は任意のプロセスに関連づけられた仮想アドレス空間（の一部）です。
プロセスの「メモリ表示」です。
0から始まる連続した領域です。
プロセスのアドレス空間がどこで終わるかは実装とアーキテクチャによって異なります。

「カーネル空間」はカーネル・モードで動作するコードの「メモリ表示」です。


## ユーザとカーネルが共有する仮想アドレス空間

ユーザ空間とカーネル空間で特徴ある実装は、仮想アドレス空間をユーザ空間のプロセスとカーネルとの間で共有する部分です。

この場合、カーネル空間はアドレス空間の一番上に位置し、ユーザ空間は反対に一番に下に置かれます。
ユーザ空間のプロセスがカーネル空間にアクセスできないようにするために、カーネルがユーザ・モードからカーネル・モードにアクセスできないようなマッピングを生成します。

![](images/Fig4-32bit-VirtualAddressSpace.png)


## いろいろな実行コンテキスト（*Execution contexts*）

   * プロセスのコンテキスト

     * ユーザ・モードで実行するコード（プロセスの一部）

     * プロセスが発行したシステム・コールの結果として、カーネル・モードで実行するコード


   * 割り込みのコンテキスト

     * 割り込みの結果として実行するコード

     * 常にカーネル・モードで実行する

カーネルの最も重要な仕事の一つがいろいろな「割り込み」を提供し、効率よく「割り込み」を提供するというものです。
これは、特別に実行する専用のコンテキストが関連づけられているほど、とても重要です。

割り込みが発生した結果としてカーネルが何かを実行する際は「割り込みのコンテキスト」の中で実行されます。
これには割り込みハンドラが含まれていますが、これに限定されることはなく、他にも割り込みモードで実行される特別なソフトウェアの概念があります。
割り込みのコンテキストの中で実行されるコードは常にカーネル・モードで動いていますが、カーネル・ハッカーが注意しなければならないお決まりの制限がいくつかあります（例えば、処理をブロックするような関数を呼び出さないこと、またはユーザ空間にアクセスしないこと）。

割り込みのコンテキストに対して、プロセスのコンテキストというものがあります。
プロセスのコンテキストの中で実行されるコードはユーザ・モード（アプリケーションの実行）またはカーネル・モード（システム・コールの実行）で処理されます。

## マルチ・タスク

   * An OS that supports the "simultaneous" execution of multiple processes

   * Implemented by fast switching between running processes to allow
     the user to interact with each program

   * Implementation:

     * Cooperative

     * Preemptive

Multitasking is the ability of the operating system to
"simultaneously" execute multiple programs. It does so by quickly
switching between running processes.

Cooperative multitasking requires the programs to cooperate to achieve
multitasking. A program will run and relinquish CPU control back
to the OS, which will then schedule another program.

With preemptive multitasking the kernel will enforce strict limits for
each process, so that all processes have a fair chance of
running. Each process is allowed to run a time slice (e.g. 100ms)
after which, if it is still running, it is forcefully preempted and
another task is scheduled.

Preemptive kernel
-----------------

.. slide:: Preemptive kernel
   :level: 2
   :inline-contents: True

   Preemptive multitasking and preemptive kernels are different terms.

   A kernel is preemptive if a process can be preempted while running
   in kernel mode.

   However, note that non-preemptive kernels may support preemptive
   multitasking.


Pageable kernel memory
----------------------

.. slide:: Pageable kernel memory
   :level: 2
   :inline-contents: True

   A kernel supports pageable kernel memory if parts of kernel memory
   (code, data, stack or dynamically allocated memory) can be swapped
   to disk.

Kernel stack
------------

.. slide:: Kernel stack
   :level: 2
   :inline-contents: True

   Each process has a kernel stack that is used to maintain the
   function call chain and local variables state while it is executing
   in kernel mode, as a result of a system call.

   The kernel stack is small (4KB - 12 KB) so the kernel developer has
   to avoid allocating large structures on stack or recursive calls
   that are not properly bounded.

Portability
-----------

In order to increase portability across various architectures and
hardware configurations, modern kernels are organized as follows at the
top level:

.. slide:: Portability
   :level: 2
   :inline-contents: True

   * Architecture and machine specific code (C & ASM)

   * Independent architecture code (C):

     * kernel core (further split in multiple subsystems)

     * device drivers

This makes it easier to reuse code as much as possible between
different architectures and machine configurations.


Asymmetric MultiProcessing (ASMP)
---------------------------------

Asymmetric MultiProcessing (ASMP) is a way of supporting multiple
processors (cores) by a kernel, where a processor is dedicated to the
kernel and all other processors run user space programs.

The disadvantage of this approach is that the kernel throughput
(e.g. system calls, interrupt handling, etc.) does not scale with the
number of processors and hence typical processes frequently use system
calls. The scalability of the approach is limited to very specific
systems (e.g. scientific applications).


.. slide:: Asymmetric MultiProcessing (ASMP)
   :level: 2
   :inline-contents: True

   .. ditaa::

                                  +-----------+
                                  |           |
              +------------------>|  Memory   |<-----------------+
              |                   |           |                  |
              |                   +-----------+                  |
              |                         ^                        |
              |                         |                        |
              v                         v                        v
      +--------------+          +---------------+         +---------------+
      |              |          |               |         |               |
      | Processor A  |          |  Processor B  |         |  Processor C  |
      |              |          |               |         |               |
      |              |          | +-----------+ |         | +-----------+ |
      |              |          | | Process 1 | |         | | Process 1 | |
      |              |          | +-----------+ |         | +-----------+ |
      |              |          |               |         |               |
      | +----------+ |          | +-----------+ |         | +-----------+ |
      | |  kernel  | |          | | Process 2 | |         | | Process 2 | |
      | +----------+ |          | +-----------+ |         | +-----------+ |
      |              |          |               |         |               |
      |              |          | +-----------+ |         | +-----------+ |
      |              |          | | Process 3 | |         | | Process 3 | |
      |              |          | +-----------+ |         | +-----------+ |
      +--------------+          +---------------+         +---------------+


Symmetric MultiProcessing (SMP)
-------------------------------

As opposed to ASMP, in SMP mode the kernel can run on any of the
existing processors, just as user processes. This approach is more
difficult to implement, because it creates race conditions in the
kernel if two processes run kernel functions that access the same
memory locations.

In order to support SMP the kernel must implement synchronization
primitives (e.g. spin locks) to guarantee that only one processor is
executing a critical section.

.. slide:: Symmetric MultiProcessing (SMP)
   :level: 2
   :inline-contents: True

   .. ditaa::

                                   +-----------+
                                   |           |
              +------------------->|  Memory   |<------------------+
              |                    |           |                   |
              |                    +-----------+                   |
              |                          ^                         |
              |                          |                         |
              v                          v                         v
      +---------------+          +---------------+         +---------------+
      |               |          |               |         |               |
      |  Processor A  |          |  Processor B  |         |  Processor C  |
      |               |          |               |         |               |
      | +-----------+ |          | +-----------+ |         | +-----------+ |
      | | Process 1 | |          | | Process 1 | |         | | Process 1 | |
      | +-----------+ |          | +-----------+ |         | +-----------+ |
      |               |          |               |         |               |
      | +-----------+ |          | +-----------+ |         | +-----------+ |
      | | Process 2 | |          | | Process 2 | |         | | Process 2 | |
      | +-----------+ |          | +-----------+ |         | +-----------+ |
      |               |          |               |         |               |
      | +-----------+ |          | +-----------+ |         | +-----------+ |
      | |   kernel  | |          | |   kernel  | |         | |   kernel  | |
      | +-----------+ |          | +-----------+ |         | +-----------+ |
      +---------------+          +---------------+         +---------------+


CPU Scalability
---------------

CPU scalability refers to how well the performance scales with
the number of cores. There are a few things that the kernel developer
should keep in mind with regard to CPU scalability:

.. slide:: CPU Scalability
   :level: 2
   :inline-contents: True

   * Use lock free algorithms when possible

   * Use fine grained locking for high contention areas

   * Pay attention to algorithm complexity


Overview the of Linux kernel
============================


Linux development model
-----------------------

.. slide:: Linux development model
   :level: 2

   * Open source, GPLv2 License

   * Contributors: companies, academia and independent developers

   * Development cycle: 3 – 4 months which consists of a 1 - 2 week
     merge window followed by bug fixing

   * Features are only allowed in the merge window

   * After the merge window a release candidate is done on a weekly
     basis (rc1, rc2, etc.)

The Linux kernel is one the largest open source projects in the world
with thousands of developers contributing code and millions of lines of
code changed for each release.

It is distributed under the GPLv2 license, which simply put,
requires that any modification of the kernel done on software that is
shipped to customer should be made available to them (the customers),
although in practice most companies make the source code publicly
available.

There are many companies (often competing) that contribute code to the
Linux kernel as well as people from academia and independent
developers.

The current development model is based on doing releases at fixed
intervals of time (usually 3 - 4 months). New features are merged into
the kernel during a one or two week merge window. After the merge
window, a release candidate is done on a weekly basis (rc1, rc2, etc.)


Maintainer hierarchy
--------------------

In order to scale the development process, Linux uses a hierarchical
maintainership model:

.. slide:: Maintainer hierarchy
   :level: 2
   :inline-contents: True

   * Linus Torvalds is the maintainer of the Linux kernel and merges pull
     requests from subsystem maintainers

   * Each subsystem has one or more maintainers that accept patches or
     pull requests from developers or device driver maintainers

   * Each maintainer has its own git tree, e.g.:

     * Linux Torvalds: git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux-2.6.git

     * David Miller (networking): git://git.kernel.org/pub/scm/linux/kernel/git/davem/net.git/

   * Each subsystem may maintain a -next tree where developers can submit
     patches for the next merge window

Since the merge window is only a maximum of two weeks, most of the
maintainers have a -next tree where they accept new features from
developers or maintainers downstream while even when the merge window
is closed.

Note that bug fixes are accepted even outside merge window in the
maintainer's tree from where they are periodically pulled by the
upstream maintainer regularly, for every release candidate.



Linux source code layout
-------------------------

.. slide:: Linux source code layout
   :level: 2
   :inline-contents: True

   .. ditaa::

      +-------+
      | linux |
      +-+-----+
        |
        +------+--------+---------+---------+--------------+--------------+
        |      |        |         |         |              |              |
        |      v        v         v         v              v              v
        |  +------+ +-------+ +-------+ +--------+ +---------------+ +---------+
        |  | arch | | block | | certs | | crypto | | Documentation | | drivers |
        |  +------+ +-------+ +-------+ +--------+ +---------------+ +---------+
        |
        +-------+----------+--------+---------+--------+--------+---------+
        |       |          |        |         |        |        |         |
        |       v          v        v         v        v        v         v
        |  +----------+ +----+ +---------+ +------+ +-----+ +--------+ +-----+
        |  | firmware | | fs | | include | | init | | ipc | | kernel | | lib |
        |  +----------+ +----+ +---------+ +------+ +-----+ +--------+ +-----+
        |
        +-----+------+---------+------------+------------+------------+
        |     |      |         |            |            |            |
        |     v      v         v            v            v            v
        |  +----+ +-----+ +---------+ +---------+  +----------+ +-------+
        |  | mm | | net | | samples | | scripts |  | security | | sound |
        |  +----+ +-----+ +---------+ +---------+  +----------+ +-------+
        |
        +------+--------+--------+
               |        |        |
               v        v        v
           +-------+ +-----+ +------+
           | tools | | usr | | virt |
           +-------+ +-----+ +------+


These are the top level of the Linux source code folders:

* arch - contains architecture specific code; each architecture is
  implemented in a specific sub-folder (e.g. arm, arm64, x86)

* block - contains the block subsystem code that deals with reading
  and writing data from block devices: creating block I/O requests,
  scheduling them (there are several I/O schedulers available),
  merging requests, and passing them down through the I/O stack to the
  block device drivers

* certs - implements support for signature checking using certificates

* crypto - software implementation of various cryptography algorithms
  as well as a framework that allows offloading such algorithms in
  hardware

* Documentation - documentation for various subsystems, Linux kernel
  command line options, description for sysfs files and format, device
  tree bindings (supported device tree nodes and format)

* drivers - driver for various devices as well as the Linux driver
  model implementation (an abstraction that describes drivers, devices
  buses and the way they are connected)

* firmware - binary or hex firmware files that are used by various
  device drivers

* fs - home of the Virtual Filesystem Switch (generic filesystem code)
  and of various filesystem drivers

* include - header files

* init - the generic (as opposed to architecture specific)
  initialization code that runs during boot

* ipc - implementation for various Inter Process Communication system
  calls such as message queue, semaphores, shared memory

* kernel - process management code (including support for kernel
  thread, workqueues), scheduler, tracing, time management, generic
  irq code, locking

* lib - various generic functions such as sorting, checksums,
  compression and decompression, bitmap manipulation, etc.

* mm - memory management code, for both physical and virtual memory,
  including the page,  SL*B and CMA allocators, swapping, virtual memory
  mapping, process address space manipulation, etc.

* net - implementation for various network stacks including IPv4 and
  IPv6; BSD socket implementation, routing, filtering, packet
  scheduling, bridging, etc.

* samples - various driver samples

* scripts - parts the build system, scripts used for building modules,
  kconfig the Linux kernel configurator, as well as various other
  scripts (e.g. checkpatch.pl that checks if a patch is conform with
  the Linux kernel coding style)

* security - home of the Linux Security Module framework that allows
  extending the default (Unix) security model as well as
  implementation for multiple such extensions such as SELinux, smack,
  apparmor, tomoyo, etc.

* sound - home of ALSA (Advanced Linux Sound System) as well as the
  old Linux sound framework (OSS)

* tools - various user space tools for testing or interacting with
  Linux kernel subsystems

* usr - support for embedding an initrd file in the kernel image

* virt - home of the KVM (Kernel Virtual Machine) hypervisor


Linux kernel architecture
-------------------------

.. slide:: Linux kernel architecture
   :level: 2
   :inline-contents: True

   .. ditaa::
      :height: 100%

      +---------------+  +--------------+      +---------------+
      | Application 1 |  | Application2 | ...  | Application n |
      +---------------+  +--------------+      +---------------+
              |                 |                      |
              v                 v                      v
      +--------------------------------------------------------+
      |                       Kernel                           |
      |                                                        |
      |   +----------------------+     +-------------------+   |
      |   |  Process Management  |     | Memory Management |   |
      |   +----------------------+     +-------------------+   |
      |                                                        |
      |   +------------+    +------------+    +------------+   |
      |   | Block I/O  |    |    VFS     |    | Networking |   |
      |   +------------+    +------------+    +------------+   |
      |                                                        |
      |   +------------+    +------------+    +------------+   |
      |   |    IPC     |    |  Security  |    |   Crypto   |   |
      |   +------------+    +------------+    +------------+   |
      |                                                        |
      |   +------------+    +------------+    +------------+   |
      |   |    DRM     |    |    ALSA    |    |    USB     |   |
      |   +------------+    +------------+    +------------+   |
      |                        ...                             |
      +--------------------------------------+-----------------+
      |           Device drivers             |     arch        |
      |                                      |                 |
      | +----+ +-----+ +--------+ +----+     |  +----------+   |
      | |char| |block| |ethernet| |wifi|     |  | machine 1|   |
      | +----+ +-----+ +--------+ +----+     |  +----------+   |
      | +----------+ +-----+ +----+ +---+    |  +----------+   |
      | |filesystem| |input| |iio | |usb|    |  | machine 2|   |
      | +----------+ +-----+ +----+ +---+    |  +----------+   |
      | +-----------+ +----------+  +---+    |                 |
      | |framebuffer| | platform |  |drm|    |     ...         |
      | +-----------+ +----------+  +---+    |                 |
      +-------------------------+----+-------+-----------------+
              |                 |                      |
              v                 v                      v

      +--------------------------------------------------------+
      |                         Hardware                       |
      +--------------------------------------------------------+


arch
....

.. slide:: arch
   :level: 2
   :inline-contents: True

   * Architecture specific code

   * May be further sub-divided in machine specific code

   * Interfacing with the boot loader and architecture specific
     initialization

   * Access to various hardware bits that are architecture or machine
     specific such as interrupt controller, SMP controllers, BUS
     controllers, exceptions and interrupt setup, virtual memory handling

   * Architecture optimized functions (e.g. memcpy, string operations,
     etc.)

This part of the Linux kernel contains architecture specific code and
may be further sub-divided in machine specific code for certain
architectures (e.g. arm).

"Linux was first developed for 32-bit x86-based PCs (386 or
higher). These days it also runs on (at least) the Compaq Alpha AXP,
Sun SPARC and UltraSPARC, Motorola 68000, PowerPC, PowerPC64, ARM,
Hitachi SuperH, IBM S/390, MIPS, HP PA-RISC, Intel IA-64, DEC VAX, AMD
x86-64 and CRIS architectures.”

It implements access to various hardware bits that are architecture or
machine specific such as interrupt controller, SMP controllers, BUS
controllers, exceptions and interrupt setup, virtual memory handling.

It also implements architecture optimized functions (e.g. memcpy,
string operations, etc.)


Device drivers
..............

.. slide:: Device drivers
   :level: 2

   * Unified device model

   * Each subsystem has its own specific driver interfaces

   * Many device driver types (TTY, serial, SCSI, fileystem, ethernet,
     USB, framebuffer, input, sound, etc.)

The Linux kernel uses a unified device model whose purpose is to
maintain internal data structures that reflect the state and structure
of the system. Such information includes what devices are present,
what is their status, what bus they are attached to, to what driver
they are attached, etc. This information is essential for implementing
system wide power management, as well as device discovery and dynamic
device removal.

Each subsystem has its own specific driver interface that is tailored
to the devices it represents in order to make it easier to write
correct drivers and to reduce code duplication.

Linux supports one of the most diverse set of device drivers type,
some examples are: TTY, serial, SCSI, fileystem, ethernet, USB,
framebuffer, input, sound, etc.


Process management
..................

.. slide:: Process management
   :level: 2

   * Unix basic process management and POSIX threads support

   * Processes and threads are abstracted as tasks

   * Operating system level virtualization

     * Namespaces

     * Control groups

Linux implements the standard Unix process management APIs such as
fork(), exec(), wait(), as well as standard POSIX threads.

However, Linux processes and threads are implemented particularly
different than other kernels. There are no internal structures
implementing processes or threads, instead there is a :c:type:`struct
task_struct` that describe an abstract scheduling unit called task.

A task has pointers to resources, such as address space, file
descriptors, IPC ids, etc. The resource pointers for tasks that are
part of the same process point to the same resources, while resources
of tasks of different processes will point to different resources.

This peculiarity, together with the `clone()` and `unshare()` system
call allows for implementing new features such as namespaces.

Namespaces are used together with control groups (cgroup) to implement
operating system virtualization in Linux.

cgroup is a mechanism to organize processes hierarchically and
distribute system resources along the hierarchy in a controlled and
configurable manner.


Memory management
.................

Linux memory management is a complex subsystem that deals with:

.. slide:: Memory management
   :level: 2
   :inline-contents: True

   * Management of the physical memory: allocating and freeing memory

   * Management of the virtual memory: paging, swapping, demand
     paging, copy on write

   * User services: user address space management (e.g. mmap(), brk(),
     shared memory)

   * Kernel services: SL*B allocators, vmalloc



Block I/O management
....................

The Linux Block I/O subsystem deals with reading and writing data from
or to block devices: creating block I/O requests, transforming block I/O
requests (e.g. for software RAID or LVM), merging and sorting the
requests and scheduling them via various I/O schedulers to the block
device drivers.

.. slide:: Block I/O management
   :level: 2
   :inline-contents: True

   .. ditaa::
      :height: 100%

      +---------------------------------+
      |    Virtual Filesystem Switch    |
      +---------------------------------+
                     ^
                     |
                     v
      +---------------------------------+
      |         Device Mapper           |
      +---------------------------------+
                     ^
                     |
                     v
      +---------------------------------+
      |       Generic Block Layer       |
      +---------------------------------+
                     ^
                     |
                     v
      +--------------------------------+
      |          I/O scheduler         |
      +--------------------------------+
             ^                ^
             |                |
             v                v
      +--------------+  +--------------+
      | Block device |  | Block device |
      |    driver    |  |    driver    |
      +--------------+  +--------------+


Virtual Filesystem Switch
.........................

The Linux Virtual Filesystem Switch implements common / generic
filesystem code to reduce duplication in filesystem drivers. It
introduces certain filesystem abstractions such as:

* inode - describes the file on disk (attributes, location of data
  blocks on disk)

* dentry - links an inode to a name

* file - describes the properties of an opened file (e.g. file
  pointer)

* superblock - describes the properties of a formatted filesystem
  (e.g. number of blocks, block size, location of root directory on
  disk, encryption, etc.)

.. slide:: Virtual Filesystem Switch
   :level: 2
   :inline-contents: True

   .. ditaa::
      :height: 100%


             ^                    ^                    ^
             | stat               | open               | read
             v                    v                    v
      +------------------------------------------------------------+
      |                   Virtual Filesystem Switch                |
      |                                                            |
      |                                                            |
      |    /-------\           /--------\           /--------\     |
      |    | inode |<----------+ dentry |<----------+  FILE  |     |
      |    \---+---/           \----+---/           \---+----/     |
      |        |                    |                   |          |
      |        |                    |                   |          |
      |        v                    v                   v          |
      |    +-------+           +--------+           +-------+      |
      |    | inode |           | dentry |           | page  |      |
      |    | cache |           | cache  |           | cache |      |
      |    +-------+           +--------+           +-------+      |
      |                                                            |
      +------------------------------------------------------------+
                   ^                                  ^
                   |                                  |
                   v                                  v
            +-------------+                    +-------------+
            | Filesystem  |                    | Filesystem  |
            |   driver    |                    |   driver    |
            +-------------+                    +-------------+


The Linux VFS also implements a complex caching mechanism which
includes the following:

* the inode cache - caches the file attributes and internal file
  metadata

* the dentry cache - caches the directory hierarchy of a filesystem

* the page cache - caches file data blocks in memory



Networking stack
................

.. slide:: Networking stack
   :level: 2
   :inline-contents: True

   .. ditaa::
      :height: 100%

      +---------------------------+
      | Berkeley Socket Interface |
      +---------------------------+

      +---------------------------+
      |      Transport layer      |
      +-------------+-------------+
      |      TCP    |     UDP     |
      +-------------+-------------+

      +---------------------------+
      |      Network layer        |
      +-----+---------+-----------+
      | IP  | Routing | NetFilter |
      +-----+---------+-----------+

      +---------------------------+
      |     Data link layer       |
      +-------+-------+-----------+
      |  ETH  |  ARP  | BRIDGING  |
      +-------+-------+-----------+

      +---------------------------+
      |    Queuing discipline     |
      +---------------------------+

      +---------------------------+
      | Network device drivers    |
      +---------------------------+

Linux Security Modules
......................

.. slide:: Linux Security Modules
   :level: 2
   :inline-contents: True

   * Hooks to extend the default Linux security model

   * Used by several Linux security extensions:

     * Security Enhancened Linux

     * AppArmor

     * Tomoyo

     * Smack
