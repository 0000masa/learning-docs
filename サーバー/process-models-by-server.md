# サーバーごとのプロセスとスレッドの使い方 — Spring Boot / Node.js / Laravel / Nginx / Apache

「Spring Boot、Node.js、Laravel、Nginx、Apache は、1 つのプロセスしか動かせないのか?」という疑問に答える学習メモ。プロセスとスレッドそのものの説明は → [process-and-thread.md](./process-and-thread.md)。結論を先に言うと:

- **どれも「1 つのプロセスしか動かせない」わけではない。** 違うのは、たくさんのリクエストを**何で同時にさばくか**である。
- **Spring Boot は「1 プロセス + 多数のスレッド」。** リクエスト 1 つにスレッド 1 本を割り当てる(1 章)。
- **Node.js は「1 プロセス + JS を実行するスレッドは 1 本」。** 待ち時間の間に別のリクエストを進める、イベントループという方式(2 章)。
- **Laravel(php-fpm)は「多数のプロセス」。** 1 つのプロセスが 1 度に 1 リクエストを処理する。キューワーカーもスケジューラーも別のプロセス(3 章)。
- **Nginx は「数個のプロセス × それぞれイベントループ」。** 少ないプロセスで、多数の接続をさばく(4 章)。
- **Apache は方式(MPM)を選べる。** プロセスだけの prefork、プロセス × スレッドの worker と event(5 章)。

## 0. 3 つの方式

たくさんのリクエストを同時にさばく方法は、大きく 3 つある。

| 方式 | 仕組み | 代表 |
|---|---|---|
| **マルチプロセス** | プロセスを何個も起動し、1 プロセスで 1 リクエストを処理する | php-fpm、Apache prefork |
| **マルチスレッド** | 1 つのプロセスの中にスレッドを何本も用意し、1 スレッドで 1 リクエストを処理する | Spring Boot(Tomcat)、Apache worker / event |
| **イベントループ** | 少ないスレッドで、待ち時間(DB の応答、ネットワーク)の間に別のリクエストを進める | Node.js、Nginx |

マルチプロセスとマルチスレッドは「担当者を増やす」方式、イベントループは「1 人の担当者が、待ち時間に別の仕事を進める」方式である。実際のサーバーは、これらを組み合わせていることが多い。

## 1. Spring Boot — 1 プロセス + 多数のスレッド

```
[java プロセス 1 つ]
 ├─ http-nio-8080-Acceptor     接続を受け付ける
 ├─ http-nio-8080-Poller       届いたリクエストを、処理するスレッドに振り分ける
 ├─ http-nio-8080-exec-1〜N    リクエストを 1 つずつ処理する(既定で最大 200 本)
 └─ その他(受信ループ、Redis の通信、JVM 自身など)
```

- 組み込みの Tomcat が、リクエストごとにスレッドを 1 本割り当てる。処理中に DB の応答を待っている間、そのスレッドは止まって待つ
- スレッド数の既定値は、最小 10 本(`server.tomcat.threads.min-spare`)、最大 200 本(`server.tomcat.threads.max`)。手元の Spring Boot 4.1.0 の jar に入っている設定のメタデータで確かめた
- 実際に動いている様子は → [process-and-thread.md](./process-and-thread.md) の 3 章(`http-nio-8080-exec` が 10 本並んでいる)

### 増やし方

- **1 つのプロセスの中**:スレッドの上限を上げる。ただし DB の接続プールの大きさなど、他の上限も合わせて考える必要がある
- **プロセスを増やす**:同じアプリを複数起動し、ロードバランサーで振り分ける。このリポジトリでは ECS のタスクを増やすことにあたる
- **役割で分ける**:同じソースから、役割の違うプロセスを起動する。このリポジトリの `backend` と `worker` がこれ(→ [../../adr/0017-auth-mail-via-sqs-worker.md](../../adr/0017-auth-mail-via-sqs-worker.md))

### 補足:仮想スレッド

Java 21 には**仮想スレッド**という、とても軽いスレッドがある。Spring Boot では `spring.threads.virtual.enabled=true` で使える。有効にすると、リクエストごとに仮想スレッドを作るので、上の 200 本という上限は効かなくなる(メタデータの説明にも「仮想スレッドが有効なら効果が無い」とある)。このリポジトリでは使っていない。

## 2. Node.js — 1 プロセス + JS を実行するスレッドは 1 本

```
[node プロセス 1 つ]
 ├─ イベントループ(1 本)   JavaScript のコードはすべてここで実行される
 └─ ワーカープール(既定 4 本)  ファイルの読み書きなど、重い処理を裏で実行する
```

- **JavaScript のコードを実行するスレッドは 1 本だけ**である(イベントループ)。DB への問い合わせのような待ち時間のある処理は、結果を待たずに次のリクエストへ進み、結果が届いたら続きを実行する
- ファイルの読み書きなど一部の処理は、libuv のワーカープールが裏で実行する。既定は 4 本で、環境変数 `UV_THREADPOOL_SIZE` で変えられる(最大 1024)
- **弱点**:イベントループの中で重い計算をすると、その間は**全部のリクエストが止まる**。スレッドが 1 本しかないからである

Node.js の公式ドキュメントは、これを「少ない数のスレッドで、たくさんのクライアントをさばく」と説明している。

### 増やし方

- **プロセスを増やす**:`cluster` モジュールや PM2 の cluster モードで、CPU のコアの数だけ node のプロセスを起動する。1 本のイベントループでは 1 コアしか使えないため
- **重い計算だけ別スレッドにする**:`worker_threads` で、JS を実行するスレッドを別に作れる
- **コンテナを増やす**:コンテナ 1 つに node を 1 つ置き、コンテナの数で増やす

### このリポジトリでは

`nuxt` コンテナの `npm run dev` が、node のプロセスを 1 つ起動している(開発用の Nuxt サーバー)。本番は Nuxt を SSG でビルドして Spring Boot から配信するので、node のプロセスは存在しない(→ CLAUDE.md の「アーキテクチャ上の重要な決定事項」)。

## 3. Laravel(php-fpm)— 多数のプロセス

PHP は、**自分では HTTP を待ち受けない**。前段の Nginx や Apache がリクエストを受け、php-fpm に渡す。

```
[nginx]  ──リクエスト──▶  [php-fpm master プロセス]
                           ├─ 子プロセス 1   1 度に 1 リクエストを処理する
                           ├─ 子プロセス 2
                           └─ …(既定の上限は 5 個)

[php artisan queue:work]   キューワーカー(別のプロセス。常駐する)
[cron → php artisan schedule:run]   スケジューラー(1 分ごとに起動して終わる)
```

- php-fpm は、master プロセスが子プロセス(ワーカー)を何個も起動して管理する。**子プロセス 1 つが、1 度に 1 リクエストを処理する**。スレッドは使わない
- 子プロセスの数は設定ファイル(プール)の `pm.max_children` で決まり、これが**同時に処理できるリクエスト数の上限**になる。php-fpm に同梱されている既定の設定(`www.conf`)では次のとおり

  | 設定 | 既定値 | 意味 |
  |---|---|---|
  | `pm` | `dynamic` | 空きの数を見ながら、子プロセスを増減させる |
  | `pm.max_children` | 5 | 子プロセスの上限 = 同時に処理できるリクエスト数の上限 |
  | `pm.start_servers` | 2 | 起動時に作る子プロセスの数 |
  | `pm.min_spare_servers` / `pm.max_spare_servers` | 1 / 3 | 待機中の子プロセスを、この範囲に保つ |

  `pm` には他に、数を固定する `static` と、リクエストが来てから作る `ondemand` がある。ディストリビューションのパッケージは独自の値にしていることがあるので、実際の値は `grep -E '^pm' /etc/php-fpm.d/www.conf` などで確かめる
- **キューワーカー**(`php artisan queue:work`)は、Web とは別の常駐プロセスである。Supervisor などで、落ちたら起動し直すように管理する
- **スケジューラー**は、cron が 1 分ごとに `php artisan schedule:run` を起動し、処理して終わる

Laravel のアプリは、最初から**役割ごとに別々のプロセス**で動く形になっている。Spring Boot がアプリの中の `@Scheduled` や `@Async` で済ませることを、Laravel は別のプロセスで行う(→ [../deploying-to-bare-linux.md](../deploying-to-bare-linux.md) の 1-2)。

### なぜプロセスなのか

PHP は「1 リクエストを処理したら全部捨てる」設計で、リクエストの間で何も共有しない(shared-nothing)。プロセスを分ければ、あるリクエストの状態が別のリクエストに漏れることがない。また、PHP の拡張モジュールには、複数のスレッドから同時に使うと壊れるものが多かったという歴史的な事情もある。

### 補足:Laravel Octane

Laravel Octane(Swoole、RoadRunner、FrankenPHP)を使うと、アプリを常駐させたまま、複数のリクエストを処理するようになる。Spring Boot に近い動き方になるが、そのぶん、リクエストをまたいで状態が残る問題に気をつける必要が出てくる(→ [../java/syntax/class-vs-instance-initialization.md](../java/syntax/class-vs-instance-initialization.md) の 8 章)。

## 4. Nginx — 数個のプロセス × それぞれイベントループ

```
[nginx master プロセス]       設定を読む、ポートを開く、worker を管理する(root で動く)
 ├─ worker プロセス 1         多数の接続を 1 本のスレッドでさばく(一般ユーザーで動く)
 ├─ worker プロセス 2
 └─ …(worker_processes の数)
```

- master プロセスは、設定を読み、80 番や 443 番のポートを開き、worker を起動して管理する。実際のリクエストは処理しない
- worker プロセスは、それぞれが**イベントループ**で、多数の接続を同時にさばく。Node.js と同じ考え方を、C で書かれた高速な処理で行っている
- worker の数は `worker_processes` で決める。公式ドキュメントでは、既定値は `1`、迷ったら CPU のコア数(`auto` で自動検出)が良い出発点とされている。ディストリビューションの設定ファイルは `auto` にしていることが多い
- 設定を読み直す(`nginx -s reload`)と、master が新しい設定で worker を起動し直し、古い worker は処理中の接続を終えてから止まる。接続を切らずに設定を変えられるのは、プロセスが分かれているからである

### このリポジトリでは

Nginx は使わない。Spring Boot の組み込み Tomcat が直接 HTTP を受け、本番の TLS 終端は ALB が担う(→ CLAUDE.md の決定事項 1)。

## 5. Apache — 方式(MPM)を選べる

Apache は、プロセスとスレッドの使い方を **MPM(Multi-Processing Module)** という部品で切り替えられる。

| MPM | 仕組み | 向いている場面 |
|---|---|---|
| **prefork** | プロセスだけ。1 プロセスで 1 接続を処理する(php-fpm と同じ考え方) | スレッドで使えない古いモジュールがあるとき。昔の `mod_php` がこれを必要とした |
| **worker** | 複数のプロセス × それぞれ複数のスレッド。1 スレッドで 1 接続を処理する | たくさんの接続をさばきたいとき |
| **event** | worker の改良版。何もしていない接続(keep-alive で次のリクエストを待っている接続)を専用のスレッドに預け、処理するスレッドを空ける | 現在の標準 |

公式ドキュメントによると、Unix 系の OS での既定は、スレッドと `epoll` / `kqueue` が使えれば **event**、スレッドだけ使えれば worker、どちらも無ければ prefork である。今の OS はどちらも使えるので、**実際にはほぼ event になる**。

PHP を Apache で動かす場合、昔は `mod_php` で Apache のプロセスの中に PHP を組み込み、prefork で動かすのが定番だった。今は php-fpm に処理を渡し、Apache は event で動かすのが主流である。

## 6. 比較

| | 同時にさばく方法 | プロセス | スレッド | 同時に処理できる数を決める設定 |
|---|---|---|---|---|
| Spring Boot | マルチスレッド | 1 | 多数 | `server.tomcat.threads.max`(既定 200) |
| Node.js | イベントループ | 1 | JS は 1 本(+ ワーカープール 4 本) | イベントループが空いているかどうか |
| Laravel(php-fpm) | マルチプロセス | 多数(+ キューワーカー、cron) | 各 1 本 | `pm.max_children`(同梱の既定 5) |
| Nginx | マルチプロセス × イベントループ | master + 数個の worker | 各 1 本 | `worker_processes` × 1 worker あたりの接続数 |
| Apache | MPM で選ぶ | prefork・worker・event とも複数 | prefork は各 1 本、worker / event は多数 | MPM ごとの設定 |

### どれも「プロセスを増やす」ことはできる

どの技術でも、プロセス(やコンテナ、サーバー)を増やしてロードバランサーで振り分ければ、処理できる量を増やせる。違うのは、**1 つのプロセスの中で、どこまで同時にさばけるか**である。

- Spring Boot は、1 つのプロセスの中でスレッドを増やせる
- Node.js は、1 つのプロセスでは CPU を 1 コアしか使えないので、コアの数だけプロセスを増やすのが定番
- php-fpm は、最初からプロセスを増やす前提で作られている

## つまずきポイント

- **「サーバーは 1 プロセス」だと思う。** Nginx も php-fpm も、起動した時点で複数のプロセスになる(3・4 章)
- **Node.js で重い計算をイベントループの中で行う。** スレッドが 1 本なので、全部のリクエストが止まる(2 章)
- **php-fpm の `pm.max_children` を既定のまま本番に出す。** 同梱の設定のままだと、同時に 5 リクエストしか処理できない。メモリの量から逆算して決める(3 章)
- **Laravel のキューワーカーが Web と同じプロセスで動くと思う。** `queue:work` は別の常駐プロセスで、落ちたときの起動し直しも自分で用意する(3 章)
- **Spring Boot のスレッドの上限だけを上げる。** DB の接続プールが足りなければ、スレッドは接続の順番待ちで止まるだけになる(1 章)
- **Apache の MPM を気にせず `mod_php` を使う。** `mod_php` はスレッドの MPM と組み合わせにくい。今は php-fpm + event が主流(5 章)

## 用語集

- **マルチプロセス** — プロセスを複数起動して、同時に処理する方式
- **マルチスレッド** — 1 つのプロセスの中に複数のスレッドを用意して、同時に処理する方式
- **イベントループ** — 少ないスレッドで、待ち時間の間に別の処理を進める方式。Node.js と Nginx が使う
- **ワーカープール(スレッドプール)** — 仕事を受け取るために、あらかじめ用意しておくスレッドの集まり。Tomcat の `http-nio-8080-exec` や、Node.js の libuv のものがある
- **master / worker** — 管理だけするプロセス(master)と、実際に処理するプロセス(worker)の分担。Nginx と php-fpm が使う
- **php-fpm** — PHP を常駐させ、Web サーバーから渡されたリクエストを処理するプロセスマネージャー(FastCGI Process Manager)
- **プール(php-fpm)** — php-fpm の子プロセスの集まりと、その設定(`www.conf` など)
- **MPM(Apache)** — Apache のプロセスとスレッドの使い方を決める部品。prefork / worker / event
- **keep-alive** — 1 つの接続で、複数のリクエストを続けて送れるようにする HTTP の仕組み。待っている間も接続は残る
- **仮想スレッド** — Java 21 で入った、とても軽いスレッド。大量に作れる
- **Laravel Octane** — Laravel のアプリを常駐させて動かす仕組み
- **Supervisor** — 指定したプロセスを常駐させ、落ちたら起動し直すツール

## 出典

- Nginx の `worker_processes`(既定値 `1`、`auto` の意味) → https://nginx.org/en/docs/ngx_core_module.html#worker_processes
- php-fpm の同梱設定(`pm` / `pm.max_children` などの既定値と説明) → https://github.com/php/php-src/blob/master/sapi/fpm/www.conf.in
- Apache の MPM と既定の決まり方 → https://httpd.apache.org/docs/2.4/mpm.html
- Node.js のイベントループとワーカープール → https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop
- libuv のスレッドプール(既定 4、`UV_THREADPOOL_SIZE`、最大 1024) → https://docs.libuv.org/en/v1.x/threadpool.html
- Spring Boot の Tomcat のスレッド数(`server.tomcat.threads.max` = 200、`min-spare` = 10) → 手元の `spring-boot-tomcat-4.1.0.jar` の `META-INF/spring-configuration-metadata.json` で確認

## 関連

- プロセスとスレッドとは何か、このリポジトリのコンテナの中での実測 → [process-and-thread.md](./process-and-thread.md)
- 素の Linux で Spring Boot と Nginx + php-fpm を動かす手順(常駐プロセスの数の違い) → [../deploying-to-bare-linux.md](../deploying-to-bare-linux.md)
- PHP の「一度だけ」の単位はリクエスト、Laravel Octane で変わること → [../java/syntax/class-vs-instance-initialization.md](../java/syntax/class-vs-instance-initialization.md) の 8 章
- Spring と Laravel のイベントとキュー(Laravel のキューワーカー) → [../java/spring/application-events-vs-queues.md](../java/spring/application-events-vs-queues.md)
