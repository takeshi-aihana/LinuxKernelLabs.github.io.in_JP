* [目次](/README.md#目次index)
* [割り込み](/interrupts.rst.md#割り込み)

---

## 対照型マルチプロセッシング

### この講義の目的

   * カーネルの並列処理

   * アトミックな操作

   * スピン・ロック

   * キャッシュのスラッシング（*thrashing*）

   * 最適化したスピン・ロック

   * プロセスと割り込みのコンテキストの同期

   * ミューテックス（*Mutexes*）

   * CPU ごとのデータ

   * メモリ・オーダリング（*Ordering*）とメモリ・バリア（*Barrier*）

   * リード・コピー・アップデート（*Read-Copy Update*）


### 同期の基本

Linux カーネルが「対照型マルチプロセッシング（**SMP**）」をサポートしているので、一連の同期メカニズムを使用して競合状態のない期待したとおりの結果を出してあげる必要があります。


---

##### Note

この講義では「コア（*Core*）」と「CPU」 と「プロセッサ」という用語を同じ意味で使用しています。

---


次に示す二つの状態が同時に発生すると競合状態になる可能性があります：

 1. 「並列」実行される実行コンテキストが最低二つ存在する状態:

    * 完全に並列実行する（例: 二つのシステム・コールが別々のプロセッサで処理される）

    * 複数ある実行コンテキストの一つが他の実行コンテキストを任意にプリエンプトする（CPU の実行権を奪う）（例： 割り込みがシステム・コールをプリエンプトする）

 2. 実行コンテキストが共有メモリに対して読み書きのアクセスを実行している状態

競合状態は、実行コンテキストが CPU コア上でかなり特殊な順番でスケジューリングされた時にだけ出現するので、デバッグが困難で間違った結果につながる可能性があります。

ここに、古典的な競合状態の例として間違ったリソース・カウンタの実装を持ったリソースの解放処理があります：

```c

      void release_resource()
      {
          counter--;

          if (!counter)
              free_resource();
      }
```

リソース・カウンタは共有リソースを最後の使用者が解放するまで利用できるようにしておくための仕組みですが、上記の実装には競合状態になるとリソースを二回解放する問題があります。

![](images/Fig22-RaceConditionScenario.png)

通常、``release_resource()`` 関数はリソースを一度だけ解放します。
しかし上の例で、``counter`` を一つ減らした直後にスレッド A がプリエンプトされ、代わりにスレッド B が ``release_resource()`` を呼び出した場合でもリソースが解放されます。
それからスレッド A に制御が戻ったら、``counter`` は 0 なのでリソースが解放されてしまいます（これがリソース解放が二回行われる問題です）。

この競合状態を回避するには、プログラマはまずその競合状態を生み出す「クリティカル・セクション（*Critical Section*）」を特定する必要があります。
クリティカル・セクションは複数の並列コンテキストから共有メモリを読み書きするコードの一部です。

上の例で言うと、最小のクリティカル・セクションは ``counter`` を一つ減らす処理から、``counter`` の値を確認するまでです。

一度、クリティカル・セクションを特定したら、次のいずれか一つの方法で競合状態を回避できます：

   * クリティカル・セクションを **アトミック（*atomic*）** にする（例： アトミックな命令を使う）

   * クリティカル・セクションの間は **プリエンプトを無効にする** （例：割り込みやボトム・ハーフのハンドラ、あるいはスレッドのプリエンプトを無効にする）

   * クリティカル・セクションに対する **アクセスをシリアル化する** （例：スピン・ロックやミューテックスを使用し、クリティカル・セクションを実行できるコンテキストやスレッドを一個だけに限定する）


### Linux カーネルのいろいろな並列処理（*Linux kernel concurrency sources*）

Linux カーネルにはカーネルの設定と、そのカーネルを実行するシステムの種類に応じた並列処理が複数あります：

   * **シングル・コアのシステム** ＋ **非プリエンプティブ・カーネル**: 現在のプロセスは割り込みによってプリエンプト（実行が中断）することが可能

   * **シングル・コアのシステム** ＋  **プリエンプティブ・カーネル**: 上記に加え、現在のプロセスは他のプロセスによってプリエンプト（実行が中断）することが可能

   * **マルチ・コアのシステム**: 上記に加え、現在のプロセスは別のプロセスまたは別のプロセッサで実行中の割り込みと平行して実行することが可能


---

##### Note

この講義ではカーネルの並列処理についてのみ説明します。シングル・コアのシステムで動いている非プリエンプティブ・カーネルの場合、プロセスと並列で処理されるのは割り込み処理だけになります。

---


### アトミックな操作（*Atomic operations*）

特定の状況ではハードウェアが提供する「アトミックな操作（*Atomic operations*)」を使えば競合状態を回避することは可能です。
Linux ではアトミックな操作にアクセスするための（ハードウェアに依存しない）統一的な API を提供しています：

   * 整数系:

     * 簡易版: ``atomic_inc()``、``atomic_dec()``、``atomic_add()``, ``atomic_sub()``

     * 条件付き: ``atomic_dec_and_test()``、``atomic_sub_and_test()``

   * ビット系:

     * 簡易版: ``test_bit()``、``set_bit()``、``change_bit()``

     * 条件付き: ``test_and_set_bit()``、``test_and_clear_bit()``、``test_and_change_bit()``


例えば ``atomic_dec_and_test()`` という関数を使って、リソース・カウンタを一つ減らしその値をチェックするという一連のアトミックな処理を実装できます。

```c

      void release_resource()
      {
          if (atomic_dec_and_test(&counter))
               free_resource();
      }
```

アトミックな操作に伴う複雑さの一つがマルチ・コアシステムで発生するという点です。すなわち、アトミックな操作はシステム・レベルではアトミックではなくなるということです（但し、コア・レベルでは依然としてアトミックです）。

この理由を理解するために、アトミックな操作をメモリのロードとストアの操作に分解する必要があります。
すると、ロードとストアの命令が複数の CPU 間で交互に処理されるような状態で競合状態が発生するシナリオを作ることができます。
例えば、一つの値を二つのプロセッサを使ってカウントアップすると予期しない結果が生じるといった以下の例のようなものです：


![](images/Fig23-AtomicOperationMayNotBe.png)


SMP のシステムでアトミックな操作を提供するために、異なるアーキテクチャがそれぞれ異なる方法を採用しています。
例えば x86 アーキテクチャの場合は ``LOCK`` という接頭詞を使い、この接頭詞が付いている操作を実行している間はシステムバスをロックします：


![](images/Fig24-FixingAtomicOperation.png)


ARM アーキテクチャの場合は ``LDREX`` 命令と ``STREX`` 命令を一緒に使用してアトミックなアクセスを保証しています。
``LDREX`` 命令は値をロードしアトミックな操作が進行中であることを「排他モニタ（*Exclusive Monitor*）」に通知します。
次に ``STREX`` 命令が新しい値をストアしようとしますが、排他モニタが他の排他処理を検出していなかった場合にのみストアが成功します。
したがって、アトミックな操作を実現するためにプログラマは、排他モニタが排他処理可能であることを通知するまで（``LDREX`` と ``STREX`` の両方の）処理をリトライさせる必要があります。

この方式はしばしば「軽量な」または「効率が良い」同期メカニズムとして解釈されます
（その理由は「この方式がスピン・ロックやコンテキスト・スイッチが不要だから」とか、「この方式がハードウェアの実装なので、もっと効率よくなるはずだ」とか、「この方式はただの命令なので、他の命令と同様に効率がよくないといけない」というものがあります）。
しかし実装の詳細を見るとわかるように、アトミックな操作は実際には「コストが高い」処理であることがわかります。


### プリエンプティブ機能の無効化（割り込み）

（前述のとおり、）シングル・コアのシステムで非プリエンプティブなカーネルにおける並列処理とは、現在のスレッドが一個の割り込みによってプリエンプト（実行が中断）されるケースしかありません。
したがって並列処理にならないようにするには割り込みを無効（*Disabling*）にするだけで事が足ります。

これはアーキテクチャ毎に専用の命令を実行することで実現されていますが、Linux では「アーキテクチャに依存せずに」割り込みを無効にしたり有効する API をいくつか提供しています：

```c
       #define local_irq_disable() \
           asm volatile („cli” : : : „memory”)

      #define local_irq_enable() \
          asm volatile („sti” : : : „memory”)

      #define local_irq_save(flags) \
          asm volatile ("pushf ; pop %0" :"=g" (flags)
                        : /* no input */: "memory") \
          asm volatile("cli": : :"memory")

      #define local_irq_restore(flags) \
          asm volatile ("push %0 ; popf"
                        : /* no output */
                        : "g" (flags) :"memory", "cc");
```

割り込みは ``local_irq_disable()`` と ``local_irq_enable()`` マクロで明示的に有効にしたり無効にすることができますが、これらの API は現在の状態（APIを呼び出す時の状態）と何の割り込みなのかが分かっている場合にのみ使用して下さい。
これらは、通常は（割り込み処理といった）カーネル・コードのコア部で使用されます。

並列処理に伴う問題のために割り込みそのものを回避したいという典型的なケースでは、``local_irq_save()`` と ``local_irq_restore()`` 系の関数の使用が推奨されています。
これらは割り込みの状態を保存したり復元する関数ですが、これらの関数を正しく呼び出している限り **【訳注１】** 、クリティカル・セクションで作業している最中に誤って割り込みを有効にしてしまうといったリスクを犯すことなく、重複するクリティカル・セクションから自由に呼び出すことができます。

**【訳注１】**

保存（``local_irq_save()``）と復元（``local_irq_restore()``）の呼び出し回数がそれぞれ同じである状態


### スピン・ロック

「**スピン・ロック**（*Spin Lock*）」はクリティカル・セクションへのアクセスをシリアル化（*serialize*）する **【訳注２】** 際に使用します。

**【訳注２】**

順番付けする。クリティカル・セクションへのアクセスを順番ずつにする。

これは、真の並行処理が可能なマルチ・コアシステムで必要となるメカニズムです。
次が典型的なスピン・ロックの実装です：



```asm

      spin_lock:
          lock bts [my_lock], 0
	  jc spin_lock

      /* クリティカル・セクション */

      spin_unlock:
          mov [my_lock], 0
```

   **bts dts, src** - （*bit test and set*）この命令は ``dts`` のメモリ・アドレスから ``src`` ビットをキャリー・フラグ ``CF`` にコピーして、それをセットする

```c

      CF <- dts[src]
      dts[src] <- 1
```

ご覧のとおり、スピン・ロックはアトミック命令を使って、クリティカル・セクションに入ることができるコアは一つだけであることを保証します。
もし複数のコアがクリティカル・セクションに入ろうとしたら、ロックが解放されるまで、コアはそれこそぶっ続けに「スピン」し続けます。

   * 少なくとも１個のコアがクリティカル・セクションのロックに入ろうとするとロックの競合が発生する

   * ロックの競合は、クリティカル・セクションの規模、クリティカル・セクションで費やした時間, そしてシステム内のコア数とともに大きくなる

スピン・ロックにあるもう一つの（マイナス面の）副作用はキャッシュのスラッシング（*Cache Thrashing*）です。

キャッシュのスラッシングは、複数のコアが同じメモリを読み書きした結果、過度なキャッシュ・ミスが起こった場合に発生します。

スピン・ロックはロックの競合中にメモリに連続的にアクセスするため、「キャッシュ・コヒーレンス（*Cache Coherence*）」が実装されているが故にキャッシュ・スラッシングがよく発生します。


### マルチ・コアのシステムにおけるキャッシュ・コヒーレンス（*Cache Coherence*）

マルチ・コアのシステムにおける物理メモリはローカル CPU のキャッシュ（L1 キャッシュ）、共有 CPU のキャッシュ（L2 キャッシュ）、そしてメイン・メモリから構成されています。
ここでキャッシュ・コヒーレンスを説明するために L2 キャッシュを無視し、L1 キャッシュとメイン・メモリだけ考慮することにします。

下の図は、変数Aと変数Bがそれぞれ異なるキャッシュ・ラインに分類され、キャッシュとメイン・メモリが同期されているメモリの階層の状態を示しています：

![](images/Fig25-SynchronizedCachesMemory.png)

キャッシュとメイン・メモリの間で同期するメカニズムが無い場合、CPU0 が ``A = A + B`` を実行し、CPU1 が ``B = A + B`` をそれぞれ実行すると、上の状態は次のようになります：

![](images/Fig26-UnsynchronizedCachesMemory.png)

上の図のような状態になるのを回避するために、マルチ・コアのシステムはキャッシュ・コヒーレンスの仕組み（プロトコル）を使用します。
このプロトコルには主に二つの種類があります：

   * バス・スヌーピング（*Bus Snooping / Sniffing*）系：メモリ・バスのトランザクションがキャッシュによって監視され、一貫性（コヒーレンス）を維持するためのアクションを実行する

   * ディレクトリ系： キャッシュの状態を維持する別のディレクトリがある：
     キャッシュはディレクトリと相互に作用して一貫性（コヒーレンス）を維持する

バス・スヌーピングは他と比較して仕組みは単純ですが、コア数が 32〜64 を超えるとパフォーマンスが低下します。

ディレクトリ系のキャッシュ・コヒーレンスのプロトコルは他と比較してはるかに優れており（コア数は最大で数千個）、NUMA システムでも標準の機能です。

実際のところ、一般的に使用される軽量なキャッシュ・コヒーレンスのプロトコルは MESI です（この用語はキャッシュ・ラインの状態名の頭文字をつなげたものです：**M**odified、**E**xclusive、**S**hared、**I**nvalid）。
主な特徴は次のとおりです：

   * キャッシュする方針: 書き戻し（*Write Back*）

   * キャッシュ・ラインの状態

     * Modified（変更）: シングル・コアが所有し、データは変更済（*dirty*）の状態

     * Exclusive（排他）: シングル・コアが所有し、データは書き戻し済（*clean*）の状態

     * Shared（共有）: 複数のコアで共有中で、データは書き戻し済（*clean*）の状態

     * Invalid（無効）: ラインはキャッシュされていない

次に示した例のように、CPU コアからキャッシュの読み込みまたは書き込みを行う要求は、キャシュ・ラインの状態遷移に応じて発行されます：

   * Invalid -> Exclusive: 読み込みの要求が発行される（それ以外の全てのコアは Invalid なラインを持つ => ラインはメイン・メモリから読み込まれたデータになる）

   * Invalid -> Shared: 読み込みの要求が発行される（少なくとも一つのコアが Shared または Exclusive なラインを持つ => ラインは「兄弟（*sibling*）」キャッシュから読み込まれたデータになる）

   * Invalid/Shared/Exclusive -> Modified: 書き込みの要求が発行される（**それ以外の全ての** コアはラインを **無効にする**）

   * Modified -> Invalid: 他のコアから書き込みの要求が発行される（ラインはメイン・メモリに書き戻される）


---

##### Note

MESI プロトコルのもっとも重要な特徴は「書き込み無効化のキャッシュ・プロトコル（*Write-Invalidate Cache Protocol*）」の一つであるという点です。
任意の共有領域にデータを書き込むと、他の全てのキャッシュが無効になります。

---

これは、特定のアクセス・パタンではパフォーマンスに大きな影響を及ぼし、上で説明したような単純なスピン・ロックの実装でさえも競合が発生するパタンがあります。

このような問題を説明する例として、3つの CPU コアを持つシステムについて考えてみることにしましょう。
1つ目の CPU コアはスピン・ロックを獲得してクリティカル・セクションを実行していますが、他の2つの CPU コアはクリティカル・セクションに入るためにスピンしながら待っているという状況です：

![](images/Fig27-CacheThrashingBySpinLockContention.png)

上の図から分かるように、ロック獲得までスピンしている2つのコアによって発行された書き込みの要求のため、キャッシュラインを無効にする操作が頻繁に発生しています。
この状態は、基本的に2つのコアがロック獲得待ちの間、キャッシュ・ラインを書き出して（*Flush*）新しく読み込む（*Load*）ことでメモリ・バス上に不必要なトラフィックを発生させることになり、最終的に1つ目のコアのメモリ・アクセスの速度低下といった事態にまで影響します。

これとは別に、1つ目の CPU コアがクリティカル・セクションにいる間に最もよくアクセスされる可能性が高いデータが、獲得したロックと同じキャッシュ・ラインに格納されてしまう現象があります
（これはロックを獲得したあとにキャッシュの中にデータを準備しておくための一般的な最適化による副作用）。
これは、スピン中の他の2つのコアによって発動されたキャッシュの無効化で1つ目のコアのクリティカル・セクションの処理が遅くなり、事実上さらに多くのキャッシュラインの無効化が発動されてしまう問題につながります。

### スピン・ロックの最適化

既に紹介した単純なスピン・ロックの実装には CPU コアの数が増えるとキャッシュのスラッシングが原因でパフォーマンスが落ちる問題が発生する可能性がることが分かりました。
この問題を回避するため考えられる方法が次の二つのです：

* メモリへの書き戻しの数を減らし、それによってキャッシュの無効化の操作を減らす

* 他のプロセッサが同じキャッシュ・ラインでスピンしないようにし、それによってキャッシュの無効化の操作を減らす

前者の考え方に基づいて最適化されたスピン・ロックの実装が以下のとおりです：

```asm
      spin_lock:
          rep ; nop
          test lock_addr, 1
          jnz spin_lock
          lock bts lock_addr
          jc spin_lock
```

   * 最初にアトミックではない命令を使ってロックが読み取り専用かどうかをテストして、スピン中に発生する書き戻しとそれによるキャッシュ無効化の操作を回避する
   * ロックが解放されている「*可能性がある*」場合にのみロックの獲得を試す

また、この実装は **PAUSE** 命令を使って、（誤検出した）メモリのアクセス順序違反（*memory order violation*）によるメモリ・パイプラインのフラッシュを回避し、わずかな遅延（メモリ・バスの周波数に比例する）を追加して消費電力を抑えます。

Linux カーネルの多くのアーキテクチャでは（経過時間に基づいてクリティカル・セクション内で許可されたCPU コアによる）「公平性（*fairness*）」をサポートした似たような実装が使用されています（[Ticket Spin Lock](https://lwn.net/Articles/267968/) ）。

但し、x86 アーキテクチャの場合、現在のスピン・ロックの実装はキューに登録されたスピン・ロックを使用しています。
これは CPU コアがいろいろなロックをスピンさせてキャッシュの無効化操作を回避しています（可能ならば別のキャッシュ・ラインで分散させる）。

![](images/Fig28-QueuedSpinLocks.png)

概念的には新しい CPU コアがロックの獲得を試みて失敗すると、その CPUをコアのプライベートなロックを待機中の CPU コアのリストに追加します。
ロックの所有者がクリティカル・セッションを終了すると、その所有者は（必要であれば）リストにある次のロックを解除します。

スピンの読み込みが最適化されている間、スピン・ロックはキャシュの無効化操作の大部分を低減しますが、ロックがある場所に近いデータ構造、すなわち同じキャッシュ・ラインの一部への書き込みのため、ロックの所有者はキャッシュの無効化操作を依然として生成できてしまいます。
これにより、ロック獲得のためにスピン中の CPU コアで、次にメモリを読み込む際にトラフィックが発生することになります。

したがってキューに登録されたスピン・ロックは、NUMA システムの場合と同様に、たくさんの CPU コアに対してはるかに優れたスケーリングを実現します。
そして、このようなスピン・ロックは Ticket Spin Lock と同様に公平性に似た属性を持っているので、x86 アーキテクチャでは推奨されている実装です。


### プロセスと割り込みのコンテキストの同期

プロセスと割り込みコンテキストの両方から共有データをアクセスすることは、比較的一般的な事象です。
シングル・コアのシステムの場合は割り込みを無効にすることで実現できますが、1つ目の CPU コアでプロセスを実行し別の CPU コアで割り込みコンテキストを実行するするようなマルチ・コアのシステムで、この方法は通用しません。

マルチ・コア向けに設計されたスピン・ロックを使うことは妥当な方法に思えますが、次に示すシナリオで詳しく説明するように、この方法だと一般的なデッドロック状態を引き起こす可能性があります。：

   1. プロセスのコンテキストでスピン・ロックを獲得する

   1. 一個の割り込みが発生し、割り込みハンドラが同じ CPU コアでスケジューリングされる

   1. 割り込みハンドラが呼び出されてスピン・ロックの獲得を試みる

   1. 現在の CPU でデッドロックが発生する


この問題を回避するために、次の二つの方法を組み合わせて使います：

   * プロセスのコンテキスト: 割り込みを無効にしてスピン・ロックを獲得する。
     これで割り込みまたは他の CPU コアの競合状態から保護される（``spin_lock_irqsave()`` と ``spin_lock_restore()`` 関数で、これら二つの操作を実現する）

   * 割り込みのコンテキスト: スピン・ロックを獲得する。
     これで他の割り込みハンドラや他の CPU コアで実行されているプロセスのコンテキストから保護される

ソフト割り込み（*Soft IRQ*）やタスクレット（*Tasklet*）、あるいはタイマー割り込みいった他の割り込みハンドラでも、上記と同じ問題が発生します。ここでも割り込みを無効にすることで対応できますが専用の API の使用が推奨されています：

   * プロセスのコンテキストの中では ``spin_lock_bh()`` (これは ``local_bh_disable()`` と ``spin_lock()`` 関数を組み合わせたもの）関数と ``spin_unlock_bh()`` (これは ``spin_unlock()`` と ``local_bh_enable()`` 関数の組み合わせ）関数を使う

   * ボトム・ハーフのコンテキストの中では ``spin_lock()`` と ```spin_unlock()`` 関数を使う（あるいはデータを複数の割り込みハンドラとで共有する場合は ``spin_lock_irqsave()`` と ``spin_lock_irqrestore()`` 関数）


前述のように、Linux カーネルにおける並列性のもう一つの要因は、他のプロセスになることを可能にする、いわゆる「プリエンプション（*Preemption*）」です。

プリエンプションは有効と無効の切り替えが可能です：有効の時はレイテンシと応答時間が向上し、無効の時はスループットが向上します。

プリエンプションはスピン・ロックとミューテックスで無効になりますが、手動でも無効にすることができます（カーネルのソースコードから）

ローカルの割り込みを有効にしたり無効する API に関しては、ボトム・ハーフとプリエンプションの API を使ってクリティカル・セクションが重なるところで呼び出すことができます。
カウンタはボトム・ハーフとプリエンプションの状態を追跡するために使用します。
実際は同じカウンタを使用し、増分値がそれぞれ異なります：


```c

      #define PREEMPT_BITS      8
      #define SOFTIRQ_BITS      8
      #define HARDIRQ_BITS      4
      #define NMI_BITS          1

      #define preempt_disable() preempt_count_inc()

      #define local_bh_disable() add_preempt_count(SOFTIRQ_OFFSET)

      #define local_bh_enable() sub_preempt_count(SOFTIRQ_OFFSET)

      #define irq_count() (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK))

      #define in_interrupt() irq_count()

      asmlinkage void do_softirq(void)
      {
          if (in_interrupt()) return;
          ...
```

### ミューテックス（*Mutexes*）

Mutexes are used to protect against race conditions from other CPU cores but they can only be used in **process context**.
As opposed to spin locks, while a thread is waiting to enter the critical section it will not use CPU time, but instead it will be added to a waiting queue until the critical section is vacated.

Since mutexes and spin locks usage intersect, it is useful to compare the two:

   * They don't "waste" CPU cycles; system throughput is better than spin locks if context switch overhead is lower than medium spinning time

   * They can't be used in interrupt context

   * They have a higher latency than spin locks

Conceptually, the ``mutex_lock()`` operation is relatively simple: if the mutex is not acquired we an take the fast path via an atomic exchange operation:


```c
      void __sched mutex_lock(struct mutex *lock)
      {
        might_sleep();

        if (!__mutex_trylock_fast(lock))
          __mutex_lock_slowpath(lock);
      }

      static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
      {
        unsigned long curr = (unsigned long)current;

        if (!atomic_long_cmpxchg_acquire(&lock->owner, 0UL, curr))
          return true;

        return false;
      }

```

otherwise we take the slow path where we add ourselves to the mutex waiting list and put ourselves to sleep:

```c
      ...
        spin_lock(&lock->wait_lock);
      ...
        /* add waiting tasks to the end of the waitqueue (FIFO): */
        list_add_tail(&waiter.list, &lock->wait_list);
      ...
        waiter.task = current;
      ...
        for (;;) {
	  if (__mutex_trylock(lock))
	    goto acquired;
        ...
	  spin_unlock(&lock->wait_lock);
	...
          set_current_state(state);
      	  spin_lock(&lock->wait_lock);
        }
        spin_lock(&lock->wait_lock);
      acquired:
        __set_current_state(TASK_RUNNING);
        mutex_remove_waiter(lock, &waiter, current);
        spin_lock(&lock->wait_lock);
      ...
```
      
The full implementation is a bit more complex:
instead of going to sleep immediately it optimistic spinning if it detects that the lock owner is currently running on a different CPU as chances are the owner will release the lock soon.
It also checks for signals and handles mutex debugging for locking dependency engine debug feature.


The ``mutex_unlock()`` operation is symmetric:
if there are no waiters on the mutex then we an take the fast path via an atomic exchange operation:

```c
      void __sched mutex_unlock(struct mutex *lock)
      {
	if (__mutex_unlock_fast(lock))
	  return;
	__mutex_unlock_slowpath(lock, _RET_IP_);
      }

      static __always_inline bool __mutex_unlock_fast(struct mutex *lock)
      {
	unsigned long curr = (unsigned long)current;

	if (atomic_long_cmpxchg_release(&lock->owner, curr, 0UL) == curr)
	  return true;

	return false;
      }

      void __mutex_lock_slowpath(struct mutex *lock)
      {
      ...
        if (__mutex_waiter_is_first(lock, &waiter))
		__mutex_set_flag(lock, MUTEX_FLAG_WAITERS);
      ...

```
---

#### Note:: Because ``struct task_struct`` is cached aligned the 7 lower bits of the owner field can be used for various flags, such as ``MUTEX_FLAG_WAITERS``.

---

Otherwise we take the slow path where we pick up first waiter from the list and wake it up:

```c
      ...
      spin_lock(&lock->wait_lock);
      if (!list_empty(&lock->wait_list)) {
        /* get the first entry from the wait-list: */
        struct mutex_waiter *waiter;
        waiter = list_first_entry(&lock->wait_list, struct mutex_waiter,
                                  list);
	next = waiter->task;
	wake_q_add(&wake_q, next);
      }
      ...
      spin_unlock(&lock->wait_lock);
      ...
      wake_up_q(&wake_q);
```


Per CPU data
============

Per CPU data avoids race conditions by avoiding to use shared
data. Instead, an array sized to the maximum possible CPU cores is
used and each core will use its own array entry to read and write
data. This approach certainly has advantages:


.. slide:: Per CPU data
   :inline-contents: True
   :level: 2

   * No need to synchronize to access the data

   * No contention, no performance impact

   * Well suited for distributed processing where aggregation is only
     seldom necessary (e.g. statistics counters)


Memory Ordering and Barriers
============================

Modern processors and compilers employ out-of-order execution to
improve performance. For example, processors can execute "future"
instructions while waiting for current instruction data to be fetched
from memory.

Here is an example of out of order compiler generated code:

.. slide:: Out of Order Compiler Generated Code
   :inline-contents: True
   :level: 2

   +-------------------+-------------------------+
   | C code            | Compiler generated code |
   +-------------------+-------------------------+
   |.. code-block:: c  |.. code-block:: asm      |
   |		       |			 |
   |   a = 1;          |  MOV R10, 1		 |
   |   b = 2;          |  MOV R11, 2		 |
   |                   |  STORE R11, b		 |
   |                   |  STORE R10, a		 |
   +-------------------+-------------------------+


.. note:: When executing instructions out of order the processor makes
          sure that data dependency is observed, i.e. it won't execute
          instructions whose input depend on the output of a previous
          instruction that has not been executed.

In most cases out of order execution is not an issue. However, in
certain situations (e.g. communicating via shared memory between
processors or between processors and hardware) we must issue some
instructions before others even without data dependency between them.

For this purpose we can use barriers to order memory operations:

.. slide:: Barriers
   :inline-contents: True
   :level: 2

   * A read barrier (:c:func:`rmb()`, :c:func:`smp_rmb()`) is used to
     make sure that no read operation crosses the barrier; that is,
     all read operation before the barrier are complete before
     executing the first instruction after the barrier

   * A write barrier (:c:func:`wmb()`, :c:func:`smp_wmb()`) is used to
     make sure that no write operation crosses the barrier

   * A simple barrier (:c:func:`mb()`, :c:func:`smp_mb()`) is used
     to make sure that no write or read operation crosses the barrier


Read Copy Update (RCU)
======================

Read Copy Update is a special synchronization mechanism similar with
read-write locks but with significant improvements over it (and some
limitations):

.. slide:: Read Copy Update (RCU)
   :level: 2
   :inline-contents: True

   * **Read-only** lock-less access at the same time with write access

   * Write accesses still requires locks in order to avoid races
     between writers

   * Requires unidirectional traversal by readers


In fact, the read-write locks in the Linux kernel have been deprecated
and then removed, in favor of RCU.

Implementing RCU for a new data structure is difficult, but a few
common data structures (lists, queues, trees) do have RCU APIs that
can be used.

RCU splits removal updates to the data structures in two phases:

.. slide:: Removal and Reclamation
   :inline-contents: True
   :level: 2

   * **Removal**: removes references to elements. Some old readers may
     still see the old reference so we can't free the element.

   * **Elimination**: free the element. This action is postponed until
     all existing readers finish traversal (quiescent cycle). New
     readers won't affect the quiescent cycle.


As an example, lets take a look on how to delete an element from a
list using RCU:

![](images/Fig29-RcuListDelete.png)

         (1) List Traversal                          (2) Removal
                                                    +-----------+
      +-----+     +-----+     +-----+      +-----+  |  +-----+  |  +-----+
      |     |     |     |     |     |      |     |  |  |     |  |  |     |
      |  A  |---->|  B  |---->|  C  |      |  A  |--+  |  B  |--+->|  C  |
      |     |     |     |     |     |      |     |     |     |     |     |
      +-----+     +-----+     +-----+      +-----+     +-----+     +-----+
         ^           ^           ^            ^           ^           ^
         |           |           |            |           |           |







         (3) Quiescent cycle over                 (4) Reclamation
               +-----------+
      +-----+  |  +-----+  |  +-----+      +-----+                 +-----+
      |     |  |  |     |  |  |     |      |     |                 |     |
      |  A  |--+  |  B  |  +->|  C  |      |  A  |---------------->|  C  |
      |     |     |     |     |     |      |     |                 |     |
      +-----+     +-----+     +-----+      +-----+                 +-----+
         ^                       ^            ^                       ^
         |                       |            |                       |


In the first step it can be seen that while readers traverse the list
all elements are referenced. In step two a writer removes
element B. Reclamation is postponed since there are still readers that
hold references to it. In step three a quiescent cycle just expired
and it can be noticed that there are no more references to
element B. Other elements still have references from readers that
started the list traversal after the element was removed. In step 4 we
finally perform reclamation (free the element).


Now that we covered how RCU functions at the high level, lets looks at
the APIs for traversing the list as well as adding and removing an
element to the list:


.. slide:: RCU list APIs cheat sheet
   :inline-contents: True
   :level: 2

   .. code-block:: c

      /* list traversal */
      rcu_read_lock();
      list_for_each_entry_rcu(i, head) {
        /* no sleeping, blocking calls or context switch allowed */
      }
      rcu_read_unlock();


      /* list element delete  */
      spin_lock(&lock);
      list_del_rcu(&node->list);
      spin_unlock(&lock);
      synchronize_rcu();
      kfree(node);

      /* list element add  */
      spin_lock(&lock);
      list_add_rcu(head, &node->list);
      spin_unlock(&lock);

