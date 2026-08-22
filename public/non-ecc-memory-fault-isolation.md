---
title: PC のメモリが物理的に壊れた!? 非 ECC 機で不良箇所を特定して隔離した話
tags:
  - Linux
  - メモリ
  - ECC
  - ハードウェア
  - トラブルシューティング
private: false
updated_at: '2026-08-22T21:09:52+09:00'
id: 3d46e17c57d9adaa9968
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

自宅の Linux マシンで、メモリを大量に使う処理を長時間動かしていたところ、ある日突然、処理が頻繁にクラッシュするようになりました。

コードも入力データも変えていないのに、落ちる場所やエラーの内容は毎回バラバラです。最初はソフトウェア側のバグを疑い、入力データや並列処理、ライブラリなどを調べ続けました。

しかし、原因を追っていった結果、問題はコードではなく 物理メモリの一部が壊れていること だと分かりました。

さらに厄介だったのは、このマシンが ECC メモリを搭載していなかったことです。OS のログを調べてもメモリエラーらしい記録は残っておらず、ハードウェア障害だと気づくまでにかなり時間がかかりました。

この記事では、そのときに行った調査と対処をまとめます。

- 症状からハードウェア障害を疑うまで
- 自作のメモリテストで異常を再現し、不良な物理アドレスを特定するまで
- 問題のある物理メモリ領域を Linux カーネルから使われないように隔離するまで
- memmap= の指定を間違えて起動不能になり、物理的に復旧する羽目になった失敗

同じような不可解なクラッシュに遭遇したときの切り分け方として、今回の経験を残しておきます。

## 入力のせいではなかった

まず気づいたのは、プロセスの死に方が毎回違うことでした。

```
attempt 1  native crash (exit -11)   SIGSEGV
attempt 2  native crash (exit -11)   SIGSEGV
attempt 3  native crash (exit  -6)   corrupted size vs. prev_size
attempt 4  ValueError: Overflow when unpacking long long
```

`corrupted size vs. prev_size` は、glibc が malloc の管理情報の破壊を検出したときのメッセージです。はじめは「どこかから異常な値が入ってきている」と考え、その値を配列の添字に使う手前で範囲チェックを入れてみました。

すると、検査そのものが落ちました。

```
ValueError: Exceeds the limit (4300 digits) for integer string conversion
```

壊れた値をログに出そうとしたら、その数が 10 進で 4,300 桁を超えていた、というのです（Python 3.11 以降は巨大整数の文字列変換に上限があります）。

私のコードが添字に書き込むのは、高々 1 万程度の小さな整数だけです。4,300 桁の整数を作る経路はどこにもありません。つまり、正しく書き込んだはずの小さな整数が、あとからメモリ上で書き換わっていた、ということです。

CPython の整数は「桁数」と「桁の配列」で表現されます。桁数の部分が別の書き込みに踏まれると、中身は小さいままなのに「4,300 桁ある」と主張するオブジェクトになる。

検証すべき「不正な入力」が存在せず、ハードウェアを疑うに至りました。

- コードを 1 行も変えていない
- 同じ条件が、通ったり落ちたりする
- 壊れ方が毎回違う（コードのバグなら、壊れ方は再現するはず）

毎回違う場所で違う壊れ方をするのは、ランダムなメモリ破壊の特徴でした。

## 非 ECC では「エラーが出ない」＝「正常」ではない

ここで少し、ECC メモリの話をします。

ECC（Error Correcting Code）メモリは、データに誤り検出・訂正用の情報を余分に持たせたメモリです。ビットが化けても、1 ビットの誤りなら自動で訂正し、その回数を記録してくれます。サーバ機でよく使われます。

非 ECC メモリには、この仕組みがありません。ビットが化けても、検出も訂正も記録もされず、壊れた値がそのままアプリケーションに渡ります。

自分のマシンがどちらかは、次で分かります。

```sh
ls /sys/devices/system/edac/mc/
```

```
power  subsystem  uevent
```

EDAC は Linux のメモリエラー報告の仕組みで、ECC 対応機ならここにエラーカウンタが並びます。空ということは、私のマシンは非 ECC でした。

そして、これが今回の肝でした。非 ECC では、次の推論が成り立ちません。

> カーネルのエラーログが空 → メモリは正常

記録する機能がそもそも無いだけで、「異常が記録されていない」は「異常がない」を意味しません。壊れていても、OS は何も教えてくれないのです。

## メモリテストで不良を確定する

メモリが本当に壊れているかを、簡単なテストプログラムで確かめます。やることは、メモリ一面に決まった数列を書き込み、それを読み戻して、書いたときと同じ値かを照合するだけです。

数列には xorshift64 という、短い式で次々に乱数を作る擬似乱数生成器を使いました。同じ種（seed）を与えれば、毎回まったく同じ数列を再現できます。

これが嬉しいのは、答え合わせ用に「書いたはずの値」を別のバッファに取っておかなくて済むからです。読み戻すときに種から数列を作り直せば、期待値はその場で分かります。メモリを二重に持たなくてよいので、同じ搭載量でもテストできる範囲が倍になります。

コード全体は次のとおりです（`sudo` は不要です）。

```c
/* メモリの書き込み/読み戻しを検証し、不良領域の有無を確認する。
 *
 *   gcc -O2 -pthread -o memtest memtest.c
 *   ./memtest [スレッド数] [1スレッドあたりMB]      既定: 1 7200
 *
 * 「不一致 0 件」なら、その確保量では不良領域に触れていない。
 */
#include <pthread.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

static int threads = 1;
static int mb_per_thread = 7200;
static const int passes = 6;

static volatile long errors = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

/* xorshift64。同じ seed から同じ列を再生成できるので、書いた値と読んだ値を
   バッファを 2 つ持たずに比較できる。 */
static inline uint64_t next(uint64_t v) {
    v ^= v << 13;
    v ^= v >> 7;
    v ^= v << 17;
    return v;
}

static void *worker(void *arg) {
    long id = (long)arg;
    size_t n = (size_t)mb_per_thread * 1024 * 1024 / sizeof(uint64_t);
    uint64_t *buf = malloc(n * sizeof(uint64_t));
    if (!buf) {
        fprintf(stderr, "  thread %ld: %dMB の確保に失敗\n", id, mb_per_thread);
        return NULL;
    }
    for (int pass = 0; pass < passes; pass++) {
        uint64_t seed = 0x9E3779B97F4A7C15ULL * (pass + 1) + id;
        uint64_t v = seed;
        for (size_t i = 0; i < n; i++) buf[i] = (v = next(v));
        v = seed;
        for (size_t i = 0; i < n; i++) {
            v = next(v);
            if (buf[i] == v) continue;
            pthread_mutex_lock(&lock);
            if (++errors <= 5)
                fprintf(stderr, "  不一致 thread=%ld pass=%d offset=%zu 期待=%llx 実際=%llx\n",
                        id, pass, i, (unsigned long long)v, (unsigned long long)buf[i]);
            pthread_mutex_unlock(&lock);
        }
    }
    free(buf);
    return NULL;
}

int main(int argc, char **argv) {
    if (argc > 1) threads = atoi(argv[1]);
    if (argc > 2) mb_per_thread = atoi(argv[2]);
    pthread_t *t = malloc(sizeof(pthread_t) * threads);
    if (!t) return 2;
    printf("%dスレッド x %dMB x %dパス (合計 %.1f GB を読み書き)\n", threads, mb_per_thread,
           passes, (double)threads * mb_per_thread * passes / 1024.0);
    fflush(stdout);
    time_t start = time(NULL);
    for (long i = 0; i < threads; i++) pthread_create(&t[i], NULL, worker, (void *)i);
    for (int i = 0; i < threads; i++) pthread_join(t[i], NULL);
    printf("完了: %ld秒  不一致 %ld 件\n", (long)(time(NULL) - start), errors);
    free(t);
    return errors ? 1 : 0;
}
```

これをコンパイルして、8 スレッド・各 900MB で走らせます。

```sh
gcc -O2 -pthread -o memtest memtest.c
./memtest 8 900        # 8スレッド x 900MB
```

```
8スレッド x 900MB x 6パス (合計 42.2 GB を読み書き)
完了: 3秒  不一致 1531 件
```

1,531 件の不一致。メモリが実際に壊れていると分かりました。

8 スレッドのときだけ不一致が出たので「マルチスレッドが原因」と思ったのですが、スレッド数を増やすと総メモリ量も一緒に増えていました。2 つの変数を分離できていなかったのです。分けて測ると、こうなりました。

```
2スレッド x 3600MB (計 7200MB)  → 1531件
8スレッド x  900MB (計 7200MB)  → 1532件
1スレッド x 4200MB              → 0件
1スレッド x 4800MB              → 1529件
```

効いていたのはスレッド数ではなく、総メモリ量だけ。確保したメモリはすでに使われている分の上に積まれていくので、確保量が増えてその壊れた位置に届くと必ず踏む、というだけの話でした。しかも、書いた値によらず特定のビットが決まった値に張り付く「stuck bit」の挙動で、再起動しても同じ物理アドレスに出ます。物理的な欠陥です。

（なお「メモリ使用量を減らせば回避できる」とはなりません。踏む確率が下がるだけで、いつか必ず踏みます。）

## 不良ページを特定して隔離する

壊れているのが固定の場所なら、その物理アドレスを突き止めれば、そこだけ使わないようにできます。

Linux では `/proc/self/pagemap` を読むと、プログラムが見ている仮想アドレスが、実際の物理メモリのどこに載っているかを調べられます。これを使って、memtest で不一致が出たアドレスの物理位置を求めるプログラム（`phys`）を書きました。

```c
/* 不良メモリ領域の物理アドレスを特定する。
 *
 * /proc/self/pagemap から物理ページ番号を読むため、ホスト側で root として
 * 実行すること。コンテナ内や一般ユーザでは、ページ番号が 0 にマスクされて
 * 物理アドレスが得られない。
 *
 *   gcc -O2 -o phys phys.c
 *   sudo ./phys [MB]        既定: 7200
 */
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define PAGEMAP_PRESENT (1ULL << 63)
#define PAGEMAP_PFN_MASK ((1ULL << 55) - 1)

/* 仮想アドレスに対応する物理アドレス。読めない場合は 0。 */
static uint64_t physical_address(int pagemap_fd, void *addr) {
    long page_size = sysconf(_SC_PAGESIZE);
    uint64_t entry = 0;
    uint64_t virtual_page = (uint64_t)addr / page_size;
    if (pread(pagemap_fd, &entry, sizeof(entry), virtual_page * sizeof(entry)) != sizeof(entry))
        return 0;
    if (!(entry & PAGEMAP_PRESENT)) return 0;
    return (entry & PAGEMAP_PFN_MASK) * page_size + (uint64_t)addr % page_size;
}

static inline uint64_t next(uint64_t v) {
    v ^= v << 13;
    v ^= v >> 7;
    v ^= v << 17;
    return v;
}

int main(int argc, char **argv) {
    int mb = argc > 1 ? atoi(argv[1]) : 7200;
    size_t n = (size_t)mb * 1024 * 1024 / sizeof(uint64_t);
    uint64_t *buf = malloc(n * sizeof(uint64_t));
    if (!buf) {
        printf("%dMB の確保に失敗\n", mb);
        return 2;
    }
    int pagemap_fd = open("/proc/self/pagemap", O_RDONLY);
    if (pagemap_fd < 0) {
        printf("/proc/self/pagemap を開けません。root で実行してください\n");
        return 3;
    }

    const uint64_t seed = 0x9E3779B97F4A7C15ULL;
    uint64_t v = seed;
    for (size_t i = 0; i < n; i++) buf[i] = (v = next(v));

    /* 物理アドレスは連続しないので、ページ単位で最小・最大を集計する。 */
    long page_size = sysconf(_SC_PAGESIZE);
    uint64_t lowest = 0, highest = 0;
    long count = 0, shown = 0;
    v = seed;
    for (size_t i = 0; i < n; i++) {
        v = next(v);
        if (buf[i] == v) continue;
        uint64_t physical = physical_address(pagemap_fd, &buf[i]);
        if (!physical) {
            printf("物理アドレスを読めません(権限不足)。ホスト側で root として実行してください\n");
            return 3;
        }
        if (!count || physical < lowest) lowest = physical;
        if (physical > highest) highest = physical;
        count++;
        if (shown++ < 5)
            printf("  仮想 %p -> 物理 0x%llx\n", (void *)&buf[i], (unsigned long long)physical);
    }
    close(pagemap_fd);

    if (!count) {
        printf("%dMB: 不一致なし(この確保量では不良領域に触れていない)\n", mb);
        free(buf);
        return 0;
    }

    uint64_t first_page = lowest & ~(uint64_t)(page_size - 1);
    uint64_t last_page = highest & ~(uint64_t)(page_size - 1);
    uint64_t span_kb = (last_page - first_page + page_size) / 1024;
    printf("\n不一致 %ld 件\n", count);
    printf("物理アドレス範囲: 0x%llx 〜 0x%llx\n", (unsigned long long)lowest,
           (unsigned long long)highest);
    printf("該当ページ: 0x%llx から %llu KB (%llu ページ)\n", (unsigned long long)first_page,
           (unsigned long long)span_kb, (unsigned long long)(span_kb * 1024 / page_size));
    free(buf);
    return 1;
}
```

物理アドレスの読み取りには root 権限が要るので、コンパイルして `sudo` で実行します。

```sh
gcc -O2 -o phys phys.c
sudo ./phys 7200
```

```
不一致 254 件
物理アドレス範囲: 0x381270620 〜 0x381277df0
該当ページ: 0x381270000 から 32 KB (8 ページ)
```

壊れていたのは `0x381270000` から 32KB、4KB ページで 8 枚分でした。物理メモリの先頭から約 14.0 GiB の位置で、再起動して測り直しても同じアドレスに出ます。物理的に固定した不良で確定です。

Linux には、特定の物理ページをカーネルに使わせなくする仕組みがあり、しかも再起動が要りません。

```sh
for a in 0 1 2 3 4 5 6 7; do
  printf '0x38127%d000\n' $a | sudo tee /sys/devices/system/memory/soft_offline_page
done
```

`soft_offline_page` に物理アドレスを書き込むと、カーネルはそのページの中身を別の場所へ移し、以後そのページを割り当てなくなります。動いているプロセスを止めずに、メモリを 1 枚そっと抜くイメージです。

隔離してから、もう一度テストします。

```sh
./memtest 1 7200
```

```
完了: 16秒  不一致 0 件
```

不一致は 0 件。あれだけ落ちていた処理も、32KB を外しただけで、長時間まわしてもクラッシュしなくなりました。

## memmap= で恒久化しようとして起動不能にした

ただし `soft_offline_page` の設定は揮発性で、再起動すると元に戻ります。恒久化するには、カーネル起動パラメータ `memmap=` で「この物理領域は最初から無いことにする」と指定する方法が定番で、`soft_offline_page` より確実です。書式は `memmap=<サイズ>$<開始アドレス>` です。

不良は多少広がる可能性もあるので、32KB を 1MB に丸め、次のように `/etc/default/grub` に書きました。

```sh
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash memmap=1M\$0x381200000"
```

`update-grub` して再起動したところ、起動しなくなりました。

原因は `$` のエスケープでした。`/etc/default/grub` は `update-grub` のときにシェルとして読まれるので、`$0x...` がシェルの変数展開に巻き込まれます。それは知っていたのでバックスラッシュでエスケープしたのですが、実は GRUB 自身の設定でも `$` は変数扱いされます。結果、生成された `grub.cfg` の中で `$0x...` が未定義変数として空文字に消え、カーネルに渡ったのは次だけになりました。

```
memmap=1M
```

`$` の無い `memmap=1M` は、まったく別の意味になります。「使えるメモリの総量を 1MB に制限する」という指定です。メモリ 1MB では、当然 OS は起動できません。

厄介なのは、ここからの復旧に画面が要ることです。私の GRUB はメニューを隠す設定だったので、次の手順で無理やり入りました。

1. 電源投入直後に `Esc` を連打（UEFI の場合）して GRUB メニューを出す
2. `Ubuntu` を選んで `e`（起動パラメータの一時編集）
3. `memmap=1M` を消す
4. `Ctrl+X` で起動

起動できたらすぐ `/etc/default/grub` から該当行を消し、`update-grub` し直します。

結局 `memmap=` は諦め、今は起動時に `soft_offline_page` を自動で叩く systemd の oneshot サービスを置いて、再起動のたびに問題の 32KB を隔離し直すようにしました。

## おわりに

コードを変えていないのに落ち続けた原因は、物理メモリの一部が壊れていたことでした。自作のメモリテストで不良を再現し、物理アドレスを特定して、その領域を `soft_offline_page` でカーネルから隔離することで、クラッシュは止まりました。

非 ECC のマシンでは、メモリが壊れても OS は基本的に何も教えてくれません。原因不明のクラッシュが続くときは、ハードウェアも候補に入れておくとよさそうです。
