# 第8章: 分析・データ転送

## 座学

## 全体の導入（なぜデータ分析が必要なのか）

> 「今回は**分析サービスとデータ転送サービス**がテーマです。サービスの話に入る前に、**なぜ企業がデータ分析にお金と時間をかけるのか**を考えてみましょう。
>
> たとえば皆さんがECサイトを運営しているとします。毎日大量のアクセスログが溜まっていきますよね。このログを分析すると、"金曜の夜にアクセスが集中してレスポンスが遅い → インフラ増強が必要"、"特定のページでカート離脱が多い → 売上改善のヒント" といったことがわかる。**ログやデータは"溜めるだけ"では意味がなく、分析して初めてビジネスの意思決定に使える情報になる**。
>
> ただし毎日数百GB〜数TBのログが生まれる規模になると、**1台のPCでは処理しきれない**。分析の仕組み自体をスケールさせる必要がある。
>
> 今回学ぶサービスは、この **"集める → 加工する → 分析する → 可視化する"** という一連の流れを実現します。
>
> 1. **集める** → Kinesis（リアルタイム）、Snow Family / DataSync / Transfer Family（大量転送）
> 2. **加工する** → Glue（ETL）、EMR（分散処理）
> 3. **分析する** → Athena（SQLクエリ）
> 4. **可視化する** → QuickSight（ダッシュボード）
>
> この流れを意識しながら、個々のサービスを見ていきましょう。」

---

## Kinesisファミリー

### 導入（ストリーミングデータ）

> 「まずは**データを集める**フェーズから。最初のサービスは**Kinesis**です。
>
> さっき"ログがどんどん溜まっていく"と言いましたよね。もしこのログデータを**リアルタイムに分析したい**としたらどうしますか？ 1日分溜めてからバッチ処理する方法もあるけど、たとえばIoTセンサーの温度データが毎秒送られてくる。ゲームのプレイデータがリアルタイムに発生する。これを翌日まで待つわけにはいかない。
>
> 川の流れで考えてみてください。バッチ処理は**ダムに水を溜めてから一気に処理する**方式。ストリーミング処理は**流れている水をその場でリアルタイムに受け取る**方式。Kinesisは後者——データの**リアルタイム収集・配信**に特化したサービス群です。」

### Kinesis Data StreamsとData Firehose

> 「Kinesisで特に重要なのが**Data Streams**と**Data Firehose**の使い分け。
>
> **Data Streams**はデータを**収集して一時保存**するサービス。コンシューマー（LambdaやEC2）が自分のペースでデータを読み取ってカスタム処理する。自由度が高いけど、**シャード**数の管理やコンシューマーのコードを自分で書く必要がある。シャードとはストリームを分割した**データの通り道**のことで、高速道路の車線のようなもの。車線（シャード）を増やせば同時に処理できるデータ量が増える。Data Streamsではこのシャード数を自分で設定・管理する必要があります。
>
> **Data Firehose**はデータを**S3やRedshiftなどの宛先に自動配信**するサービス。コンシューマーのコードを書かなくていい。シャード管理も不要で自動スケーリング。
>
> ここで1つ**重要な特性**を押さえておきましょう。Firehoseは受信したデータを即座に配信するのではなく、**一定量（例: 5MB）か一定時間（例: 60秒）どちらかの閾値に達したらまとめて配信**します。これを**バッファリング**と言います。つまり数十秒〜数分の**タイムラグ**が出る。"完全なリアルタイム" ではなく "**準リアルタイム**" です。その代わりリクエスト数が減るのでコストが安い。試験では "データが数分遅れても問題なく、コストを抑えたい" と出てきたらFirehoseを選びます。逆に "1秒でも遅れたくない、厳密なリアルタイム処理がしたい" なら Data Streams + カスタムコンシューマーの構成になります。
>
> 使い分けのポイントは、**カスタム処理が必要か、宛先への配信だけでいいか**。そしてFirehoseを選ぶときは**数分の遅延を許容できるか**も確認してください。
>
> "リアルタイムデータをS3に自動保存したい"→ **Firehose**。"リアルタイムデータに独自のロジックで処理したい"→ **Data Streams**。
>
> ハンズオン②でFirehoseからS3への自動配信を実際に体験します。」

### Video StreamsとData Analytics

> 「残り2つは簡単に。
>
> **Video Streams**は動画のリアルタイム収集。防犯カメラの映像分析などに使います。
>
> **Data Analytics**はストリーミングデータに対する**リアルタイム分析**サービス。**SQL**か**Apache Flink**で書けて、リアルタイムダッシュボードや異常検知に使います。Flinkは "1件1件のデータをイベント単位でリアルタイムに処理できる" ストリーミング専用フレームワーク、とだけ押さえておいてください。Sparkとの比較は後ほどEMRのところで詳しくやります。
>
> ここで**名称の注意**を1つ。このサービス、元々は "Kinesis Data Analytics" という名前だったんですが、現在は **"Amazon Managed Service for Apache Flink"** にリブランドされています。長い名前ですよね。試験では新旧どちらの名称でも出題されるので、**"Kinesis Data Analytics" と "Managed Service for Apache Flink" は同じサービス**だと覚えておいてください。選択肢に "Managed Service for Apache Flink" が出てきて焦らないように。」

---

## Snow Family

### 導入（物理デバイスによるデータ転送）

> 「引き続き**データを集める**フェーズ。次は**Snow Family**。
>
> Kinesisはリアルタイムのストリーミングデータを集めるサービスでした。では、すでに手元にある**大量のデータをAWSに移したい**場合はどうするか？
>
> 100TBのデータをネットワーク経由で転送するとどれくらいかかると思いますか？ 1Gbpsの回線でも、単純計算で**10日以上**。しかもその間ずっと帯域を占有する。
>
> そこで発想を変える。**物理デバイスにデータを詰めて、トラックで運ぶ**。引っ越しで荷物が多すぎて宅配便では間に合わないとき、トラックをチャーターするのと同じです。」

### Snowcone・Snowball Edge・Snowmobile

**📌 講義中にGoogle画像検索で実物写真を見せる**
- 「AWS Snowcone device」で検索
- 「AWS Snowball Edge device」で検索
- 「AWS Snowmobile truck」で検索

> 「Snow Familyは3サイズ展開。
>
> **Snowcone**は最小。手のひらサイズ約2.1kg、8〜14TB。エッジでの小規模データ収集向け。
>
> **Snowball Edge**は中〜大規模。最大80TB。Compute Optimizedタイプは計算能力もあって、エッジで機械学習推論にも使える。データセンター移行の定番です。
>
> **Snowmobile**は……**コンテナトラック**です。最大100PB。AWSスタッフがトラックをデータセンターに横付けして直接データを吸い出す。10PB超の超大規模移行用。
>
> 試験は**データ量で判断**。数十TB → Snowball Edge、100PB規模 → Snowmobile、エッジで小規模 → Snowcone。」

---

## DataSync

> 「Snow Familyは物理デバイスでしたが、**ネットワーク経由**で大量データを高速に転送したい場合はどうするか？ それが**DataSync**です。
>
> **DataSync**は、オンプレミスとAWS間、またはAWS間で**大量データを高速転送・同期**するサービス。ネットワーク帯域の最大10倍に最適化される。整合性検証、暗号化も自動。
>
> オンプレミスのNASからS3への大規模データ移行、定期的なデータバックアップ、AWSリージョン間のデータレプリケーションなどに使います。」

---

## Transfer Family

> 「もう一つネットワーク経由のデータ転送サービス、**Transfer Family**。
>
> Transfer Familyは、**SFTP/FTPでデータをやり取りしている既存の業務をAWSに移行する**ためのサービス。
>
> まずSFTP/FTPがどういう場面で使われているかイメージしてもらいましょう。たとえば経理部門で、毎月取引先から請求書データをファイルで受け取る業務。取引先はSFTPクライアントというソフトで、こちらのサーバーにファイルをアップロードしてくる。こういった**定期的なSFTP/FTPのファイル連携**は、特定の業界では今でも非常に多いです。
>
> 問題は、このSFTPサーバーが社内のオンプレミスにあること。ハードウェアの保守、OSのアップデート、ディスク容量の管理……全部自分たちでやらないといけない。これをAWSに移行したい。でも取引先に「SFTPの接続先を変えてください」とお願いするのは大変ですよね。先方のシステム改修が必要になるかもしれない。
>
> Transfer Familyを使えば、取引先のSFTPクライアント設定を一切変えずに、バックエンドだけS3に切り替えられる。取引先から見たら今まで通りSFTPで送っているだけ。裏側がS3になったことすら気づかない。**既存のプロトコルを維持したまま移行**できるのがポイントです。
>
> DataSyncとの違いを整理しましょう。
>
> "既存のSFTP/FTPプロトコルを維持したい"→ **Transfer Family**。
> "とにかく大量データを高速に転送・同期したい"→ **DataSync**。
>
> 試験ではこの使い分けがよく出ます。」

---

## ETL・AWS Glue

### ETLの概念

> 「ここから**データを加工する**フェーズに入ります。重要なキーワードが**ETL**。
>
> **E**xtract（抽出）→ **T**ransform（変換）→ **L**oad（読み込み）。データソースからデータを取り出して、必要な形式に加工して、分析基盤に格納する。この3ステップです。
>
> たとえば、各店舗の売上データ（CSV、Excelなどバラバラの形式）を集めてきて（E）、統一フォーマットに変換して（T）、データウェアハウスに入れる（L）。」

### AWS Glue

> 「AWSでETLを担うのが**AWS Glue**。サーバーレスのETLサービスです。Glueの中で特に重要な機能が**クローラー**と**データカタログ**なので、ここをしっかり説明します。
>
> まず問題を考えてみましょう。S3バケットに大量のCSVファイルが保存されているとします。このデータをSQLで分析したい。でもCSVファイルってただのテキストですよね。"1列目が日付で、2列目が商品名で、3列目が金額"みたいな**構造の情報**は、ファイル自体には書かれていない。データベースならテーブル定義があるけど、S3にはそれがない。
>
> じゃあ誰かがデータの構造を調べて登録する必要がある。でもファイルが何百、何千とあったら、手作業ではやってられない。
>
> ここで登場するのが**クローラー**です。クローラーはS3バケットの中身を**自動でスキャンして、データの構造を推測してくれるロボット**です。"このCSVの1列目はdate型、2列目はstring型、3列目はint型……"というように、カラム名やデータ型を自動で認識する。
>
> そしてクローラーが認識した結果を登録する先が**データカタログ**です。データカタログは、S3に保存されたデータの**目次・索引**のようなもの。"どこに""どんな構造の""どんなデータがあるか"が一覧になっている。本そのもの（データ）はS3にあるけど、目次（データカタログ）を見れば全体像がすぐわかる、というイメージです。
>
> 流れを整理すると：
> 1. S3にデータが保存されている（CSV、JSON、Parquetなど）
> 2. **クローラー**がS3を自動スキャンして、データの構造（スキーマ）を認識する
> 3. 認識した結果を**データカタログ**（テーブル定義）として登録する
> 4. Athenaなどの分析サービスがデータカタログを参照して、S3のデータにSQLクエリを実行できるようになる
>
> つまり、クローラーがデータの構造を自動で読み取って、データカタログに登録してくれるおかげで、**S3のファイルをまるでデータベースのテーブルのように扱える**ようになる。これがGlueの最大の価値です。
>
> ハンズオン②で実際にクローラーを動かして、S3のデータからテーブルが自動生成される様子を体験しましょう。」

---

## EMR（Elastic MapReduce）

### 導入（ビッグデータ・分散処理）

> 「もう一つデータを加工するサービス、**EMR**です。
>
> さっき"1台のPCでは処理しきれない"と言いましたが、具体的にイメージしてみましょう。100GBのログファイルを1台のパソコンで処理したら何時間かかると思いますか？ 下手すると丸一日。じゃあ100台で分担したら？ 単純計算で1/100。
>
> これが**分散処理**の考え方です。大量のデータを複数のサーバーに分割して並列処理する。EMRはこの分散処理を簡単に構築できるマネージドサービスです。」

### HadoopとApache Spark

> 「EMRでよく使われるフレームワークが2つ。**Hadoop**と**Apache Spark**。
>
> Hadoopは元祖の分散処理フレームワーク。Hadoopには2つの主要コンポーネントがあります。1つが**HDFS（Hadoop Distributed File System）**。これはデータを複数のサーバーに分散して保存するファイルシステムです。普通のファイルシステムは1台のディスクにデータを置きますが、HDFSは同じデータを複数のノードにコピー（レプリケーション）して保存するので、1台が壊れてもデータが失われない。もう1つが**MapReduce**という処理モデル。データを分割して各ノードで処理（Map）し、結果を集約（Reduce）する。ディスクベースなので大量データのバッチ処理に強い。ただし遅い。
>
> Apache Sparkはその次世代。データを**メモリ上**で処理するので、Hadoop MapReduceと比べて**最大100倍高速**。バッチ処理だけでなくストリーミング処理や機械学習にも対応。
>
> 使い分けはシンプルで、**処理速度重視ならSpark、大量データをコスト効率よくバッチ処理するならHadoop**。Sparkはメモリ処理で高速なうえ、バッチ・ストリーミング・機械学習まで1つでカバーする汎用性が強み。Hadoopはディスクベースで安定しているので、数百TBのログを翌朝までに集計するような大規模バッチに向いています。
>
> ちなみに先ほどKinesisで出てきた**Apache Flink**との違い。Sparkは短い間隔でまとめて処理する**マイクロバッチ方式**の準リアルタイム、Flinkは**1件1件イベント単位のリアルタイム処理**です。
>
> 試験で"高速""インメモリ"ときたら**Spark**、"大規模バッチ処理"ときたら**Hadoop**です。」

### EMRクラスターのノード構成

> 「EMRのクラスターは3種類のノードで構成されます。**クラスター**は複数のサーバーをまとめて1つのシステムとして動かすグループ、**ノード**はクラスター内の1台1台のサーバー（EC2インスタンス）です。EMRでは役割ごとにノードを分けて、複数台で協力して処理を行います。
>
> **プライマリノード**は司令塔。タスクの分配とクラスター管理を行う。1つだけ存在して、これが止まるとクラスター全体が止まる。
>
> **コアノード**は主力部隊。データの保存（HDFS）と処理の両方を担う。最低1つ必要。
>
> **タスクノード**は助っ人。処理だけやってデータは保存しない。忙しいときに追加して、終わったら削除できる。オプションです。
>
> ポイントは、**コアノードはデータを保存している**ので、削除するとHDFSのデータが失われること。タスクノードはデータを持っていないので、スケールダウンするならタスクノードから削除する。」

---

## Data Pipeline

> 「**Data Pipeline**はAWSサービス間のデータ移動・変換を定期的に自動実行するサービスですが、これも新規開発では**GlueやStep Functions**を使うのが推奨です。試験で出たら"レガシーなETLスケジューラ"と思っておけば大丈夫です。」

---

## Athena

> 「ここから**データを分析する**フェーズです。**Athena**、覚えていますか？ 第5回のデータベースの回で、Redshift Spectrumとの比較で少し触れました。"S3のデータをそのまま分析するなら、Redshiftクラスターがあれば**Spectrum**、なければ**Athena**"という話でしたね。
>
> 改めて整理すると、Athenaは**S3上のデータに直接SQLクエリを実行できるサーバーレスのクエリサービス**です。通常、データを分析しようと思ったらデータベースにロードする必要がありますよね。AthenaならそのステップがいらないS3にあるCSV、JSON、Parquetなどのファイルにそのままクエリが打てる。サーバーもデータベースも立てなくていい。
>
> そして、さっきGlueでデータカタログを作る話をしましたよね。Athenaはそのデータカタログを参照して、S3のデータの構造を理解します。つまり**Glueのクローラーがスキーマを自動認識 → データカタログに登録 → Athenaがカタログを見てSQLクエリを実行**という連携です。
>
> 料金はスキャンしたデータ量に対して課金されるので、**Parquetなどの列指向フォーマット**で保存して、必要な列だけスキャンすることでコスト削減できます。
>
> ハンズオン②でこの**Firehose → S3 → Glue → Athena**のパイプラインを一気通貫で作ります。ストリーミングデータの収集から分析までの流れを体験してもらいます。」

---

## QuickSight

> 「最後に**データを可視化する**フェーズ。Athenaで分析したデータを**見える化**するのが**QuickSight**。AWSの**BIツール**です。
>
> Athena、S3、RDS、Redshiftなど様々なデータソースに接続して、ドラッグ&ドロップでダッシュボードやグラフを作れる。
>
> QuickSightで覚えておいてほしいキーワードが **SPICE**。"Super-fast, Parallel, In-memory Calculation Engine" の略で、**QuickSight独自のインメモリエンジン**です。データソースから直接クエリを投げるとAthenaやRDSへの往復で時間がかかりますが、一度SPICEにデータを取り込んでおくと**メモリ上から高速に応答**できる。営業チームがダッシュボードを触ってグリグリ切り替えても待たされない、というのがSPICEの価値です。試験で "**高速なクエリ応答**" や "**インメモリ**" というキーワードがQuickSightと一緒に出てきたらSPICEを選びます。
>
> 試験で"データの視覚化""ダッシュボード"ときたら**QuickSight**。"高速応答"まで問われたら**SPICE**。」

---

## 説明のコツまとめ

1. **「〇〇とは」で始めない**。「こんな問題がある → だからこのサービスが必要」の順番で話す
2. **受講生に問いかけてから説明する**。「どんな場面で順番が入れ替わっても大丈夫ですか？」「100TBの転送に何日かかると思いますか？」と聞いて、2〜3秒の間を取る
3. **たとえ話を入れる**。注文伝票置き場、校内放送、ダムと川、引っ越しのトラックなど。ただし、たとえ話の後に必ず技術的な説明に戻すこと
4. **似たサービスは対比で教える**。「SQSとSNSの違いは？」「Data StreamsとFirehoseの使い分けは？」「Transfer FamilyとDataSyncは？」 対比させると記憶に残りやすい
5. **試験のポイントは最後に添える**。説明の流れの中で自然に「試験ではここが問われます」と伝える。最初に言うと暗記感が出てしまう

---

## ハンズオン

### 第8回（4/11） ハンズオン手順書

> **所要時間の目安**: 全体で約55〜70分（ハンズオン①: 15〜20分、ハンズオン②: 20〜25分 ※バッファ間隔の待ち時間含む、ハンズオン③: 15〜20分）

> **注意事項**
> - ハンズオン終了後は必ずリソースを削除してください（コスト発生防止）
> - 各手順の所要時間は目安です。チームで役割分担して進めてください
> - スクリーンショットを撮りながら進めると復習に役立ちます

> **講義の進め方**
> ハンズオン②のステップ3でテストデータ投入後、バッファ間隔（約60秒）の待ち時間があります。**待ち時間の間にステップ4（Glueクローラー作成）を先に進めてください**。

---

#### 第8回（4/11）ハンズオン①：Kinesis Data Streams → Lambda リアルタイムデータ処理

##### 想定ユースケース
工場やビルに設置された**IoT温度センサー**のデータをリアルタイムに監視するシステムを構築します。各センサーが定期的に温度データを送信し、異常な高温（30°C以上）を即座に検知してアラートを出す——という運用監視のシナリオです。実際の現場では、温度だけでなく湿度・振動・電流値なども同様の仕組みで処理され、設備異常の早期発見や予防保全に活用されています。

##### このハンズオンで学ぶこと
- Kinesis Data Streamsのデータストリーム作成とシャードの概念
- Lambda関数をKinesisのコンシューマーとして設定する方法
- ストリームへのデータ投入とリアルタイム処理の体験
- ストリーミングデータ処理の基本的な仕組みの理解

##### リソース状況
- **新規作成**: sensor-stream（Kinesis Data Streams）、process-sensor-data（Lambda関数）、イベントソースマッピング、IAMロール（Lambda用）

##### ステップ1: Kinesis Data Streamsを作成

1. コンソール上部の検索バーに「Kinesis」と入力 →「Kinesis」を選択
2. 「データストリーム」→「データストリームを作成」
3. 以下を設定:
   - データストリーム名: `sensor-stream`
   - 容量モード: `オンデマンド`（シャード数の自動管理）
   - 最大レコードサイズ / データストリーム設定 / タグ: **デフォルトのまま**
4. 「データストリームを作成」

##### ステップ2: Lambda関数を作成

1. Lambdaコンソール →「関数の作成」→「一から作成」
2. 以下を設定:
   - 関数名: `process-sensor-data`
   - ランタイム: `Python 3.14`
   - アーキテクチャ / 詳細設定: **デフォルトのまま**
   - 「デフォルトの実行ロールの変更」→「デフォルトロールを作成」のまま（CloudWatch Logsへのログ出力権限を持つロールが自動作成される）
3. 「関数の作成」
4. 作成されたロールにKinesisの読み取り権限を追加:
   - 「設定」タブ →「アクセス権限」→ ロール名をクリック → IAMコンソールが開く
   - 「許可を追加」→「ポリシーをアタッチ」→ `AWSLambdaKinesisExecutionRole` を検索して選択 →「許可を追加」
5. コードソースに以下を貼り付けて「Deploy」:
   ```python
   import json
   import base64

   def lambda_handler(event, context):
       for record in event['Records']:
           # Kinesisのデータはbase64エンコードされている
           payload = base64.b64decode(record['kinesis']['data']).decode('utf-8')
           data = json.loads(payload)

           sensor_id = data.get('sensorId', 'unknown')
           temperature = data.get('temperature', 0)

           # 温度が30度以上なら警告を出力
           if temperature >= 30:
               print(f"[ALERT] Sensor {sensor_id}: High temperature {temperature}°C!")
           else:
               print(f"[OK] Sensor {sensor_id}: Temperature {temperature}°C")

       return {'statusCode': 200, 'processedRecords': len(event['Records'])}
   ```
   > **コードの解説**
   > - Kinesisから渡されるデータは `event['Records']` にリスト形式で格納される（バッチサイズ分まとめて届く）
   > - 各レコードの `kinesis.data` はBase64エンコードされているため、`base64.b64decode()` でデコードしてからJSONとして読み取る
   > - デコードしたデータから `sensorId` と `temperature` を取り出し、30°C以上なら `[ALERT]`、それ以下なら `[OK]` としてログ出力する
   > - `print()` の出力は自動的にCloudWatch Logsに記録されるため、後のステップで確認できる

##### ステップ3: LambdaにKinesisトリガーを設定

1. `process-sensor-data` 関数の画面 →「トリガーを追加」
2. ソース: `Kinesis`
3. Kinesisストリーム: `sensor-stream`
4. バッチサイズ: `10`（最大10レコードをまとめて1回のLambda呼び出しで処理する設定。レコードが少ない場合はそれ以下の数で呼び出される）
5. 開始位置: `最新`
6. その他の設定: **デフォルトのまま**
7. 「追加」をクリック

##### ステップ4: テストデータをストリームに投入

1. **AWS CloudShell** を開く
   - コンソール左下の「CloudShell」をクリック
   - CloudShellはブラウザ上で使えるターミナルで、AWS CLIがあらかじめインストールされている。ローカルPCへの環境構築なしにAWSコマンドを実行できる
   - 初回起動時は環境の準備に1分ほどかかる場合がある
   - 複数行のコマンドを貼り付けると「複数行のテキストの安全な貼り付け」という確認ポップアップが表示されるので、「貼り付け」をクリックすればOK
2. 以下のコマンドを実行（正常な温度データ）:
   ```bash
   aws kinesis put-record \
     --stream-name sensor-stream \
     --partition-key sensor-001 \
     --data "$(echo '{"sensorId":"sensor-001","temperature":25,"location":"Tokyo"}' | base64)" \
     --region ap-northeast-1
   ```
3. 続けて以下を実行（高温アラートが出るデータ）:
   ```bash
   aws kinesis put-record \
     --stream-name sensor-stream \
     --partition-key sensor-002 \
     --data "$(echo '{"sensorId":"sensor-002","temperature":38,"location":"Osaka"}' | base64)" \
     --region ap-northeast-1
   ```

##### ステップ5: Lambda関数のログを確認

1. Lambdaコンソール → `process-sensor-data` →「モニタリング」→「CloudWatch Logsを表示」
2. 最新のログストリームをクリック
3. **確認**:
   - `[OK] Sensor sensor-001: Temperature 25°C` → 正常データ
   - `[ALERT] Sensor sensor-002: High temperature 38°C!` → 高温アラート
   - Kinesisに投入したデータがLambda関数でリアルタイムに処理されたことを確認

##### リソース削除

1. Lambda関数のKinesisトリガーを削除
2. Lambda関数 `process-sensor-data` を削除
3. Kinesis Data Streams `sensor-stream` を削除
4. IAMロール `process-sensor-data-role-xxxxx` を削除（Lambda作成時に自動生成されたロール。「設定」→「アクセス権限」でロール名を確認できる）

---

#### 第8回（4/11）ハンズオン②：Firehose → S3 → Glue → Athena データ分析パイプライン

##### 想定ユースケース
**株価のリアルタイムデータを収集・蓄積し、後から自由に分析できる基盤**を構築します。証券会社やフィンテック企業では、取引所から送られてくる大量の株価データ（銘柄コード・セクター・価格変動など）をリアルタイムに受け取り、S3に蓄積したうえでSQLで分析するというパイプラインが広く使われています。本ハンズオンでは、Firehoseがデータの受け取りとS3への自動配信を担い、Glueがデータ構造を自動認識し、Athenaでサーバーレスにクエリを実行する——という一連の流れを体験します。

##### このハンズオンで学ぶこと
- Firehose配信ストリームによるリアルタイムデータのS3自動配信
- Glueクローラーによるスキーマの自動検出とデータカタログの作成
- AthenaによるS3データへの直接SQLクエリの実行
- ストリーミングデータ収集から分析までの一気通貫パイプラインの理解

##### リソース状況
- **新規作成**: handson-pipeline-\<チーム名\>（S3バケット）、handson-stream（Firehose配信ストリーム）、pipeline-crawler（Glueクローラー）、handson_db（Glueデータベース）、IAMロール（Glue用）

##### ステップ1: 配信先S3バケットを作成

1. S3コンソール →「バケットを作成」
2. 以下を設定:
   - バケットタイプ: `汎用`
   - バケット名前空間: `グローバル名前空間`
   - バケット名: `handson-pipeline-<自分の名前>`
   - オブジェクト所有者 / ブロックパブリックアクセス設定 / バージョニング / タグ / デフォルトの暗号化 / 詳細設定: **すべてデフォルトのまま**
3. 「バケットを作成」

##### ステップ2: Firehose配信ストリームを作成

1. コンソール上部の検索バーに「Firehose」と入力 →「Amazon Data Firehose」を選択
2. 「Firehose ストリームを作成」をクリック
3. 以下を設定:
   - **ソースと送信先を選択**
     - ソース: `Direct PUT`（アプリケーションやCLIから直接Firehoseにデータを送る方式。Kinesis Data Streamsを経由する場合は別のソースを選ぶ）
     - 送信先: `Amazon S3`
     - Firehose ストリーム名: `handson-stream`
   - **レコードを変換および転換**: **すべてデフォルトのまま**（Lambda変換・Parquet転換・CloudWatch Logs解凍はいずれもオフ）
   - **送信先の設定**
     - S3バケット: `handson-pipeline-<自分の名前>` を「参照」から選択
     - 改行の区切り文字 / 動的パーティショニング: **デフォルトのまま**
     - S3バケットプレフィックス: `stream-data/`
     - S3バケットエラー出力プレフィックス: **空欄のまま**
   - **バッファのヒント、圧縮、ファイル拡張子、暗号化**
     - バッファサイズ: `1` MiB
     - バッファ間隔: `60` 秒（最短で確認するため）
     - 圧縮 / ファイル拡張子 / 暗号化: **デフォルトのまま**
   - **詳細設定**: **デフォルトのまま**（IAMロールが自動作成される）
4. 「Firehose ストリームを作成」
   - ※ 「Firehose is unable to assume role...」というエラーが出た場合は、IAMロールの作成が間に合っていないだけなので、1〜2分待ってからもう一度「Firehose ストリームを作成」をクリックしてください

##### ステップ3: テストデータを投入

1. 作成した `handson-stream` を選択
2. 画面内の「デモデータでテスト」セクションで「デモデータの送信を開始」をクリック
   - 株価のサンプルデータ（TICKER_SYMBOL、SECTOR、CHANGE、PRICEなど）が自動的にストリームに投入される
3. **約60秒待つ**（バッファ間隔）→ **待ち時間の間にステップ4を先に進める**
4. ステップ4の作業が終わったら、戻って「デモデータの送信を停止」をクリック（余計な料金が発生しないようにする）

##### ステップ4: Glueクローラーを作成（待ち時間中に実施）

1. コンソール上部の検索バーに「Glue」と入力 →「AWS Glue」を選択
2. 左メニュー「データカタログ」→「クローラー」→「クローラーの作成」
3. **Set crawler properties**: 名前に `pipeline-crawler` を入力 →「Next」
4. **Choose data sources and classifiers**:
   - 「Is your data already mapped to Glue tables?」→ `Not yet` を選択
   - 「Add a data source」をクリック:
     - データソース: `S3`
     - S3パス: `s3://handson-pipeline-<自分の名前>/stream-data/` を入力（「Browse S3」から選択も可）
     - その他の設定: **デフォルトのまま**
     - →「Add an S3 data source」
   - Custom classifiers: **空欄のまま**
   - →「Next」
5. **Configure security settings**:
   - 「Create new IAM role」→ `AWSGlueServiceRole-` の後に `pipeline` と入力 →「Create」
   - Lake Formation configuration / Security configuration: **デフォルトのまま**
   - →「Next」
6. **Set output and scheduling**:
   - 「Target database」→「Add database」→ Database type: `Glue Database` → Name: `handson_db` → Description / Location: **空欄のまま** →「Create database」→ 作成した `handson_db` を選択
   - Table name prefix / Maximum table threshold / Advanced options: **空欄・デフォルトのまま**
   - Crawler schedule → Frequency: `On demand`（デフォルトのまま。手動実行する）
   - →「Next」
7. **Review and create** → 内容を確認 →「Create crawler」

##### ステップ5: S3への配信を確認してクローラーを実行

1. S3コンソール → `handson-pipeline-<自分の名前>` → `stream-data/` フォルダを開く
2. `年/月/日/時/` のフォルダ構造で配信されたファイルを確認
3. **確認**: ファイルをダウンロード → テストデータがJSON形式で格納されている
4. Glueコンソールに戻る → `pipeline-crawler` →「クローラーを実行」
5. State が `RUNNING` → `READY` に戻るまで待つ（1〜2分）

##### ステップ6: データカタログを確認

1. 左メニュー「データカタログ」→「データベース」→ `handson_db` をクリック
2. 「テーブル」→ `stream_data` テーブルが自動作成されていることを確認
3. テーブルをクリック → **確認**: Schema セクションに `change`（double）、`price`（double）、`ticker_symbol`（string）、`sector`（string）の4カラムが表示されている。これはGlueがS3上のJSONデータからフィールド名とデータ型を自動的に判別した結果

##### ステップ7: Athenaでクエリを実行

1. コンソール上部の検索バーに「Athena」と入力 →「Athena」を選択
2. 初回はランディングページが表示されるので、左メニューの「クエリエディタ」をクリック
3. 「最初のクエリを実行する前に、Amazon S3 でクエリ結果の場所を設定する必要があります」と表示されるので、「設定を編集」をクリック →「クエリ設定」画面が開く →「管理」をクリック → 「Location of query result」に `s3://handson-pipeline-<自分の名前>/athena-results/` を入力 → その他の項目はデフォルトのまま →「保存」
4. 「エディタ」タブに戻る → 左側パネルでデータベース `handson_db` が選択されていることを確認（選択されていなければプルダウンから選択）
5. クエリエディタに以下を貼り付けて「実行」:
   ```sql
   SELECT * FROM stream_data LIMIT 10;
   ```
6. **確認**: 画面下部の「クエリ結果」エリアに、`change`、`price`、`ticker_symbol`、`sector` のカラムにデモデータの値がテーブル形式で表示される。S3に保存されたデータに対して、データベースを構築することなくSQLで直接クエリできることを体験
   - ※ `partition_0`〜`partition_3` という列が表示される場合がありますが、これはS3の `年/月/日/時/` フォルダ構造をGlueがパーティション（データの分割単位）として自動検出したものです

##### リソース削除

1. Firehose配信ストリーム `handson-stream` を削除
2. Glueクローラー `pipeline-crawler` を削除
3. Glueデータベース `handson_db` を削除
4. S3バケット内のオブジェクトをすべて削除（athena-results含む）→ バケットを削除
5. IAMロール（Glue用、Firehose用）を削除

---

#### 第8回（4/11）ハンズオン③：自作CSV → S3 → Glue → Athena バッチデータ分析

##### 想定ユースケース
小売チェーンの**店舗別・商品別の売上データを集計・分析する**シナリオです。各店舗から日次でCSVファイルとして売上データが届き、それをS3に蓄積して分析基盤とします。従来はデータベースサーバーを立ててデータを取り込む必要がありましたが、S3 + Glue + Athena の組み合わせなら、CSVをS3に置くだけでサーバーレスにSQLで集計できます。ハンズオン②がリアルタイム（ストリーミング）データの分析だったのに対し、こちらは蓄積済み（バッチ）データの分析パイプラインを体験します。

##### このハンズオンで学ぶこと
- 自分で作成したCSVデータをS3にアップロードしてデータレイクとして活用する方法
- GlueクローラーによるCSVのスキーマ自動検出を体験する
- AthenaでSQLの集計クエリ（GROUP BY、COUNT、SUMなど）を実行する
- ハンズオン②（ストリーミングデータ）との違いとして、バッチデータの分析パイプラインを理解する

##### リソース状況
- **新規作成**: sales-data-\<チーム名\>（S3バケット）、sales-crawler（Glueクローラー）、sales_db（Glueデータベース）、IAMロール（Glue用）

##### ステップ1: CSVファイルを作成

1. **AWS CloudShell** を開く（コンソール左下の「CloudShell」をクリック）
2. 以下のコマンドを実行して売上データCSVを作成:
   ```bash
   cat <<'EOF' > sales_data.csv
   date,store,product,quantity,price
   2024-04-01,Tokyo,Laptop,3,120000
   2024-04-01,Tokyo,Mouse,10,2500
   2024-04-01,Osaka,Laptop,2,120000
   2024-04-01,Osaka,Keyboard,5,8000
   2024-04-02,Tokyo,Laptop,1,120000
   2024-04-02,Tokyo,Monitor,4,35000
   2024-04-02,Osaka,Mouse,8,2500
   2024-04-02,Osaka,Monitor,3,35000
   2024-04-03,Tokyo,Keyboard,6,8000
   2024-04-03,Tokyo,Mouse,12,2500
   2024-04-03,Osaka,Laptop,4,120000
   2024-04-03,Osaka,Keyboard,7,8000
   EOF
   ```

##### ステップ2: S3バケットを作成してCSVをアップロード

1. 以下のコマンドを実行:
   ```bash
   aws s3 mb s3://sales-data-<自分の名前> --region ap-northeast-1
   ```
2. CSVをアップロード:
   ```bash
   aws s3 cp sales_data.csv s3://sales-data-<自分の名前>/sales/sales_data.csv
   ```
3. アップロードを確認:
   ```bash
   aws s3 ls s3://sales-data-<自分の名前>/sales/
   ```

##### ステップ3: Glueクローラーでスキーマを自動検出

1. Glueコンソール → 左メニュー「データカタログ」→「クローラー」→「クローラーの作成」
2. **Set crawler properties**: 名前に `sales-crawler` を入力 →「Next」
3. **Choose data sources and classifiers**:
   - 「Is your data already mapped to Glue tables?」→ `Not yet` を選択
   - 「Add a data source」をクリック:
     - データソース: `S3`
     - S3パス: `s3://sales-data-<自分の名前>/sales/` を入力
     - その他の設定: **デフォルトのまま**
     - →「Add an S3 data source」
   - Custom classifiers: **空欄のまま**
   - →「Next」
4. **Configure security settings**:
   - 「Create new IAM role」→ `AWSGlueServiceRole-` の後に `sales` と入力 →「Create」
   - Lake Formation configuration / Security configuration: **デフォルトのまま**
   - →「Next」
5. **Set output and scheduling**:
   - 「Target database」→「Add database」→ Database type: `Glue Database` → Name: `sales_db` → Description / Location: **空欄のまま** →「Create database」→ 作成した `sales_db` を選択
   - Table name prefix / Maximum table threshold / Advanced options: **空欄・デフォルトのまま**
   - Crawler schedule → Frequency: `On demand`（デフォルトのまま。手動実行する）
   - →「Next」
6. **Review and create** → 内容を確認 →「Create crawler」
7. `sales-crawler` →「Run crawler」→ State が `RUNNING` → `READY` に戻るまで待つ（1〜2分）

##### ステップ4: データカタログを確認

1. 左メニュー「データカタログ」→「データベース」→ `sales_db` をクリック
2. 「テーブル」→ `sales` テーブルが自動作成されていることを確認
3. テーブルをクリック → **確認**: CSVのヘッダー行（date、store、product、quantity、price）がカラム名として自動認識されている

##### ステップ5: Athenaで集計クエリを実行

1. Athenaコンソール →「クエリ設定」タブ →「管理」→ 「Location of query result」に `s3://sales-data-<自分の名前>/athena-results/` を入力 → その他の項目はデフォルトのまま →「保存」
2. 「エディタ」タブに戻る → 左側パネルでデータベース `sales_db` を選択
3. まず全データを確認:
   ```sql
   SELECT * FROM sales;
   ```
4. 店舗ごとの売上合計を集計:
   ```sql
   SELECT store, SUM(quantity * price) AS total_sales
   FROM sales
   GROUP BY store
   ORDER BY total_sales DESC;
   ```
5. 商品ごとの販売数量を集計:
   ```sql
   SELECT product, SUM(quantity) AS total_quantity
   FROM sales
   GROUP BY product
   ORDER BY total_quantity DESC;
   ```
6. **確認**: S3にあるCSVファイルに対して、データベースを構築することなくSQLで自由に集計できることを体験

##### リソース削除

1. Glueクローラー `sales-crawler` を削除
2. Glueデータベース `sales_db` を削除
3. S3バケット内のオブジェクトをすべて削除（athena-results含む）→ バケットを削除
4. IAMロール（Glue用）を削除

---

## 練習問題

### 問題1

ある企業は、工場内の数百のエッジデバイスを利用して製造ラインのストリームデータを収集しています。個々のデータサイズは2KBあまりですが、集計すると毎日合計で500MBになります。同社は、これらのストリームデータを取得して保存するソリューションを実装する必要があります。このストリーミングデータはリアルタイムに利用されるわけではないため、保存に数分のタイムラグが発生しても問題ありませんが、データは喪失することなく確実に保存される必要があります。また、コストを最小限に抑える必要があります。さらに、取得後10日以内のデータはすぐに利用できるようにして、10日経過したデータはアーカイブすることが必要です。

この要件を満たす、最も費用対効果が高いソリューションはどれでしょうか。

<details>
<summary>選択肢を見る</summary>

A. Amazon Kinesis Data Firehose配信ストリームを作成してデータストリームを取得する。Amazon Kinesis Data Firehoseにより、データストリームをAmazon S3バケットに保存する。10日経過したデータをAmazon S3 Glacierに移動するようにS3ライフサイクルルールを設定する。

B. Amazon Kinesis Data Streamsを作成してデータストリームを取得して、データストリームをAmazon S3バケットに保存する。10日経過したデータをAmazon S3 Glacierに移動するようにS3ライフサイクルルールを設定する。

C. Amazon EMRのHadoopストリーミングを構成して、Amazon Kinesis Data Streamsからデータストリームを取得して、データ処理した上でAmazon S3バケットに保存する。10日経過したデータをAmazon S3 Glacierに移動するようにS3ライフサイクルルールを設定する。

D. Amazon SQSスタンダードキューを作成してデータストリームを取得する。SQSキューをポーリングしてメッセージをAmazon S3バケットに保存する。コンシューマーを使用して、メッセージ保持期間を10日に設定して、10日経過したメッセージをAmazon S3 Glacierにコピーして、SQSキューから削除する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
A

### 解説
Amazon Kinesis Data Firehoseはストリーミングデータを変換してS3、Redshift、OpenSearchなどの宛先に自動配信するサービスです。

- データロードは60秒間隔のバッチ処理のため、ミリ秒単位のリアルタイム処理はできないが、「数分のタイムラグOK」の要件には十分対応できる
- シャード管理が不要で自動スケーリングするため、運用コストが低い
- Data Streamsと比較してコスト効率が高い
- S3ライフサイクルルールで10日後にGlacierへアーカイブすることでストレージコストも最適化できる

### 他の選択肢が不適切な理由
B
Amazon Kinesis Data Streamsは1秒以下のリアルタイム処理向けに設計されており、S3への自動配信機能を持っていません。S3に保存するにはFirehoseまたはカスタムコンシューマーが必要です。今回はリアルタイム性が不要なので、Data Streamsはオーバースペックかつコスト高となります。

C
Amazon EMRのHadoopストリーミングを構成するのは、ストリームデータの保存という要件に対して過剰な構成です。EMRクラスターの運用コストもかかるため、コスト最適ではありません。

D
Amazon SQSはメッセージキューサービスであり、ストリーミングデータの収集・配信には向いていません。また、SQSのメッセージ保持期間は最大14日であり、ライフサイクル管理の柔軟性に欠けます。

### まとめ
リアルタイム性が不要なストリームデータのS3への自動配信にはKinesis Data Firehoseが最適。Data Streamsはリアルタイム処理が必要な場合に使用する。S3ライフサイクルルールと組み合わせてストレージコストも最適化できる。

## AthenaによるS3データのサーバーレスSQLクエリ

</details>

---

### 問題2

ある企業では請求処理のアウトソーシング事業を行っています。請求情報はCSVデータ形式でストレージに保管され、SQLクエリ処理を実施されます。そして、処理結果データは30日間だけ保存されます。30日が経過するとこれらのデータは必要がなくなります。

このデータを保存するための、コスト最適なストレージタイプと処理方法を選択してください。

<details>
<summary>選択肢を見る</summary>

A. Amazon RDS MySQLによってデータのクエリ処理を実施後、データを30日後に削除するライフサイクルポリシーを設定する。

B. Amazon S3バケットの標準ストレージにデータを保存して、Amazon DynamoDBを利用してデータ解析し、データを30日後に削除するライフサイクルポリシーを設定する。

C. Amazon S3バケットの標準ストレージにデータを保存して、Amazon Athenaを利用してデータ解析し、データを30日後に削除するライフサイクルポリシーを設定する。

D. Amazon S3 Glacier Instant Retrievalにデータを保存してAmazon Redshiftでデータ解析を実施後、データを30日後に削除するライフサイクルポリシーを設定する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
C

### 解説
Amazon AthenaはS3に保存されたデータに対して直接SQLクエリを実行できるサーバーレスのクエリサービスです。

- CSV、JSON、Parquetなどのファイル形式にデータベースにロードすることなく直接クエリを実行できる
- サーバーレスでインフラ管理が不要
- スキャンしたデータ量に対してのみ課金されるため、30日間だけ使って捨てるデータには最適
- S3のライフサイクルポリシーで30日後にデータを自動削除できる

### 他の選択肢が不適切な理由
A
Amazon RDS MySQLでもSQLクエリは可能ですが、常時稼働するDBインスタンスのコストが発生します。30日で消えるデータに常設DBはコスト効率が悪いです。また、RDSにはライフサイクルポリシーの機能はありません。

B
Amazon DynamoDBはNoSQLデータベースであるため、SQLクエリ処理を実行できません。

D
Amazon S3 Glacier Instant Retrievalはアーカイブ用ストレージであり、データ分析用途には向いていません。保存料金は安い（S3 Standardの約1/5）が、データを取り出すたびに取得料金が発生するため、何度もデータを読み込む分析処理では取り出しコストが膨れ上がります。さらに最低保持期間が90日と設定されているため、30日間でデータを削除する場合は60日分の無駄なコストが発生します。Amazon Redshiftは大規模データウェアハウス向けで高額です。

### まとめ
S3のCSVデータにサーバーレスでSQLクエリを実行したい場合はAthena。料金はスキャンしたデータ量のみ。S3ライフサイクルポリシーと組み合わせてデータの自動削除も実現できる。

## AWS Data PipelineによるDynamoDBデータの定期取得

</details>

---

### 問題3

ある企業では、DynamoDBを利用したデータソリューションを構築しています。このソリューションでは、毎週定期的なタイミングでDynamoDBからセッションデータを取得して、レポート生成処理に送信する必要があります。

この要件を満たすための方法を選択してください。

<details>
<summary>選択肢を見る</summary>

A. DynamoDBイベントにより定期的にDynamoDBからデータを取得する。

B. AWS Data Pipelineにより定期的にDynamoDBからデータを取得する。

C. AWS Data Workflowにより定期的にDynamoDBからデータを取得する。

D. DynamoDBストリームにより定期的にDynamoDBからデータを取得する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
B

### 解説
AWS Data PipelineはAWSサービス間のデータ移動と変換をスケジュールベースで自動実行するサービスです。

- DynamoDBに対して定期的なデータ取得タスクを設定できる
- データ駆動型のワークフローを定義し、タスクの正常完了をトリガーにデータ変換と送信タスクを実行可能
- スケジュールベースの実行が可能で、毎週定期的なデータ取得の要件に合致する

### 他の選択肢が不適切な理由
A
「DynamoDBイベント」という機能名はDynamoDBに存在しません。架空のサービス名に注意が必要です。

C
「AWS Data Workflow」というサービスはAWSに存在しません。実在しないサービス名で作られたひっかけ選択肢です。

D
DynamoDBストリームはテーブルへの書き込みイベント（追加・変更・削除）をトリガーにしてLambda関数などを起動する仕組みです。「データが変更されたとき」に動くイベント駆動型であり、「毎週定期的に」動くスケジュール型ではありません。

### まとめ
DynamoDBからスケジュールベースで定期的にデータを取得するにはAWS Data Pipelineを使用する。ただし、Data Pipelineはレガシーサービスであり、新規開発ではGlueやStep Functionsが推奨されるケースが増えている。

## Kinesis Data Streams + Apache Flinkによるリアルタイムストリーム分析

</details>

---

### 問題4

ある企業は、AWS上でIoTアプリケーションを構築しています。このアプリケーションでは数千台のデバイスからのストリームデータを収集して、リアルタイムデータをクエリ処理することで時系列で分割します。さらに分割後のデータをストレージに保存することが必要です。

この要件を満たすために、ソリューションアーキテクトはどうするべきでしょうか。

<details>
<summary>選択肢を見る</summary>

A. デバイスからのデータをAmazon Kinesis Data Firehoseの配信ストリームで取得して、Amazon Managed Service for Apache Flinkのクエリ処理でデータを分割する。その上で、Amazon Kinesis Data FirehoseによってデータをAmazon S3に保存する。

B. デバイスからのデータをAmazon Kinesis Data Streamに取得して、Amazon Managed Service for Apache Flinkのクエリ処理でデータを分割する。その後、Amazon Kinesis Data Firehoseの配信ストリームにデータを送信して、Amazon S3に保存する。

C. デバイスからのデータをAmazon Kinesis Data Streamに取得して、Amazon Kinesis Data FirehoseのSQL処理でデータを分割して、その上で、Amazon Kinesis Data FirehoseによってデータをAmazon S3に保存する。

D. デバイスからのデータをAmazon Kinesis Data Streamに取得して、Amazon Managed Service for Apache Flinkのクエリ処理でデータを分割してAmazon S3に保存する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
D

### 解説
リアルタイムにデータを収集するにはKinesis Data Streamsを使用し、収集したデータに対してリアルタイム分析を行うにはAmazon Managed Service for Apache Flink（旧Kinesis Data Analytics）を使用します。

- Kinesis Data Streamsは1秒以下のリアルタイムデータ収集に対応
- Amazon Managed Service for Apache FlinkはApache Flinkを利用してストリーミングデータのリアルタイムクエリ処理を実行できる
- Apache FlinkアプリケーションはS3やOpenSearchに直接配信する機能を持っているため、Firehoseを経由する必要がない

### 他の選択肢が不適切な理由
A
Kinesis Data Firehoseは60秒間隔のバッチ処理であり、リアルタイムクエリ処理には向いていません。リアルタイムデータ収集にはData Streamsを使用すべきです。

B
Apache FlinkはS3に直接配信できるため、Firehoseを間に挟む構成は冗長です。

C
Kinesis Data FirehoseにはSQL処理機能はありません。データ変換はLambda関数との連携で行います。ストリーミングデータのクエリ処理にはApache Flinkが適しています。

### まとめ
リアルタイムデータ収集にはData Streams、リアルタイム分析にはApache Flink。Apache FlinkはS3に直接配信可能なため、Firehoseを経由する必要はない。Firehoseは「宛先への自動配信」が主目的。

## 総合データパイプライン（Kinesis + Firehose + Glue + EMR Spark）

</details>

---

### 問題5

ある企業のアプリケーションでは、大量のクリックストリームデータをAmazon S3バケットに保存するためのクリックストリーム分析ツールを開発しています。要件は、ウェブサイトから送信される膨大なクリックストリームをAPI経由で取得し、適切なデータ形式に変換してAmazon S3バケットに保存することです。その後、Amazon S3バケットに保存されたクリックストリームデータを迅速に分析するためのデータパイプラインを構成します。このパイプラインでは、データ内容に応じて保存データを分類した上で、一部のデータはさらに分析する必要があります。

運用上のオーバーヘッドを抑えつつ、これらの要件を満たすソリューションはどれでしょうか。（3つ選択）

<details>
<summary>選択肢を見る</summary>

A. Amazon API Gateway APIに連動したAWS Lambda関数がクリックストリームを取得し、Amazon Kinesis Data Streamsに配信する。

B. Amazon API Gateway APIに連動したAWS Lambda関数がクリックストリームを取得し、Amazon Data Firehoseに配信する。

C. Kinesis Data Streamsがクリックストリームデータを処理し、Amazon Data Firehoseを使用してデータをAmazon S3バケットに保存する。

D. Amazon Data Firehoseがクリックストリームデータを処理して、データをAmazon S3バケットに保存する。

E. AWS Glueクローラーを設定し、Amazon S3バケット内のデータを分割する。Amazon EMR Sparkジョブを設定し、データをクエリする。

F. AWS Glueクローラーを設定し、Amazon S3バケット内のデータを分割する。Amazon Athenaを設定し、データをクエリする。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
A、C、E

### 解説
クリックストリーム分析パイプラインは2つのフェーズに分かれます。

**フェーズ1: データの収集・変換・保存（A + C）**
- API Gateway + Lambda関数でクリックストリームを取得し、Kinesis Data Streamsに送信する
- Kinesis Data Streamsがリアルタイムにデータを処理し、Data Firehoseがデータを変換してS3に配信する

**フェーズ2: 保存データの分類・分析（E）**
- AWS Glueクローラーがデータを自動スキャンして分類し、データカタログに登録する
- Amazon EMR Sparkジョブで大量データを高速に分析する

### 他の選択肢が不適切な理由
B
LambdaからFirehoseに直接送信する構成では、Firehose単独ではクリックストリームデータの処理を十分に行えません。Data Streamsで受けてからFirehoseで変換・配信する構成が適切です。

D
Data Firehose単独ではクリックストリームデータの処理は不十分です。Data Streamsによるリアルタイム処理が必要です。

F
Amazon Athenaは通常のクエリ分析には便利ですが、大量のクリックストリームデータをリアルタイムかつ迅速に分析するにはEMR Sparkの方が適しています。EMR Sparkは数千台のインスタンスにスケール可能で、大量データの高速分析に向いています。

### まとめ
エンドツーエンドのデータパイプライン: API Gateway + Lambda（取得）→ Data Streams（リアルタイム処理）→ Firehose（変換・配信）→ S3（保存）→ Glue（分類）→ EMR Spark（高速分析）。

**Data StreamsとFirehoseの組み合わせについて**: Data Streamsはリアルタイムでデータを受け取って処理する（1秒以下のレイテンシー）が、S3への自動配信機能を持たない。Firehoseはデータを変換して宛先（S3、Redshiftなど）に自動配信できるが、リアルタイム処理はできない（60秒間隔のバッチ）。両方を組み合わせることで「リアルタイム処理」と「自動配信」を同時に実現できる。単独でも使えるが、それぞれの弱点を補い合う構成として覚えておくとよい。

## Snow Familyデバイスの選択による大規模データ移行

</details>

---

### 問題6

ある企業は、オンプレミスのデータセンターに保存されている50TBのデータをAWSに移行する必要があります。データセンターのネットワーク帯域は限られており、ネットワーク経由での転送では数週間かかることが見込まれます。移行は1週間以内に完了する必要があります。

この要件を満たすために、最も適切なソリューションはどれでしょうか。

<details>
<summary>選択肢を見る</summary>

A. AWS Direct Connectを構成して、専用線経由でデータを転送する。

B. AWS Snowball Edgeデバイスを利用して、データを物理的に転送する。

C. AWS DataSyncを利用して、既存のネットワーク経由でデータを高速転送する。

D. AWS Transfer Familyを利用して、SFTPプロトコルでデータを転送する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
B

### 解説
AWS Snowball Edgeは中〜大規模データ転送向けのポータブルデバイスで、最大80TBの使用可能ストレージを持ちます。

- 50TBのデータを1台のSnowball Edgeで転送可能（最大80TB）
- AWSに注文してデバイスを受け取り、データを格納して返送するだけで移行完了
- ネットワーク帯域に依存しないため、帯域が限られた環境でも1週間以内に完了できる
- データは転送中に暗号化される

### 他の選択肢が不適切な理由
A
AWS Direct Connectは専用線接続の構築に数週間〜数ヶ月かかるため、1週間以内という要件を満たせません。

C
AWS DataSyncはネットワーク帯域の最大10倍に最適化して転送しますが、既存のネットワーク経由で転送するため、帯域が限られている状況では数週間かかる可能性があります。

D
AWS Transfer FamilyもSFTPプロトコル経由でのネットワーク転送であるため、帯域が限られている状況では時間がかかります。

### まとめ
ネットワーク転送が困難な数十TBのデータ移行にはSnowball Edge。データ量で判断する: 8〜14TB → Snowcone、最大80TB → Snowball Edge、100PB規模 → Snowmobile。

## DataSyncとTransfer Familyの使い分け

</details>

---

### 問題7

ある企業は、オンプレミス環境のNASストレージに保存されたデータを、定期的にAmazon S3バケットに同期する必要があります。データの整合性検証と暗号化が自動的に行われることが求められています。また、別の部門では取引先がSFTPプロトコルでファイルを送信しており、そのファイルもS3バケットに保存したいと考えています。

オンプレミスNASからS3への定期同期と、SFTPによるファイル受信のそれぞれに最適なサービスの組み合わせはどれでしょうか。

<details>
<summary>選択肢を見る</summary>

A. オンプレミスNASからの同期にAWS Transfer Familyを使用し、SFTPファイル受信にもAWS Transfer Familyを使用する。

B. オンプレミスNASからの同期にAWS DataSyncを使用し、SFTPファイル受信にもAWS DataSyncを使用する。

C. オンプレミスNASからの同期にAWS DataSyncを使用し、SFTPファイル受信にAWS Transfer Familyを使用する。

D. オンプレミスNASからの同期にAWS Storage Gatewayを使用し、SFTPファイル受信にAWS Transfer Familyを使用する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
C

### 解説
2つの異なる要件に対して、それぞれ最適なサービスを選択します。

**NAS（Network Attached Storage）とは**: ネットワークに接続して複数のPCやサーバーからファイルを共有できるストレージ機器。家庭用のファイルサーバーから企業の大規模ファイル共有まで幅広く使われる。AWSで対応するサービスはEFS（Elastic File System）。NASはNFS/SMBプロトコルでファイル単位のアクセスを行う「ファイルストレージ」であり、S3のようなHTTP APIでアクセスする「オブジェクトストレージ」やEBSのようなブロック単位でアクセスする「ブロックストレージ」とは種類が異なる。

**オンプレミスNASからS3への定期同期 → DataSync**
- DataSyncはオンプレミスとAWS間で大量データを高速に転送・同期するサービス
- ネットワーク帯域の最大10倍に最適化して転送し、整合性検証と暗号化を自動で実行する
- 定期的な同期スケジュールの設定が可能

**SFTPによるファイル受信 → Transfer Family**
- Transfer FamilyはSFTP/FTPS/FTPプロトコルでS3やEFSとファイル転送を行うフルマネージドサービス
- 取引先のSFTPクライアント設定を変更せずに、バックエンドだけS3に切り替えられる

### 他の選択肢が不適切な理由
A
Transfer FamilyはSFTPサーバーとしてのファイル受信が主目的であり、NASからの大量データの定期同期には向いていません。

B
DataSyncはSFTPプロトコルをサポートしていないため、取引先のSFTPファイル受信には使用できません。

D
Storage Gatewayはハイブリッドストレージ構成（オンプレミスストレージとS3の連携）に使うサービスであり、NASからの定期同期にはDataSyncの方が適切です。

### まとめ
大量データを高速に転送・同期したい → DataSync。既存のSFTP/FTPプロトコルを維持したい → Transfer Family。この使い分けが試験の重要ポイント。

## EMRクラスターのタスクノードによるスケーラブルな計算能力

</details>

---

### 問題8

ある企業は、Amazon EMRクラスターを使用してビッグデータの分散処理を行っています。処理負荷が変動するため、ピーク時に計算能力を追加し、ピークが過ぎたら削除してコストを抑えたいと考えています。ただし、HDFSに保存されたデータが失われることは許容できません。

この要件を満たすために、どのノードを追加・削除するべきでしょうか。

<details>
<summary>選択肢を見る</summary>

A. コアノードを追加・削除する。

B. タスクノードを追加・削除する。

C. プライマリノードを追加・削除する。

D. コアノードとタスクノードの両方を追加・削除する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
B

### 解説
EMRクラスターは3種類のノードで構成されます。タスクノードは処理（タスク実行）のみを担い、HDFSにデータを保存しません。

- タスクノードはデータを持たないため、追加も削除も自由に行える
- ピーク時にタスクノードを追加して計算能力を増強し、ピーク後に削除してコストを抑えられる
- HDFSのデータには一切影響しない

### 他の選択肢が不適切な理由
A
コアノードはデータの保存（HDFS）と処理の両方を担います。コアノードを削除するとHDFSのデータが失われるため、「データが失われることは許容できない」という要件に反します。

C
プライマリノードはクラスター全体の管理・調整を行う唯一のノードです。プライマリノードを削除するとクラスター全体が停止します。追加もできません。

D
コアノードを含む追加・削除はHDFSデータ喪失のリスクがあるため不適切です。

### まとめ
EMRクラスターのスケーラブルな計算リソースにはタスクノードを使用する。コアノードを削除するとHDFSのデータが失われる。プライマリノードは1つだけ必須。スケールダウンはタスクノードから。

## QuickSightとSPICEによるインタラクティブダッシュボード

</details>

---

### 問題9

ある企業は、Amazon S3、Amazon Athena、Amazon RDSなど複数のデータソースに格納されたデータを使って、営業チーム向けのインタラクティブなダッシュボードを作成したいと考えています。ダッシュボードはドラッグ&ドロップで作成でき、高速なクエリ応答が必要です。

この要件を満たすために、最も適切なAWSサービスはどれでしょうか。

<details>
<summary>選択肢を見る</summary>

A. Amazon Athenaでクエリ結果をCSV出力し、Excelでグラフを作成する。

B. Amazon QuickSightを利用してダッシュボードを作成し、SPICEにデータを取り込んで高速応答を実現する。

C. Amazon CloudWatchダッシュボードを利用して、データの視覚化を行う。

D. Amazon OpenSearch Serviceを利用して、Kibanaダッシュボードを作成する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
B

### 解説
Amazon QuickSightはAWSが提供するBIツール（ビジネスインテリジェンスツール）で、データをダッシュボードやグラフで視覚化します。

- S3、Athena、RDS、Redshiftなど様々なデータソースに接続可能
- ドラッグ&ドロップでインタラクティブなダッシュボードを作成できる
- SPICE（Super-fast, Parallel, In-memory Calculation Engine）にデータを取り込むことで、データソースに毎回クエリを投げなくても高速にクエリ応答できる
- SPICEはQuickSight独自のインメモリエンジンで、高速なダッシュボード表示を実現する

### 他の選択肢が不適切な理由
A
AthenaでCSV出力してExcelでグラフ作成は可能ですが、インタラクティブなダッシュボードではなく、更新のたびに手動でCSVエクスポートが必要です。

C
Amazon CloudWatchダッシュボードはAWSリソースの監視（CPU使用率、メモリなど）に使うものであり、ビジネスデータの視覚化には向いていません。

D
Amazon OpenSearch Service + Kibanaはログ分析やリアルタイムモニタリングに向いており、営業チーム向けのBIダッシュボードとしてはQuickSightの方が適切です。

### まとめ
データの視覚化・ダッシュボード作成にはQuickSight。高速クエリ応答が必要な場合はSPICEにデータを取り込む。CloudWatchダッシュボードはインフラ監視用であり、ビジネスデータの可視化にはQuickSightを使う。

## Kinesis Data Streams + Firehose + Lambdaによるデータパイプライン

</details>

---

### 問題10

あなたはソリューションアーキテクトとして、AWSを利用したIoTデータソリューションを構築しています。このソリューションは様々なセンサーデータを取得して蓄積しつつ、リアルタイムにストリームデータを蓄積する処理を実現します。ストリームデータを蓄積する際は、データを最適なデータ形式に変換してから、コスト効率が良いデータレイヤーに保存する必要があります。

この要件を満たす方法を選択してください。（2つ選択）

<details>
<summary>選択肢を見る</summary>

A. Amazon Kinesis Data Streamsアプリケーションを利用して、リアルタイムデータ処理を実施した上で、Amazon Kinesis Data Firehoseに連携する。

B. Amazon Kinesis Data StreamsがデータをAmazon Managed Service for Apache Flinkアプリケーションによって処理してから、Amazon Kinesis Data Firehoseに連携する。

C. Amazon Kinesis Data Firehoseがデータを取得し、そのストリームデータをLambda関数がデータ処理を実施してから、Amazon S3バケットにデータを格納する。

D. API GatewayとLambda関数を統合したアプリケーションがデータ処理を実行して、Amazon Kinesis Data Firehoseに連携する。

E. Amazon Kinesis Data Firehoseがデータを取得し、そのストリームデータをAmazon Kinesis Data Firehoseがデータ処理を実施して、Amazon S3バケットにデータを格納する。

</details>

<details>
<summary>正解と解説を見る</summary>

### 正解
A、C

### 解説
このパイプラインは2つのステップで構成されます。

**ステップ1: リアルタイムデータ処理（A）**
- Kinesis Data Streamsアプリケーション（Kinesis Client Libraryを使用）でリアルタイムにセンサーデータを処理する
- 処理後のデータをKinesis Data Firehoseに連携する

**ステップ2: データ変換と保存（C）**
- Kinesis Data FirehoseはAWS Lambda関数と連携してデータ変換を実行できる
- Lambda関数がデータを最適な形式に変換してから、FirehoseがS3バケットに配信する

### 他の選択肢が不適切な理由
B
Amazon Managed Service for Apache Flinkはストリーミングデータのリアルタイムクエリ分析に使用するサービスです。今回の要件はクエリ分析ではなくデータ変換であるため過剰な構成です。

D
IoTセンサーのストリームデータ取得にAPI Gatewayは不要です。API GatewayはHTTPリクエストベースのAPIアクセスに使用するものであり、センサーからの継続的なストリームデータ取得にはKinesisが適しています。

E
Kinesis Data Firehose単独ではシンプルなデータ変換（フォーマット変換や圧縮）しかできません。ユーザー独自のカスタムされたデータ変換を実現するにはLambda関数を組み込む必要があります。

### まとめ
Kinesisエコシステムの典型的なパイプライン: Data Streams（リアルタイム収集・処理）→ Firehose + Lambda（データ変換・配信）→ S3（保存）。Data Streamsはリアルタイム処理、FirehoseはS3等への自動配信、Lambda関数はカスタムデータ変換を担う。

</details>

---