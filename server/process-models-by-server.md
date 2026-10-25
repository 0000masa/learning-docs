# サーバーごとのプロセスとスレッドの使い方 — Spring Boot / Node.js / Laravel / Nginx / Apache

「Spring Boot、Node.js、Laravel、Nginx、Apacheは、1つのプロセスしか動かせないのか？」という疑問に答えるノート。プロセスとスレッドそのものの説明は → [process-and-thread.md](./process-and-thread.md)。結論を先に言うと：

- **どれも「1つのプロセスしか動かせない」わけではない。** 違うのは、たくさんのリクエストを**何で同時にさばくか**である。
- **Spring Bootは「1プロセス + 多数のスレッド」。** リクエスト1つにスレッド1本を割り当てる（1章）。
- **Node.jsは「1プロセス + JSを実行するスレッドは1本」。** 待ち時間の間に別のリクエストを進める、イベントループという方式（2章）。
- **Laravel（php-fpm）は「多数のプロセス」。** 1つのプロセスが1度に1リクエストを処理する。キューワーカーもスケジューラーも別のプロセス（3章）。
- **Nginxは「数個のプロセス × それぞれイベントループ」。** 少ないプロセスで、多数の接続をさばく（4章）。
- **Apacheは方式（MPM）を選べる。** プロセスだけのprefork、プロセス × スレッドのworkerとevent（5章）。

## 0. 3つの方式

たくさんのリクエストを同時にさばく方法は、大きく3つある。

| 方式 | 仕組み | 代表 |
|---|---|---|
| **マルチプロセス** | プロセスを何個も起動し、1プロセスで1リクエストを処理する | php-fpm、Apache prefork |
| **マルチスレッド** | 1つのプロセスの中にスレッドを何本も用意し、1スレッドで1リクエストを処理する | Spring Boot（Tomcat）、Apache worker / event |
| **イベントループ** | 少ないスレッドで、待ち時間（DBの応答、ネットワーク）の間に別のリクエストを進める | Node.js、Nginx |

マルチプロセスとマルチスレッドは「担当者を増やす」方式、イベントループは「1人の担当者が、待ち時間に別の仕事を進める」方式である。実際のサーバーは、これらを組み合わせていることが多い。

## 1. Spring Boot — 1プロセス + 多数のスレッド

```
[java プロセス 1 つ]
 ├─ http-nio-8080-Acceptor     接続を受け付ける
 ├─ http-nio-8080-Poller       届いたリクエストを、処理するスレッドに振り分ける
 ├─ http-nio-8080-exec-1〜N    リクエストを 1 つずつ処理する(既定で最大 200 本)
 └─ その他(受信ループ、Redis の通信、JVM 自身など)
```

- 組み込みのTomcatが、リクエストごとにスレッドを1本割り当てる。処理中にDBの応答を待っている間、そのスレッドは止まって待つ
- スレッド数の既定値は、最小10本（`server.tomcat.threads.min-spare`）、最大200本（`server.tomcat.threads.max`）。手元のSpring Boot 4.1.0のjarに入っている設定のメタデータで確かめた
- 実際に動いている様子は → [process-and-thread.md](./process-and-thread.md)の3章（`http-nio-8080-exec`が10本並んでいる）

### 増やし方

- **1つのプロセスの中**：スレッドの上限を上げる。ただしDBの接続プールの大きさなど、他の上限も合わせて考える必要がある
- **プロセスを増やす**：同じアプリを複数起動し、ロードバランサーで振り分ける。このリポジトリではECSのタスクを増やすことにあたる
- **役割で分ける**：同じソースから、役割の違うプロセスを起動する。このリポジトリの`backend`と`worker`がこれで、認証メールの送信をSQS経由で`worker`に任せている（→ [process-and-thread.md](./process-and-thread.md)の4章）

### 補足：仮想スレッド

Java 21には**仮想スレッド**という、とても軽いスレッドがある。Spring Bootでは`spring.threads.virtual.enabled=true`で使える。有効にすると、リクエストごとに仮想スレッドを作るので、上の200本という上限は効かなくなる（メタデータの説明にも「仮想スレッドが有効なら効果が無い」とある）。このリポジトリでは使っていない。

## 2. Node.js — 1プロセス + JSを実行するスレッドは1本

```
[node プロセス 1 つ]
 ├─ イベントループ(1 本)   JavaScript のコードはすべてここで実行される
 └─ ワーカープール(既定 4 本)  ファイルの読み書きなど、重い処理を裏で実行する
```

- **JavaScriptのコードを実行するスレッドは1本だけ**である（イベントループ）。DBへの問い合わせのような待ち時間のある処理は、結果を待たずに次のリクエストへ進み、結果が届いたら続きを実行する
- ファイルの読み書きなど一部の処理は、libuvのワーカープールが裏で実行する。既定は4本で、環境変数`UV_THREADPOOL_SIZE`で変えられる（最大1024）
- **弱点**：イベントループの中で重い計算をすると、その間は**全部のリクエストが止まる**。スレッドが1本しかないからである

Node.jsの公式ドキュメントは、これを「少ない数のスレッドで、たくさんのクライアントをさばく」と説明している。

### 増やし方

- **プロセスを増やす**：`cluster`モジュールやPM2のclusterモードで、CPUのコアの数だけnodeのプロセスを起動する。1本のイベントループでは1コアしか使えないため
- **重い計算だけ別スレッドにする**：`worker_threads`で、JSを実行するスレッドを別に作れる
- **コンテナを増やす**：コンテナ1つにnodeを1つ置き、コンテナの数で増やす

### このリポジトリでは

`nuxt`コンテナの`npm run dev`が、nodeのプロセスを1つ起動している（開発用のNuxtサーバー）。本番はNuxtをSSGでビルドしてSpring Bootから配信するので、nodeのプロセスは存在しない。

## 3. Laravel（php-fpm）— 多数のプロセス

PHPは、**自分ではHTTPを待ち受けない**。前段のNginxやApacheがリクエストを受け、php-fpmに渡す。

```
[nginx]  ──リクエスト──▶  [php-fpm master プロセス]
                           ├─ 子プロセス 1   1 度に 1 リクエストを処理する
                           ├─ 子プロセス 2
                           └─ …(既定の上限は 5 個)

[php artisan queue:work]   キューワーカー(別のプロセス。常駐する)
[cron → php artisan schedule:run]   スケジューラー(1 分ごとに起動して終わる)
```

- php-fpmは、masterプロセスが子プロセス（ワーカー）を何個も起動して管理する。**子プロセス1つが、1度に1リクエストを処理する**。スレッドは使わない
- 子プロセスの数は設定ファイル（プール）の`pm.max_children`で決まり、これが**同時に処理できるリクエスト数の上限**になる。php-fpmに同梱されている既定の設定（`www.conf`）では次のとおり

  | 設定 | 既定値 | 意味 |
  |---|---|---|
  | `pm` | `dynamic` | 空きの数を見ながら、子プロセスを増減させる |
  | `pm.max_children` | 5 | 子プロセスの上限 = 同時に処理できるリクエスト数の上限 |
  | `pm.start_servers` | 2 | 起動時に作る子プロセスの数 |
  | `pm.min_spare_servers` / `pm.max_spare_servers` | 1 / 3 | 待機中の子プロセスを、この範囲に保つ |

  `pm`には他に、数を固定する`static`と、リクエストが来てから作る`ondemand`がある。ディストリビューションのパッケージは独自の値にしていることがあるので、実際の値は`grep -E '^pm' /etc/php-fpm.d/www.conf`などで確かめる
- **キューワーカー**（`php artisan queue:work`）は、Webとは別の常駐プロセスである。Supervisorなどで、落ちたら起動し直すように管理する
- **スケジューラー**は、cronが1分ごとに`php artisan schedule:run`を起動し、処理して終わる

Laravelのアプリは、最初から**役割ごとに別々のプロセス**で動く形になっている。Spring Bootがアプリの中の`@Scheduled`や`@Async`で済ませることを、Laravelは別のプロセスで行う。素のLinuxでこれらのプロセスを動かす手順は別ノート（未作成）で扱う。

### なぜプロセスなのか

PHPは「1リクエストを処理したら全部捨てる」設計で、リクエストの間で何も共有しない（shared-nothing）。プロセスを分ければ、あるリクエストの状態が別のリクエストに漏れることがない。また、PHPの拡張モジュールには、複数のスレッドから同時に使うと壊れるものが多かったという歴史的な事情もある。

### 補足：Laravel Octane

Laravel Octane（Swoole、RoadRunner、FrankenPHP）を使うと、アプリを常駐させたまま、複数のリクエストを処理するようになる。Spring Bootに近い動き方になるが、そのぶん、リクエストをまたいで状態が残る問題に気をつける必要が出てくる。普通のPHPでは、`static`変数のように「一度だけ」作るものも1リクエストごとに捨てられるが、Octaneではリクエストをまたいで残る。

## 4. Nginx — 数個のプロセス × それぞれイベントループ

```
[nginx master プロセス]       設定を読む、ポートを開く、worker を管理する(root で動く)
 ├─ worker プロセス 1         多数の接続を 1 本のスレッドでさばく(一般ユーザーで動く)
 ├─ worker プロセス 2
 └─ …(worker_processes の数)
```

- masterプロセスは、設定を読み、80番や443番のポートを開き、workerを起動して管理する。実際のリクエストは処理しない
- workerプロセスは、それぞれが**イベントループ**で、多数の接続を同時にさばく。Node.jsと同じ考え方を、Cで書かれた高速な処理で行っている
- workerの数は`worker_processes`で決める。公式ドキュメントでは、既定値は`1`、迷ったらCPUのコア数（`auto`で自動検出）が良い出発点とされている。ディストリビューションの設定ファイルは`auto`にしていることが多い
- 設定を読み直す（`nginx -s reload`）と、masterが新しい設定でworkerを起動し直し、古いworkerは処理中の接続を終えてから止まる。接続を切らずに設定を変えられるのは、プロセスが分かれているからである

### このリポジトリでは

Nginxは使わない。Spring Bootの組み込みTomcatが直接HTTPを受け、本番のTLS終端はALBが担う。

## 5. Apache — 方式（MPM）を選べる

Apacheは、プロセスとスレッドの使い方を**MPM**（Multi-Processing Module）という部品で切り替えられる。

| MPM | 仕組み | 向いている場面 |
|---|---|---|
| **prefork** | プロセスだけ。1プロセスで1接続を処理する（php-fpmと同じ考え方） | スレッドで使えない古いモジュールがあるとき。昔の`mod_php`がこれを必要とした |
| **worker** | 複数のプロセス × それぞれ複数のスレッド。1スレッドで1接続を処理する | たくさんの接続をさばきたいとき |
| **event** | workerの改良版。何もしていない接続（keep-aliveで次のリクエストを待っている接続）を専用のスレッドに預け、処理するスレッドを空ける | 現在の標準 |

公式ドキュメントによると、Unix系のOSでの既定は、スレッドと`epoll` / `kqueue`が使えれば**event**、スレッドだけ使えればworker、どちらも無ければpreforkである。今のOSはどちらも使えるので、**実際にはほぼeventになる**。

PHPをApacheで動かす場合、昔は`mod_php`でApacheのプロセスの中にPHPを組み込み、preforkで動かすのが定番だった。今はphp-fpmに処理を渡し、Apacheはeventで動かすのが主流である。

## 6. 比較

| | 同時にさばく方法 | プロセス | スレッド | 同時に処理できる数を決める設定 |
|---|---|---|---|---|
| Spring Boot | マルチスレッド | 1 | 多数 | `server.tomcat.threads.max`（既定200） |
| Node.js | イベントループ | 1 | JSは1本（+ ワーカープール4本） | イベントループが空いているかどうか |
| Laravel（php-fpm） | マルチプロセス | 多数（+ キューワーカー、cron） | 各1本 | `pm.max_children`（同梱の既定5） |
| Nginx | マルチプロセス × イベントループ | master + 数個のworker | 各1本 | `worker_processes` × 1 workerあたりの接続数 |
| Apache | MPMで選ぶ | prefork・worker・eventとも複数 | preforkは各1本、worker / eventは多数 | MPMごとの設定 |

### どれも「プロセスを増やす」ことはできる

どの技術でも、プロセス（やコンテナ、サーバー）を増やしてロードバランサーで振り分ければ、処理できる量を増やせる。違うのは、**1つのプロセスの中で、どこまで同時にさばけるか**である。

- Spring Bootは、1つのプロセスの中でスレッドを増やせる
- Node.jsは、1つのプロセスではCPUを1コアしか使えないので、コアの数だけプロセスを増やすのが定番
- php-fpmは、最初からプロセスを増やす前提で作られている

## つまずきポイント

- **「サーバーは1プロセス」だと思う。** Nginxもphp-fpmも、起動した時点で複数のプロセスになる（3・4章）
- **Node.jsで重い計算をイベントループの中で行う。** スレッドが1本なので、全部のリクエストが止まる（2章）
- **php-fpmの`pm.max_children`を既定のまま本番に出す。** 同梱の設定のままだと、同時に5リクエストしか処理できない。メモリの量から逆算して決める（3章）
- **LaravelのキューワーカーがWebと同じプロセスで動くと思う。**`queue:work`は別の常駐プロセスで、落ちたときの起動し直しも自分で用意する（3章）
- **Spring Bootのスレッドの上限だけを上げる。** DBの接続プールが足りなければ、スレッドは接続の順番待ちで止まるだけになる（1章）
- **ApacheのMPMを気にせず`mod_php`を使う。**`mod_php`はスレッドのMPMと組み合わせにくい。今はphp-fpm + eventが主流（5章）

## 用語集

- **マルチプロセス** — プロセスを複数起動して、同時に処理する方式
- **マルチスレッド** — 1つのプロセスの中に複数のスレッドを用意して、同時に処理する方式
- **イベントループ** — 少ないスレッドで、待ち時間の間に別の処理を進める方式。Node.jsとNginxが使う
- **ワーカープール（スレッドプール）** — 仕事を受け取るために、あらかじめ用意しておくスレッドの集まり。Tomcatの`http-nio-8080-exec`や、Node.jsのlibuvのものがある
- **master / worker** — 管理だけするプロセス（master）と、実際に処理するプロセス（worker）の分担。Nginxとphp-fpmが使う
- **php-fpm** — PHPを常駐させ、Webサーバーから渡されたリクエストを処理するプロセスマネージャー（FastCGI Process Manager）
- **プール（php-fpm）** — php-fpmの子プロセスの集まりと、その設定（`www.conf`など）
- **MPM（Apache）** — Apacheのプロセスとスレッドの使い方を決める部品。prefork / worker / event
- **keep-alive** — 1つの接続で、複数のリクエストを続けて送れるようにするHTTPの仕組み。待っている間も接続は残る
- **仮想スレッド** — Java 21で入った、とても軽いスレッド。大量に作れる
- **Laravel Octane** — Laravelのアプリを常駐させて動かす仕組み
- **Supervisor** — 指定したプロセスを常駐させ、落ちたら起動し直すツール

## 出典

- Nginxの`worker_processes`（既定値`1`、`auto`の意味） → https://nginx.org/en/docs/ngx_core_module.html#worker_processes
- php-fpmの同梱設定（`pm` / `pm.max_children`などの既定値と説明） → https://github.com/php/php-src/blob/master/sapi/fpm/www.conf.in
- ApacheのMPMと既定の決まり方 → https://httpd.apache.org/docs/2.4/mpm.html
- Node.jsのイベントループとワーカープール → https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop
- libuvのスレッドプール（既定4、`UV_THREADPOOL_SIZE`、最大1024） → https://docs.libuv.org/en/v1.x/threadpool.html
- Spring BootのTomcatのスレッド数（`server.tomcat.threads.max` = 200、`min-spare` = 10） → 手元の`spring-boot-tomcat-4.1.0.jar`の`META-INF/spring-configuration-metadata.json`で確認

## 関連

- プロセスとスレッドとは何か、このリポジトリのコンテナの中での実測 → [process-and-thread.md](./process-and-thread.md)
