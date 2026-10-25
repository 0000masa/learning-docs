# プロセスとスレッド — 「メモリを分けるか、共有するか」の違い

「プロセスとはそもそも何か? タスクマネージャーで見えて、PID で検索できるもの? Spring Boot は 1 つのプロセスの中で `http-nio-8080-exec-…` や `sqs-receiver-auth-mail` のスレッドを動かしているが、これらを別々のプロセスに分けることはできないのか? サーバーは 1 つのプロセスしか動かせないのか?」という疑問に答える学習メモ。結論を先に言うと:

- **プロセスは「実行中のプログラム 1 つ」。** OS が 1 つずつに PID を振って管理する。タスクマネージャーの「詳細」タブや、Linux の `ps` で見えるもの。**プロセスごとにメモリが分けられている**(1 章)。
- **スレッドは「プロセスの中で、コードを順に実行していく流れ」。** 1 つのプロセスの中に何本も持てて、**同じプロセスのスレッド同士はメモリを共有する**(2 章)。
- **スレッドを、そのまま別のプロセスに移すことはできない。** スレッドが同じプロセスにいるのは、オブジェクトを共有するためだから。ただし**役割を別のプロセスに分ける**ことはできる。このリポジトリの `backend` と `worker` がそれで、2 つのプロセスはメモリを共有できないので、依頼を SQS 経由で渡している(4 章)。
- **サーバーは 1 つのプロセスしか動かせない、ということはない。** OS の上では普段から数百のプロセスが動いている。「1 つ」になりがちなのはコンテナのほうで、**1 コンテナにつき主なプロセスは 1 つ**という慣習がある(5 章)。

技術ごとのプロセスとスレッドの使い方(Spring Boot / Node.js / Laravel / Nginx / Apache)は別ノートにまとめた → [process-models-by-server.md](./process-models-by-server.md)

親プロセスと子プロセス、タスクマネージャーで親子関係を見る方法も別ノートにまとめた → [parent-and-child-process.md](./parent-and-child-process.md)

## 0. どこで出てきたか

[AuthMailQueueConfig.java](../../../backend/src/main/java/com/example/app/auth/AuthMailQueueConfig.java) の `exit` メソッドは、`new Thread(...)` で `auth-mail-exit` というスレッドを作り、その中で Spring と JVM を終了させている。受信ループの [SqsReceiveLoop.java](../../../backend/src/main/java/com/example/app/common/queue/SqsReceiveLoop.java) も、`sqs-receiver-auth-mail` という専用のスレッドで動いている。ここから「スレッドとは何か」「プロセスとはどう違うのか」という疑問が出た。

## 1. プロセスとは

**実行中のプログラム 1 つ**のこと。ディスクにある `java` や `node` のファイルは「プログラム」で、それを起動して動いている状態のものが「プロセス」。同じプログラムを 2 回起動すれば、プロセスは 2 つになる。

OS はプロセスごとに次のものを用意する。

| OS が用意するもの | 意味 |
|---|---|
| **PID**(プロセス ID) | プロセスを見分ける番号。起動するたびに振られる |
| **専用のメモリ空間** | そのプロセスだけが読み書きできるメモリ。**別のプロセスのメモリは直接読めない** |
| 実行ユーザー | どの権限で動いているか(→ [../os-users-and-file-locations.md](../os-users-and-file-locations.md)) |
| 開いているファイルや通信 | ファイル、ネットワークの接続など |
| 環境変数 | 起動時に渡された変数(→ [../env-vars-basics.md](../env-vars-basics.md)) |

### どこで見えるか

| 環境 | 見る方法 |
|---|---|
| Windows | タスクマネージャーの「詳細」タブ。PID の列がある。列の見出しを右クリックして「列の選択」から「スレッド」を足すと、プロセスごとのスレッド数も見える |
| Linux / Mac | `ps -ef`(一覧)、`top`(使用率の多い順に更新し続ける) |
| コンテナ | `docker compose exec <サービス> ps -ef`(コンテナの中)、`docker compose top`(ホストから見た一覧) |

## 2. スレッドとは

**プロセスの中で、コードを上から順に実行していく流れ**のこと。1 つのプロセスは、少なくとも 1 本のスレッドを持つ。必要なら何本でも足せる。

例えるなら、プロセスが**厨房**、スレッドがそこで働く**料理人**である。

- 料理人(スレッド)は、それぞれ別の料理(処理)を同時に進める
- 冷蔵庫(メモリ上のオブジェクト)は、同じ厨房の料理人全員で共有する
- 別の厨房(別のプロセス)の冷蔵庫には、手が届かない

### プロセスとスレッドの違い

| | プロセス | スレッド |
|---|---|---|
| メモリ | **プロセスごとに別々** | **同じプロセスのスレッド同士で共有** |
| データの受け渡し | 通信が要る(ネットワーク、ファイル、キューなど) | 同じオブジェクトを読み書きするだけ |
| 1 つが落ちたとき | 他のプロセスは無事 | **同じプロセスごと落ちることがある**(メモリ不足など) |
| 作る手間 | 重い(メモリ空間を丸ごと用意する) | 軽い |
| 同時に書き換えたときの事故 | 起きない(共有していないので) | 起きる。`volatile` や `synchronized` で守る必要がある |

最後の行が、[SqsReceiveLoop.java](../../../backend/src/main/java/com/example/app/common/queue/SqsReceiveLoop.java) で `running` フィールドに `volatile` を付けたり、`synchronized (lock)` で囲んだりしている理由である。受信スレッドと終了処理のスレッドが、同じ `running` を読み書きするからだ。

### なぜスレッドを分けるのか

同時に進めたい仕事があるからである。受信ループは `receiveMessage` で最大 20 秒待つことを、ずっと繰り返す。これを起動処理と同じスレッドで実行すると、ループから抜けられず、起動がいつまでも終わらない。専用のスレッドに任せれば、起動処理は先に進み、受信は裏で続く。

HTTP も同じで、Tomcat はリクエストごとに別のスレッドを割り当てる。1 本のスレッドで順番に処理すると、遅いリクエストが 1 つあるだけで、後ろのリクエストが全部待たされる。

## 3. 実測 — このリポジトリのコンテナの中を見る

開発環境の `worker` コンテナの中で `ps` を実行した(行は一部省略。`NLWP` はそのプロセスのスレッド数)。

```
$ docker compose exec worker ps -eo pid,ppid,nlwp,args
    PID    PPID NLWP COMMAND
      1       0   25 java ... -jar /app/gradle/wrapper/gradle-wrapper.jar bootRun --project-cache-dir /tmp/gradle-project-cache
     41       1   83 java ... (Gradle デーモン)
    138      41   57 java -XX:TieredStopAtLevel=1 ... -cp /app/build/classes/java/main:...
```

`jcmd -l` で、それぞれの Java プロセスが何かを確かめた。

```
$ docker compose exec worker jcmd -l
1 /app/gradle/wrapper/gradle-wrapper.jar bootRun --project-cache-dir /tmp/gradle-project-cache
41 org.gradle.launcher.daemon.bootstrap.GradleDaemon 9.5.1
138 com.example.app.Application
```

### 1 つのコンテナに Java のプロセスが 3 つある

開発環境では、アプリを `./gradlew bootRun` で起動している(`docker/backend/Dockerfile` の `CMD`)。そのため、3 つのプロセスが親子の関係で並ぶ。`PPID` は親プロセスの PID である(親子関係そのものは → [parent-and-child-process.md](./parent-and-child-process.md))。

```
PID 1    gradlew(Gradle を起動するだけの小さな Java)
└─ PID 41   Gradle デーモン(ビルドを実行する)
   └─ PID 138  アプリ本体(com.example.app.Application)
```

本番のイメージ(`docker/app/Dockerfile`)は `exec java -jar /app/app.jar` で jar を直接起動するので、Java のプロセスはアプリ本体の 1 つだけになる。`exec` は、`sh` のプロセスを `java` に置き換える命令で、`sh` が親として残らない。

### 1 つのプロセスの中のスレッド

アプリ本体(PID 138)のスレッドを `jstack` で取り出し、名前ごとに数えた(主なものだけ)。

```
$ docker compose exec worker sh -c "jstack 138 | grep '^\"'"
```

| スレッド名 | 本数 | 何をしているか |
|---|---|---|
| `http-nio-8080-exec-N` | 10 | Tomcat の、HTTP リクエストを処理するスレッド |
| `http-nio-8080-Acceptor` / `-Poller` | 1 / 1 | Tomcat の、接続を受け付けて処理するスレッドに振り分けるスレッド |
| `sqs-receiver-auth-mail` | 1 | 認証メールの受信ループ([SqsReceiveLoop.java](../../../backend/src/main/java/com/example/app/common/queue/SqsReceiveLoop.java) の `start()` で作られる) |
| `sdk-ScheduledExecutor-N` | 5 | AWS SDK が、通信のタイムアウトなどを見張るスレッド |
| `lettuce-nioEventLoop-N` | 2 | Redis の通信(Spring Session) |
| `GC Thread#N` など | 多数 | JVM 自身のスレッド(ガベージコレクション、コンパイルなど) |
| `DestroyJavaVM` | 1 | 後述 |

`/proc/138/status` で見たスレッド数は 58 本だった。1 つのプロセスの中に、これだけのスレッドが同時に存在している。

`http-nio-8080-exec` が 10 本なのは、Spring Boot の既定値(`server.tomcat.threads.min-spare` = 10)のとおり。混んでくると最大 200 本(`server.tomcat.threads.max`)まで増える。

### `backend` には受信ループのスレッドが無い

同じことを `backend` コンテナのアプリ本体で行うと、`sqs-receiver-auth-mail` が**無い**。

| スレッド | `backend`(`APP_ROLE` 未設定 = web) | `worker`(`APP_ROLE=worker`) |
|---|---|---|
| `http-nio-8080-exec-N` | 10 | 10 |
| `sqs-receiver-auth-mail` | **無い** | 1 |

同じソースから起動していても、`APP_ROLE=worker` のときだけ `AuthMailQueueConfig` が読み込まれ、受信ループのスレッドが作られる。設定の違いが、スレッドの有無として見えている。

### `main` スレッドはもう無い

一覧に `main` という名前のスレッドは無い。`main` スレッドは `main()` メソッドを実行するスレッドで、Spring Boot では `SpringApplication.run(...)` が起動を終えて戻ると、`main()` も終わり、`main` スレッドも終わる。

それでもプロセスが終わらないのは、Tomcat などの**デーモンでない**スレッドが残っているからである。JVM は、デーモンでないスレッドが 1 本でも動いている間は終了しない。`DestroyJavaVM` は、`main` が終わった後、それらが全部終わるのを待って JVM を片付けるためのスレッドである。

### ホストから見ると、PID が違う

ホスト側から `docker compose top` で見ると、同じプロセスに別の PID が付いている。

```
$ docker compose top backend worker
SERVICE  PID    PPID   CMD
backend  1430   1407   java ... gradle-wrapper.jar bootRun
backend  1669   1430   java ... (Gradle デーモン)
backend  2103   1669   java ... (アプリ本体)
worker   1537   1481   java ... gradle-wrapper.jar bootRun --project-cache-dir /tmp/gradle-project-cache
worker   1715   1537   java ... (Gradle デーモン)
worker   2072   1715   java ... (アプリ本体)
```

コンテナは、中のプロセスに**別の PID の番号体系**(PID 名前空間)を見せている。コンテナの中では自分が PID 1 から始まるように見えるが、ホストから見れば、どれも 1 台の OS の上で動く普通のプロセスである。`backend` のアプリ本体(ホストの PID 2103)と `worker` のアプリ本体(ホストの PID 2072)は、**別々のプロセス**だと分かる。

## 4. スレッドを別のプロセスに分けられるか

### スレッドをそのまま移すことはできない

スレッドが同じプロセスにいるのは、**オブジェクトを共有するため**である。たとえば `sqs-receiver-auth-mail` スレッドは、Spring が作った `AuthMailWorker` や `SqsClient` の Bean を使って処理している。これらの Bean は、そのプロセスのメモリの中にある。別のプロセスに移ったら、そのメモリには手が届かない。

### 役割を別のプロセスに分けることはできる

できる。分けた側では、必要なオブジェクトを**自分のプロセスで作り直す**ことになる。このリポジトリが、まさにそうしている。

```
[backend プロセス]                        [worker プロセス]
 http-nio-8080-exec-N                      sqs-receiver-auth-mail
 (登録を受ける)                             (認証メールを送る)
   │                                           ▲
   │ メモリは共有できないので、                    │
   └──── JSON にして SQS に送る ──── SQS ────────┘
```

- `backend` と `worker` は**同じソースから起動した、別々の JVM プロセス**。Bean はそれぞれのプロセスで 1 セットずつ作られる
- プロセス同士はメモリを共有できないので、依頼は JSON の文字列にして、プロセスの外(SQS)を経由して渡す。受け取った側は、JSON から `AuthMailRequestedEvent` を作り直す

同じプロセスの中なら、オブジェクトをそのまま渡せば済む。フェーズ20 より前は、実際にそうしていた(→ [../java/spring/application-events-vs-queues.md](../java/spring/application-events-vs-queues.md))。

### 分けると何が良くなるか

| | 1 つのプロセス | 別々のプロセス |
|---|---|---|
| 片方の障害 | メモリ不足などで、プロセスごと両方止まる | もう片方は動き続ける |
| 片方だけ止める・増やす | できない | できる(`docker compose stop worker`、ECS でワーカーだけ台数を増やす) |
| データの受け渡し | オブジェクトを渡すだけ | JSON にして、外部(SQS)を経由する必要がある |
| 二重に必要になるもの | 無い | DB の接続プール、メモリなどがプロセスの数だけ要る |

設計としての判断は → [../../adr/0017-auth-mail-via-sqs-worker.md](../../adr/0017-auth-mail-via-sqs-worker.md)

## 5. サーバーは 1 つのプロセスしか動かせないのか

### OS は数百のプロセスを同時に動かす

1 台のサーバー(OS)の上では、普段から多数のプロセスが動いている。Linux なら、`systemd`(他のプロセスを起動・管理する親玉)、`sshd`(SSH の受付)、ログの収集などである。アプリのプロセスは、その中の 1 つにすぎない。`ps -ef | wc -l` で数えると、何もしていないサーバーでも数十〜数百の行が出る。

CPU のコアが 4 つなら、ある瞬間に**本当に同時に**実行できるのは 4 本のスレッドまでである。それ以上のスレッドは、OS がごく短い時間で切り替えながら順に実行していて、人間には同時に動いているように見える。

この 4 本は、**同じプロセスのスレッドでなくてよい。** OS が CPU に載せる単位はスレッドで、どのプロセスのスレッドをどのコアで動かすかは、OS(スケジューラ)が決める。ある瞬間を切り取ると、たとえば次のようになっている。

```
コア 0: backend プロセスの http-nio-8080-exec-3
コア 1: worker プロセスの sqs-receiver-auth-mail
コア 2: MySQL プロセスのスレッド
コア 3: sshd プロセスのスレッド
```

次の瞬間には、それぞれのコアが別のスレッドに切り替わっている。別のプロセスのスレッドに切り替えるときは、メモリ空間も切り替えるので、同じプロセスのスレッド同士で切り替えるより少し重い。

### 「4 コア 8 スレッド」の CPU

ハイパースレッディング(SMT)のある CPU は、「4 コア 8 スレッド」のように書かれる。ここで言う「スレッド」は、ハードウェア側のスレッド(**論理プロセッサ**)のことで、この章で扱ってきたソフトウェアのスレッドとは別物である。論理プロセッサは、ソフトウェアのスレッドを載せる**座席**にあたる。OS は、数千本あるソフトウェアのスレッドを、この座席に順番に載せて実行する。Windows のタスクマネージャーの「パフォーマンス」タブにある「論理プロセッサ数」が、座席の数である。

SMT は、コアの中の**計算する部品(演算器など)は 1 組のまま**、**どこまで実行したかの記録(レジスタなど)を 2 組にした**ものである。1 本のスレッドがメモリからデータが届くのを待っている間、計算する部品は遊んでしまう。その隙間に、もう 1 本の命令をハードウェアが流し込む。OS が数ミリ秒ごとに入れ替えるのとは違い、ハードウェアが 1 サイクル単位で 2 本の命令を混ぜて流すので、2 本は本当に同時に実行中である。

ただし、計算する部品は 2 本で取り合うので、速さは 2 倍にはならない。一般に 1.2〜1.3 倍程度で、メモリ待ちの多い処理ほど伸び、計算ばかりする処理ではほとんど伸びない。まとめると、4 コア 8 スレッドの CPU は次のようになる。

- **同時に実行中のスレッド**: 8 本
- **計算する力**: 4 コアぶん + α(8 コアぶんではない)

### コンテナは「主なプロセス 1 つ」が慣習

1 つのコンテナでは、主なプロセスを 1 つだけ動かすのが Docker の慣習である(コンテナの中の PID 1)。PID 1 が終わると、コンテナも止まる。

ただしこれは慣習で、制限ではない。3 章で見たとおり、開発用の `backend` コンテナの中には Java のプロセスが 3 つある。VS Code の Dev Container で接続すると、VS Code のサーバーのプロセスも中に増える。本番のイメージが `exec java -jar ...` でアプリ本体だけを PID 1 にしているのは、ECS の停止の合図(SIGTERM)を、アプリが直接受け取れるようにするためでもある。

### ECS でも同じ慣習

ECS も、コンテナの中の **PID 1 しか見ていない**。

| ECS がすること | 見ている相手 |
|---|---|
| コンテナが止まったか判断する(止まればタスクを入れ替える) | PID 1 が終了したかどうか |
| 停止の合図 SIGTERM を送る | PID 1 だけ。`stopTimeout`(既定 30 秒)待っても終わらなければ SIGKILL |
| ログを CloudWatch Logs に送る(`awslogs`) | コンテナの標準出力・標準エラー |

そのため、1 つのコンテナで子プロセスを 2 つ動かすと困ったことが起きる。片方が落ちても、PID 1 が生きていれば ECS は気づかない。SIGTERM も、子プロセスまで届くとは限らない。

複数のプロセスを一緒に動かしたいときは、プロセスごとに別のコンテナにして、1 つのタスク定義の `ContainerDefinitions` に並べる。これを**サイドカー**と呼ぶ。ログ転送の FireLens(Fluent Bit)、CloudWatch エージェント、Service Connect のプロキシ(Envoy。ECS が自動で足す)などがこの形である。同じタスクのコンテナ同士は、`awsvpc` なら `localhost` で通信でき、ボリュームも共有できる。`Essential: true` を付けたコンテナが止まると、タスクごと止まる。

このリポジトリのタスク定義(`cloudformation/app.yml` の `app` / `migrate` / `mysql`)は、どれもコンテナが 1 つだけである。

### ECS でも、1 つのコンテナに複数のプロセスが動くことはある

慣習は「**主な**プロセスを 1 つにする」ことで、補助のプロセスまで禁じてはいない。

| 例 | 中身 |
|---|---|
| ECS Exec | `EnableExecuteCommand: true` にすると、ECS が SSM エージェントをコンテナに差し込み、**コンテナの中のプロセスとして**動かす。`aws ecs execute-command` で入ったシェルも、そのコンテナの中のプロセスになる |
| シェルスクリプトから起動する | `sh start.sh` の中で `exec` を付けずに `java` を起動すると、`sh` が PID 1 のまま残り、`java` はその子プロセスになる。SIGTERM が `java` に届かない原因になる |
| `initProcessEnabled` | 小さな init プロセスが PID 1 になり、アプリはその子プロセスになる(次の節) |
| supervisord で nginx + php-fpm をまとめる | 昔の Laravel のイメージでよく見る形。動くが、片方が落ちても ECS が気づかない |

このリポジトリでは、web サービスに `EnableExecuteCommand: true` が付いている(`cloudformation/app.yml` の `Service`)。そのため本番の `app` コンテナの中では、アプリの `java`(PID 1)のほかに、SSM エージェントのプロセスも動いている。

### `initProcessEnabled` — PID 1 の仕事を init に任せる

タスク定義のコンテナに付ける設定で、`docker run --init` と同じものである。有効にすると、コンテナの中で **tini という小さな init プロセスが PID 1 になり**、`ENTRYPOINT` のプロセス(このリポジトリなら `java`)はその子プロセスになる。

```
無効(このリポジトリの今)        有効
PID 1  java                      PID 1  /sbin/docker-init(tini)
                                 └─ PID 7  java
```

PID 1 は、Linux では特別扱いされるプロセスで、普通のプロセスには無い仕事が 2 つある。tini は、この 2 つを肩代わりするためだけのプロセスである。

| PID 1 の仕事 | 起きる問題 | tini がすること |
|---|---|---|
| **シグナルを自分で処理する** | PID 1 には、シグナルを受けたときの既定の動き(SIGTERM なら終了)が働かない。処理を書いていないプログラムが PID 1 だと、SIGTERM を無視し、30 秒後に SIGKILL で強制終了される | 受けたシグナルを子プロセスに中継する |
| **親を失ったプロセスを引き取って片付ける** | 親が先に終わったプロセスは、PID 1 の子に付け替えられる。PID 1 がその終了を受け取らない(`wait` しない)と、終わったプロセスの記録が**ゾンビ**として残り続ける | 引き取ったプロセスの終了を受け取り、ゾンビを片付ける |

1 つ目は、このリポジトリでは問題にならない。JVM は SIGTERM の処理を自分で持っていて、受け取ると Spring の終了処理(graceful shutdown)が走る。

2 つ目は、ECS Exec を使うと起きうる。Exec で入ったシェルや、SSM エージェントが作った子プロセスが親を失うと、PID 1 の `java` の子になる。JVM はそれを片付けないので、ゾンビが溜まっていく。AWS が ECS Exec と一緒に `initProcessEnabled` を有効にするよう勧めているのは、このためである。

CloudFormation では、コンテナ定義に次のように書く。このリポジトリでは、まだ設定していない。

```yaml
ContainerDefinitions:
  - Name: app
    LinuxParameters:
      InitProcessEnabled: true
```

### 技術ごとに、プロセスとスレッドの使い方が違う

「1 つのプロセスで多数のスレッドを使う」のは Spring Boot の形で、すべてのサーバーがそうではない。Nginx や php-fpm は、最初から**複数のプロセス**を起動して動く。比較は別ノートで扱う → [process-models-by-server.md](./process-models-by-server.md)

## つまずきポイント

- **「スレッドを別のプロセスに移せばよい」と考える。** スレッドは、同じプロセスのメモリを共有するために同じ場所にいる。分けるなら、役割ごとプロセスを分け、データはプロセスの外を経由して渡す(4 章)
- **「サーバーは 1 つのプロセスしか動かせない」と考える。** OS の上では多数のプロセスが動く。1 つになりがちなのはコンテナの慣習(5 章)
- **「CPU は 1 つのプロセスしか動かせず、そのスレッドをコアの数だけ同時に動かす」と考える。** コアに載る単位はスレッドで、プロセスは問わない。4 コアなら、4 つの別々のプロセスのスレッドが同時に動くこともある(5 章)
- **ECS で、1 つのコンテナに複数の主なプロセスを入れる。** ECS は PID 1 しか見ないので、子プロセスが落ちても気づかない。プロセスごとにコンテナを分け、同じタスクに並べる(5 章)
- **PID 1 を普通のプロセスと同じだと思う。** シグナルの既定の動きが働かず、ゾンビの片付けも任される。自分で処理しないプログラムを PID 1 にするなら、`initProcessEnabled` で init に任せる(5 章)
- **コンテナの中の PID とホストの PID を同じものだと思う。** コンテナは別の番号体系を見せている。同じプロセスでも番号が違う(3 章)
- **`main` スレッドが終わったら、アプリも終わると思う。** デーモンでないスレッドが残っている間、JVM は終わらない(3 章)
- **スレッド同士で共有している値を、何も守らずに読み書きする。** 古い値を読み続けたり、書き込みがぶつかったりする。`volatile` や `synchronized` で守る(2 章)
- **開発環境のプロセス構成を本番と同じだと思う。** 開発は `gradlew bootRun` なので Java が 3 つ、本番は `java -jar` なので 1 つ(3 章)

## 用語集

- **プログラム** — ディスクに置かれた実行ファイル。動いていない状態
- **プロセス** — 実行中のプログラム 1 つ。OS が PID とメモリ空間を割り当てる
- **PID(プロセス ID)** — OS がプロセスに振る番号
- **PPID** — 親プロセスの PID。そのプロセスを起動したプロセス
- **スレッド** — プロセスの中で、コードを順に実行していく流れ。同じプロセスのスレッド同士はメモリを共有する
- **コア** — CPU の中で、実際に命令を実行する単位。計算する部品を 1 組ずつ持つ
- **論理プロセッサ(ハードウェアのスレッド)** — ソフトウェアのスレッドを載せる座席。SMT のある CPU では 1 コアに 2 つある
- **SMT(ハイパースレッディング)** — 1 つのコアに記録(レジスタなど)を 2 組持たせ、2 本のスレッドを同時に実行させる仕組み。計算する部品は共有するので、速さは 2 倍にならない
- **メモリ空間** — プロセスごとに分けられたメモリの範囲。別のプロセスからは直接読み書きできない
- **デーモンスレッド** — JVM の終了を引き止めないスレッド。デーモンでないスレッドが残っている間、JVM は終了しない
- **PID 名前空間** — コンテナなどに、別の PID の番号体系を見せる Linux の仕組み
- **サイドカー** — 主なコンテナの補助として、同じタスク(Pod)に並べるコンテナ。ログ転送やプロキシなど
- **`Essential`** — ECS のコンテナ定義の設定。`true` のコンテナが止まると、タスクごと止まる
- **ECS Exec** — 動いているコンテナの中でコマンドを実行する ECS の機能。SSM エージェントがコンテナの中で動く
- **init プロセス / tini** — PID 1 として動き、シグナルの中継とゾンビの片付けだけをする小さなプログラム。ECS では `initProcessEnabled`、Docker では `--init` で使う
- **ゾンビプロセス** — 終了したのに、親がその終了を受け取らないため、記録だけが残っているプロセス
- **`ps` / `top`** — プロセスの一覧を表示するコマンド。`top` は使用率の多い順に更新し続ける
- **`jcmd` / `jstack`** — JDK 付属のコマンド。動いている JVM の一覧、スレッドの一覧(スレッドダンプ)を表示する
- **NLWP** — `ps` の列の 1 つ。そのプロセスのスレッド数(Number of Light-Weight Processes)

## 関連

- 親プロセスと子プロセス、親子関係の見方、親が先に終わったときの扱い → [parent-and-child-process.md](./parent-and-child-process.md)
- 技術ごとのプロセスとスレッドの使い方(Spring Boot / Node.js / Laravel / Nginx / Apache) → [process-models-by-server.md](./process-models-by-server.md)
- プロセスとユーザー、タスクマネージャーの「ユーザー名」列 → [../os-users-and-file-locations.md](../os-users-and-file-locations.md)
- 素の Linux で、Spring Boot と Nginx + php-fpm の常駐プロセスがどう違うか → [../deploying-to-bare-linux.md](../deploying-to-bare-linux.md)
- 同じプロセス内のイベントと、プロセスの外のキューの違い → [../java/spring/application-events-vs-queues.md](../java/spring/application-events-vs-queues.md)
- `docker compose exec` で何が起きているか → [../docker-exec-and-shells.md](../docker-exec-and-shells.md)
- 今回の題材のコード → [AuthMailQueueConfig.java](../../../backend/src/main/java/com/example/app/auth/AuthMailQueueConfig.java) と [SqsReceiveLoop.java](../../../backend/src/main/java/com/example/app/common/queue/SqsReceiveLoop.java)
