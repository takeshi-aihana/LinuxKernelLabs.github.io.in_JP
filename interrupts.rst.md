* [目次](/README.md#目次index)
* [システム・コール](/syscalls.rst.md#システムコール)

---

## 割り込み

### この講義の目的

   * いろいろな割り込みと例外（x86 の場合）

   * いろいろな割り込みと例外 (Linux の場合)

   * 遅延可能な作業

   * いろいろなタイマー


### 「割り込み」とは何か？

「割り込み」とはプログラムの通常の実行フローを変更する1個のイベントであり、いろいろなハードウェア・デバイスや CPU 自身によって生成されます。

割り込みは、その生成方法に基づいて二つのグループに分類することができます：

   * 同期方法

     - **同期型** （*synchronous*）： これは実行中の命令が生成する割り込み
     - **非同期型** （*asynchronous*）：これは外部のイベントが生成する割り込み

  * マスク可否

     - **マスク可能** （*maskable*）： これは無視することができる割り込みで、CPU の ``INT`` ピン経由で発行される
     - **マスク不可** （*non-maskable*）：これは無視できない割り込みで、CPU の ``NMI`` ピン経由で発行される

通常、「例外」と呼ばれる同期型の割り込みは、一個の命令を実行している過程でプロセッサ自身が検出したさまざまな状態を扱います。
例えば「０による除算」（*Divide by Zero*）やシステムコールは「例外」に該当します。

通常、「割り込み」と呼ばれる非同期型の割り込みはいろいろな I/O デバイスによって生成される外部イベントです。
例えばネットワーク・カードは割り込みを生成してパケットが到着したことを通知します。


#### いろいろな例外

例外の生成元は二つあります:

   * プロセッサの場合：

     - **fault**
     - **trap**
     - **abort**

   * プログラムの場合

     - **int N**

CPU が命令を実行中に異常状態（*Abnormal Condition*) を検出すると、プロセッサが検出したいろいろな例外が発生します。

例外の一種である「フォルト（**fault**）」は CPU が命令を実行する前に発生し、通常は修正が可能です。
保存された ``EIP`` にはフォルト発生の原因となった命令のアドレスが格納されているので、プログラムはこのフォルトを修正した後に問題のあった命令を再び実行することができます。
この例外は、例えばページ・フォルトなどがあります。

「トラップ（**trap**）」も例外の一種で、これは例外が検出された命令を実行した後に発生します。
保存された ``EIP`` にはトラップ発生の原因となった命令の次の命令のアドレスが格納されます。
この例外は、例えばデバッグのトラップがあります。



### ハードウェアの概念

#### 割り込みコントローラ（*Programmable Interrupt Controller*)

![](images/Fig14-Hardware_PIC.png)

割り込みをサポートするデバイスは、割り込みの要求（*Interrupt ReQuest*）を発行する際に使用する出力ピンを持っています。
この「``IRQ`` ピン」は「割り込みコントローラ（``PIC``）」と呼ばれるデバイスに接続され、さらに PIC は CPU の ``INTR`` ピンに接続しています。

通常 PIC には CPU と情報を交換する際に使用する複数のポートがあります。
PIC が持つ IRQ 線の一つに接続している任意のデバイスが CPU から「注目」してもらいたい時、次の手順で処理が発生します：

   1. デバイスが該当する ``IRQ``*N* ピン上で割り込みを発生させる
   1. PIC が IRQ をベクタ番号に変換し、その番号を CPU が読み取れるようにするために PIC のポートに書き込む
   1. PIC が CPU の INTR ピン上で割り込みを発生させる
   1. PIC は CPU が割り込みを認識するまで待つ
   1. CPU が割り込みを処理する


CPU が割り込みを処理する方法はのちほど説明します。
重要な点は、設計上 PIC は CPU が現在の割り込みを認識するまで別の割り込みを発生させることはないと言うことです。

複数ある IRQ 線は個別に無効にすることができます。
これにより複数の割り込みハンドラが常に順番に実行されることを保証することができ、さらに設計が簡略化できます。


#### x86 専用割り込みコントローラ（*Advanced Programmable Interrupt Controller*)

![](images/Fig15-Hardware_APIC.png)

複数の CPU コアを持つシステムでは、コア毎に一個の「ローカル ``APIC`` （*Local APIC*）」を持ち、タイマー割り込みや温度センサのようなコア専用で接続しているデバイスからの割り込みを処理します。

「I/O APIC」は外部のデバイスから CPU コアに IRQ を分配する際に使用します。

これらのハードウェアについて説明した後に、どのようにプロセッサが一個の割り込みを処理するのかを見てみることにしましょう。

#### 割り込みの制御

割り込みハンドラと、他の潜在的な並列処理（例えばドライバの初期化やドライバでのデータ処理）との間でデータを共有するためにアクセスを同期するには、制御された方法で割り込みを有効にしたり無効にする必要がでてきます。

これは複数のレベルで実現することが可能です：

   * デバイスのレベル

     * デバイスのコントロール・レジスタをプログラミングすることで実現する

   * PIC のレベル

     * PIC は指定した IRQ 線を無効にするようプログラミングが可能である

   * CPU のレベル

     * 例えば x86 系では次の命令を使用できる：

         * ``cli`` (_CLear Interrupt flag_ / 割り込みフラグをクリアする)

         * ``sti`` (_SeT Interrupt flag_ / 割り込みフラグをセットする)


### アーキテクチャ専用の割り込み処理（Linuxの場合）

このセクションでは、Linux で x86 系アーキテクチャ向けの割り込みを処理する方法について説明します。

#### 割り込みディスクリプタ・テーブル（*Interrupt Descriptor Table*）

「割り込みディスクリプタ・テーブル（``IDT``）」は割り込みや例外の識別子と対応するイベントを処理する命令のディスクリプタを関連付けたものです。
ここでは割り込みや例外の識別子を「ベクタ番号」と呼び、イベントを処理する命令を「割り込み/例外ハンドラ」と呼ぶことにします。

``IDT`` には次のような特徴があります：

   * ``IDT`` は、特定のベクタが発行されたら CPU によって使用される「ジャンプ・テーブル」（ハンドラへのポインタまたは機械語のジャンプ命令を格納した配列）である
   * ``IDT`` は、 256 x 8 バイトのエントリを持つ配列である
   * ``IDT`` は、物理メモリのどこにでも常駐できる
   * プロセッサは ``IDTR`` を使って ``IDT`` を特定する

以下に Linux における ``IRQ`` ベクタのレイアウトを示します。
最初の32個のエントリは例外用に予約され、「ベクタ#128」がシステムコールのインタフェースとして使用される以外、残りは主にハードウェアの割り込みハンドラとして使用されます。

![](images/Fig16-LinuxIRQVectorLayout.png)

x86 系アーキテクチャでは一個の IDT エントリのサイズは 8 バイトで、「ゲート（*Gate*）」と呼ばれています。
ゲートには三つの種類があります：

  * 割り込みゲート

    このゲートは割り込みハンドラまたは例外ハンドラのアドレスを保持しており、これらのハンドラにジャンプすると（もし割り込みフラグがクリアされていれば）マスク可能な割り込みが無効になる

  * トラップ・ゲート

    このゲートは割り込みゲートに似ていますが、割り込みハンドラや例外ハンドラにジャンプしてもマスク可能な割り込みを無効にしない点が違う

  * タスク・ゲート

    Linux では使用しない

それでは IDT エントリの各項目を見てみることにしましょう：

   * セグメント・セレクタ

     割り込みハンドラが格納されているコード・セグメント（の開始アドレス）を見つけるために使用する ``GDT/LDT`` 内のインデックス

   * オフセット

     コード・セグメント内でのオフセット値

   * T

     ゲートの種類を表す

   * DPL

     セグメントの内容を利用する際に必要となる最小限の権限


![](images/Fig17-InterruptDescripterTableEntry.png)


#### 割り込みハンドラのアドレス

割り込みハンドラの命令が格納されているアドレス（``ISR`` アドレス / *Interrupt Service Routine Address* ）を見つけるには、まずその命令が格納されているコード・セグメントの開始アドレスを見つける必要があります。
そのためセグメント・セレクタを使って ``GDB/LDT`` 内で該当する情報を検索します。これは対応するセグメント・ディスクリプタを見つけることができる情報です。
これは「ベース」項目に保持されている開始アドレスが提供されます。
そしてベース・アドレスとオフセットを使って、割り込みハンドラの先頭にジャンプすることができます。

![](images/Fig18-InterruptHandlerAddress.png)


#### 割り込みハンドラのスタック

通常の関数を呼び出す場合と同様に、割り込みハンドラや例外ハンドラの呼び出しも割り込みを発生させた箇所（コード）へ戻るために必要な情報を保存しておくのにスタックを使用します。

次の図でもわかるように、割り込みが発生した命令（コード）のアドレスを保存する前に ``EFLAGS`` レジスタを Push しておきます。
特定の種類の例外もまたデバッグしやすくするために、その例外が発生した時の問題あるコードをスタックに Push しておきます。


![](images/Fig19-InterruptHandlerStack.png)


#### 割り込み要求の処理

一個の割り込みが生成されると、プロセッサは次に示すイベント・シーケンスを実行します。このシーケンスは最終的にカーネルの割り込みハンドラを呼び出します：

   1. CPU は現在の特権レベルをチェックする
   1. <特権レベルを変更する必要がある場合>
      * 現在使用中のスタックを、新しい特権に関連づけられたスタックに変更する
      * 古いスタックの情報を新しいスタックに Push する

   1. ``EFLAGS``、``CS``、``EIP`` の情報をスタックに Push する
   1. <**abort** の場合> はエラー・コードをスタックに Push する
   1. カーネルの割り込みハンドラを呼び出す


#### 割り込みハンドラから戻る処理

ほとんどのアーキテクチャは、割り込みハンドラを実行したあとにスタックをクリアして（一時停止していた）実行を再開する特別な命令を提供しています。
x86 系では、割り込みハンドラから戻る際に ``IRET`` 命令を使用します。
``IRET`` は ``RET`` 命令と似ていますが、前者はスタックに保存しているフラグのために ``ESP`` を4バイト余分にカウントし、そのフラグを ``EFLAGS`` レジスタに Pop してくる点が違います。

x86 系では、割り込みの処理のあとに実行を再開する際は次のシーケンスを実行します：

   1. <**abort** の場合> エラー・コードを Pop する
   1. ``IRET`` 命令を呼び出す
      * スタックからいろいろな値を Pop して、``CS``、``EIP``、``EFLAGS`` といったレジスタにリストアする
      * もし <特権レベルを変更していたら> 古いスタックと古い特権レベルに戻す


### Generic interrupt handling in Linux

In Linux the interrupt handling is done in three phases: critical, immediate and deferred.

In the first phase the kernel will run the generic interrupt handler that determines the interrupt number, the interrupt handler for this particular interrupt and the interrupt controller.
At this point any timing critical actions will also be performed (e.g. acknowledge the interrupt at the interrupt controller level).
Local processor interrupts are disabled for the duration of this phase and continue to be disabled in the next phase.

In the second phase all of the device drivers handler associated with this interrupt will be executed [#f1]_.
At the end of this phase the interrupt controller's "end of interrupt" method is called to allow the interrupt controller to reassert this interrupt.
The local processor interrupts are enabled at this point.

.. [#f1] Note that it is possible that one interrupt is associated with multiple
	 devices and in this case it is said that the interrupt is
	 shared. Usually, when using shared interrupts it is the responsibility
	 of the device driver to determine if the interrupt is target to it's
	 device or not.

Finally, in the last phase of interrupt handling interrupt context deferrable actions will be run.
These are also sometimes known as "bottom half" of the interrupt (the upper half being the part of the interrupt handling that runs with interrupts disabled).
At this point interrupts are enabled on the local processor.


![](images/Fig20-InterruptHandlingInLinux.png)


Nested interrupts and exceptions
--------------------------------

Nesting interrupts is permitted on many architectures.
Some architectures define interrupt levels that allow preemption of an interrupt only if the pending interrupt has a greater priority then the current (settable) level (e.g see ARM's priority mask).

In order to support as many architectures as possible, Linux has a more restrictive interrupt nesting implementation:

.. slide:: IRQ nesting in Linux
   :inline-contents: True
   :level: 2

   * an exception (e.g. page fault, system call) can not preempt an interrupt;
     if that occurs it is considered a bug

   * an interrupt can preempt an exception or other interrupts; however, only
     one level of interrupt nesting is allowed

The diagram below shows the possible nesting scenarios:

![](images/Fig21-IRQ_NestingInLinux.png)


Interrupt context
-----------------

While an interrupt is handled (from the time the CPU jumps to the interrupt
handler until the interrupt handler returns - e.g.  IRET is issued) it is said
that code runs in "interrupt context".

Code that runs in interrupt context has the following properties:

.. slide:: Interrupt context
   :inline-contents: True
   :level: 2

    * it runs as a result of an IRQ (not of an exception)
    * there is no well defined process context associated
    * not allowed to trigger a context switch (no sleep, schedule, or user memory access)

Deferrable actions
------------------

Deferrable actions are used to run callback functions at a later time. If
deferrable actions scheduled from an interrupt handler, the associated callback
function will run after the interrupt handler has completed.

There are two large categories of deferrable actions: those that run in
interrupt context and those that run in process context.

The purpose of interrupt context deferrable actions is to avoid doing too much
work in the interrupt handler function. Running for too long with interrupts
disabled can have undesired effects such as increased latency or poor system
performance due to missing other interrupts (e.g. dropping network packets
because the CPU did not react in time to dequeue packets from the network
interface and the network card buffer is full).

In Linux there are three types of deferrable actions:

.. slide:: Deferrable actions in Linux
   :inline-contents: True
   :level: 2


    * softIRQ

      * runs in interrupt context
      * statically allocated
      * same handler may run in parallel on multiple cores

    * tasklet

      * runs in interrupt context
      * can be dynamically allocated
      * same handler runs are serialized

    * workqueues

      * run in process context

Deferrable actions have APIs to: **initialize** an instance, **activate** or
**schedule** the action and **mask/disable** and **unmask/enable** the execution
of the callback function. The later is used for synchronization purposes between
the callback function and other contexts.

Soft IRQs
---------

Soft IRQs is the term used for the low level mechanism that implements deferring
work from interrupt handlers but that still runs in interrupt context.

.. slide:: Soft IRQs
   :inline-contents: True
   :level: 2

    Soft IRQ APIs:

      * initialize: :c:func:`open_softirq`
      * activation: :c:func:`raise_softirq`
      * masking: :c:func:`local_bh_disable`, :c:func:`local_bh_enable`

    Once activated, the callback function :c:func:`do_softirq` runs either:

      * after an interrupt handler or
      * from the ksoftirqd kernel thread


.. slide:: ksoftirqd
   :inline-contents: False
   :level: 2

    * minimum priority kernel thread
    * runs softirqs after certain limits are reached
    * tries to achieve good latency and avoid process starvation


Since softirqs can reschedule themselves or other interrupts can occur that
reschedules them, they can potentially lead to (temporary) process starvation if
checks are not put into place. Currently, the Linux kernel does not allow
running soft irqs for more than :c:macro:`MAX_SOFTIRQ_TIME` or rescheduling for
more than :c:macro:`MAX_SOFTIRQ_RESTART` consecutive times.

Once these limits are reached a special kernel thread, **ksoftirqd** is wake-up
and all of the rest of pending soft irqs will be run from the context of this
kernel thread.

Soft irqs usage is restricted, they are use by a handful of subsystems that have
low latency requirements. For 4.19 this is the full list of soft irqs:

.. slide:: Types of soft IRQ
   :inline-contents: True
   :level: 2

    * HI_SOFTIRQ
    * TIMER_SOFTIRQ
    * NET_TX_SOFTIRQ
    * NET_RX_SOFTIRQ
    * BLOCK_SOFTIRQ
    * IRQ_POLL_SOFTIRQ
    * TASKLET_SOFTIRQ
    * SCHED_SOFTIRQ
    * HRTIMER_SOFTIRQ,
    * RCU_SOFTIRQ

Tasklets
--------

.. slide:: Tasklets
   :inline-contents: True
   :level: 2

   Tasklets are a dynamic type (not limited to a fixed number) of
   deferred work running in interrupt context.

   Tasklets API:

    * initialization: :c:func:`tasklet_init`
    * activation: :c:func:`tasklet_schedule`
    * masking: :c:func:`tasklet_disable`, :c:func:`tasklet_enable`

   Tasklets are implemented on top of two dedicated softirqs:
   :c:macro:`TASKLET_SOFITIRQ` and :c:macro:`HI_SOFTIRQ`

   Tasklets are also serialized, i.e. the same tasklet can only execute on one processor.


Workqueues
----------

 .. slide:: Workqueues
   :inline-contents: True
   :level: 2

   Workqueues are a type of deferred work that runs in process context.

   They are implemented on top of kernel threads.

   Workqueues API:

    * init: :c:macro:`INIT_WORK`
    * activation: :c:func:`schedule_work`

Timers
------

.. slide:: Timers
   :inline-contents: True
   :level: 2

    Timers are implemented on top of the :c:macro:`TIMER_SOFTIRQ`

    Timer API:

    * initialization: :c:func:`setup_timer`
    * activation: :c:func:`mod_timer`

---

* [目次](/README.md#目次index)
