---
title: Rust + OpenTelemetryで分散トレーシング入門 ― ECS FargateからAWS X-Rayへ
tags:
  - Rust
  - AWS
  - OpenTelemetry
  - ECS
  - Jr.Champions
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
:::note info
この記事は「2026 Japan AWS Jr. Champions 真夏のQiitaリレー」の記事となります。
過去の投稿は以下のリンク集からご覧ください。
:::

https://qiita.com/ys-yoshida/items/6f7c7f85155a993e2c86

## はじめに

こんにちは。株式会社スリーシェイクの太田暢です。

今回は分散トレーシング入門として、RustのWebアプリからECS Fargate上のADOT Collectorを経由し、AWS X-Rayへトレースを送信してみます。

## ログ・メトリクス・トレース

システムの中で何が起きているかを外から把握できる状態を**オブザーバビリティ**（可観測性）と呼びます。その手がかりになる観測データがメトリクス・ログ・トレースです。

- **メトリクス**はCPU使用率や応答時間のような時系列の測定値です。一定の間隔で集計され、全体の傾向や急な変化が読み取れます。

- **ログ**は出来事を1行ずつ記録したものです。エラーの内容や処理の経過が発生時刻とともに残ります。

- **トレース**はリクエスト1回が通った経路と、各区間の所要時間の記録です。区間ひとつひとつを**Span**と呼び、同じtrace_idを持つSpanを集めると1本のトレースになります。Spanは入れ子にできるため、「リクエスト全体」の親Spanの中に「データベース照会」の子Spanを持たせる、という親子関係も表現できます。

今回はトレースに注目していきます。各Spanは開始時刻と所要時間を持つので、時間軸に沿って並べると1回のリクエストの内訳が見えます。次の図では認証に70 ms、データベース照会に550 ms、描画に90 msという配分になっており、どこで時間を使ったかが読み取れます。

![800 msのPOST /orderのSpanの下に、auth check 70 ms・db query 550 ms・render 90 msの子Spanが時間軸に沿って並んだ図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/44789001-328c-4acf-8c9e-79157d0a5c6b.png)

## OpenTelemetryとは

https://opentelemetry.io/ja/

OpenTelemetryはメトリクス・ログ・トレースなどの観測データを特定の監視サービスに依存せず扱うためのAPI、SDK、プロトコルなどを標準化・提供するオープンソースプロジェクトです。CNCFのgraduatedプロジェクトとして開発されています。

アプリケーションの動作を観測できるよう、Spanなどのテレメトリを生成する仕組みを組み込むことを**計装**（instrumentation）と呼びます。

ログ・メトリクス・トレースの3種類のデータはOpenTelemetryの用語で**シグナル**と呼ばれます。

計装には2通りのやり方があります。

- **手動計装** — Spanの生成や属性の付与を自分のコードで書きます。今回はSpanが作られて送信されるまでの仕組みを追うため、手動計装を使います。
- **自動計装** — フレームワーク向けの計装ライブラリやエージェントがSpanを作ります。RustではActix Webなどに[計装ライブラリ](https://opentelemetry.io/docs/languages/rust/libraries/)をミドルウェアとして追加する方法があります。

今回の構成では、Spanの作成からX-Rayへの送信までに、主に次の部品が関わります。

**API → SDK → Exporter → Collector → オブザーバビリティバックエンド（今回はX-Ray）**

この記事では、API・SDK・ExporterをRustのクレートとしてアプリケーションに組み込みます。Collectorはアプリケーションとは別のコンテナとして動き、受け取ったデータをX-Rayへ転送します。

- **API** — Spanの開始や終了など、アプリケーションから利用する操作を定義します。Rustでは`opentelemetry`クレートです。
- **SDK** — APIの呼び出しを受けてSpanを組み立て、送信まで一時保存します。`opentelemetry_sdk`クレートです。
- **Exporter** — SDKに一時保存されたSpanを送信用の形式に変換してCollectorへ送ります。今回は`opentelemetry-otlp`クレートです。
- **Collector** — 受け取ったデータを送り先に合わせて変換し、転送します。
- **オブザーバビリティバックエンド** — テレメトリを受け取り、保存・検索・可視化するシステムです。Webアプリケーションにおける「バックエンド」とは別の意味で、X-RayやDatadogなどが該当します。今回はX-Rayを使います。

## AWS X-Rayとは

X-RayはAWSのマネージド分散トレーシングサービスです。送られてきたトレースを保存し、検索と表示の画面を提供します。保持期間は30日で、保存の基盤を自前で運用せずに済みます。

X-Rayの画面はCloudWatchコンソールに統合されています。2026年8月時点では、トレース関連のメニューはCloudWatchの「Application Signals (APM)」の下にあります。AWSマネジメントコンソールで「X-Ray」を検索して開いても、このCloudWatchの画面へ移動します。

## 今回の構成

トレースはアプリケーションからX-Rayまで次の順で流れます。

1. アプリケーションがSpanを作る
2. Collectorが受け取り、X-Ray向けの形式へ変換して送る
3. X-Rayが保存し、コンソールやGrafanaから読む

![ECS FargateタスクのRustアプリからADOT Collector経由でX-Rayへトレースを送り、X-RayコンソールとローカルのGrafanaで読む構成](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/ca223abf-0cc6-49dc-9091-6ba581c7334d.png)

アプリからCollectorへは**OTLP**（OpenTelemetry Protocol）という共通フォーマットで送ります。OTLPにはgRPC版とHTTP版があり、今回使うのは既定ポートが4317番のgRPC版です。

Collectorには**ADOT**（AWS Distro for OpenTelemetry）を使います。ADOT CollectorはOSSのOpenTelemetry CollectorをベースにしたAWSのディストリビューションで、AWS X-RayなどAWSサービスへの送信に必要なコンポーネントを含んでいます。

Collectorの配置には主に次のような選択肢があります。

- ホストごとに1つ常駐させる
- Collector専用のサービスとして起動する
- アプリケーションと同じECSタスクにサイドカーとして配置する

今回は3つ目のサイドカー方式を使います。ECSのタスクは複数のコンテナをまとめて起動する単位で、同じタスク内のコンテナ同士はlocalhostで通信できます。Fargateではホスト上にCollectorをデーモンとして常駐させる構成が使えないため、[ADOTのドキュメント](https://aws-otel.github.io/docs/setup/ecs)で基本的な構成として紹介されているのもこの方式です。アプリとCollectorを同じタスクに入れ、localhostの4317番ポートでOTLP/gRPCのデータを受け渡します。

トレースを見る画面としては、X-RayコンソールとローカルのGrafanaの2つを用意します。

https://grafana.com/

Grafanaは登録したデータソースへの問い合わせ結果を表示するOSSの可視化ツールです。データソースにはX-Rayのほか、CloudWatchのメトリクスやログ、Prometheusも登録できるため、保存先がサービスごとに分かれていても確認画面をGrafanaに集約できます。今回はX-Rayを登録し、同じトレースをGrafanaからも確認します。

## 環境

- AWS CLI v2（検証用アカウントのIAM Identity Centerで認証するプロファイルを設定済み）
- Docker（Rancher DesktopまたはDocker Desktop）
- Terraform v1.5以降
- Apple SiliconのMac

Terraform自体には認証情報を書かず、AWS CLIと同じ認証チェーンを読みます。認証とアカウントの確認を先に済ませます。

```bash
aws sso login --profile <プロファイル名>
export AWS_PROFILE=<プロファイル名>
aws sts get-caller-identity
```

3つ目のコマンドで表示されるアカウントに対して`terraform apply`が実行されます。

リージョンも環境変数で揃えておきます。

main.tfのproviderにはap-northeast-1を指定しているため、Terraformが作成するリソースは東京リージョンに置かれます。一方、手順3以降で実行するAWS CLIはCLI側で設定された既定のリージョンを参照します。

両者がずれていると、Terraformで作成したECSクラスタをAWS CLIから見つけられず、`ClusterNotFoundException`になります。

```bash
export AWS_REGION=ap-northeast-1
```

## 1. Rustアプリケーションの作成

エンドポイントが3つだけの小さなHTTPサーバーを作ります。

- `/ok` — すぐ200を返します。
- `/slow` — 500 ms待ってから200を返します。
- `/error` — 500を返します。

計装として書き足すのはCargo.tomlの依存クレート、main関数の冒頭の初期化、各ハンドラの中でのSpanの開始と終了です。初期化ではSpanをどこへどう送るかを決め、ハンドラでは計測したい範囲をSpanで囲みます。

ファイル構成は次のとおりです。

```
rust-xray-handson/
├── app/
│   ├── Cargo.toml
│   ├── Dockerfile
│   ├── .dockerignore
│   └── src/
│       └── main.rs
└── terraform/
    └── main.tf
```

### Cargo.toml

```toml
[package]
name = "rust-xray-handson"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8.9"                        # Web フレームワーク
opentelemetry = "0.32.0"              # API。Span を作る操作はここから呼ぶ
opentelemetry-otlp = { version = "0.32.0", features = ["grpc-tonic"] }  # Exporter。OTLP/gRPC で送る
opentelemetry_sdk = "0.32.1"          # SDK。Span を一時保存して Exporter に渡す
tokio = { version = "1.53.1", features = ["full"] }  # 非同期ランタイム
```

`opentelemetry`・`opentelemetry_sdk`・`opentelemetry-otlp`は互換性のある同じマイナーバージョンで揃えます。ここでは3つとも0.32系です。

### src/main.rs

```rust
use std::time::Duration;

use axum::{http::StatusCode, routing::get, Router};
use opentelemetry::global::{self, BoxedTracer};
use opentelemetry::trace::{Span, Status, TraceContextExt, Tracer};
use opentelemetry::{Context, KeyValue};
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;

// main で登録した TracerProvider から Tracer を取り出す
fn tracer() -> BoxedTracer {
    global::tracer("rust-xray-handson")
}

async fn ok_handler() -> &'static str {
    // 属性の付け方を示す最小例。start 直後に end するため、HTTP処理全体は計測しない
    let mut span = tracer().start("GET /ok");
    // 属性は Span に付ける key-value。あとから絞り込む手がかりになる
    span.set_attribute(KeyValue::new("app.exercise", "first-trace"));
    span.end();
    "ok\n"
}

async fn slow_handler() -> &'static str {
    let tracer = tracer();
    let parent = tracer.start("GET /slow");
    // 親 Span を Context に入れ、その Context を渡して子 Span を作る
    let cx = Context::current_with_span(parent);
    let mut child = tracer.start_with_context("wait_backend", &cx);
    tokio::time::sleep(Duration::from_millis(500)).await;
    child.end();
    // 親は Context から取り出して終える
    cx.span().end();
    "slow\n"
}

async fn error_handler() -> (StatusCode, &'static str) {
    let mut span = tracer().start("GET /error");
    // エラーの印を Span に付ける。バックエンド側でエラーとして表示される
    span.set_status(Status::error("simulated failure"));
    span.end();
    (StatusCode::INTERNAL_SERVER_ERROR, "error\n")
}

// ECS はタスク停止時にまず SIGTERM を送る。これを待ってから shutdown へ進む
async fn shutdown_signal() {
    use tokio::signal::unix::{signal, SignalKind};

    let mut term = signal(SignalKind::terminate()).expect("SIGTERM ハンドラを登録できない");
    let mut int = signal(SignalKind::interrupt()).expect("SIGINT ハンドラを登録できない");

    tokio::select! {
        _ = term.recv() => {}
        _ = int.recv() => {}
    }
    println!("shutdown signal received");
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    // Exporter: 溜まった Span を OTLP/gRPC で送り出す。
    // エンドポイント未指定時は http://localhost:4317 に送る。
    // 同一タスク内のサイドカーとは localhost で通信できる
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .build()?;
    // SDK: Span をバッチにまとめ、service.name などを全 Span に付けて Exporter へ渡す
    let provider = SdkTracerProvider::builder()
        .with_batch_exporter(exporter)
        .with_resource(
            Resource::builder()
                .with_service_name("rust-xray-handson")
                .build(),
        )
        .build();
    // 以降 global::tracer() でこの provider の Tracer を取り出せる
    global::set_tracer_provider(provider.clone());

    let app = Router::new()
        .route("/ok", get(ok_handler))
        .route("/slow", get(slow_handler))
        .route("/error", get(error_handler));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    println!("listening on 0.0.0.0:8080");
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    // ここまで到達して初めてバッチに残った Span が送り切られる
    provider.shutdown()?;
    println!("tracer provider shut down");
    Ok(())
}
```

`/ok`は属性を1つ付けたSpanを作ります。`/error`もSpanは1つで、こちらにはエラー状態を付けます。`/slow`はリクエスト全体の親Spanの中に待ち処理の子Spanを作るので、1リクエストでSpanが親子2つになります。

![/ok は属性つきのSpanが1つ、/slow は親子2つ、/error はエラーの印が付いたSpanが1つ生成されることを示した図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/02b94139-1359-4d21-875b-55b3a8efe62a.png)

ECS停止時にバッチへ残ったSpanを送り切るため、SIGTERMを受けて`provider.shutdown()`を呼びます。後半で実際にタスクを停止して確認します。

この構成で使うSDKの既定サンプラーは`ParentBased(AlwaysOn)`で、親Spanの判断を引き継ぎ、親がない場合はすべて記録します。検証では全件を送りますが、X-Rayは記録したトレース数に応じて課金されるため、本番ではトラフィック量に合わせてサンプラーを設定します。

### Dockerfile

```dockerfile
FROM rust:1-slim AS builder
WORKDIR /build
COPY . .
RUN cargo build --release

FROM gcr.io/distroless/cc-debian12
COPY --from=builder /build/target/release/rust-xray-handson /rust-xray-handson
EXPOSE 8080
ENTRYPOINT ["/rust-xray-handson"]
```

`.dockerignore`には`target`とだけ書いておきます。

## 2. Terraformでデプロイする

次のような構成にしています。詳細は[こちらのリポジトリ](https://github.com/Ota1022/rust-xray-handson/blob/main/terraform/main.tf)をご覧ください。

- NAT GatewayとALBを作らない。パブリックサブネットにタスクを直接置いて費用を抑える
- タスクロールに`AWSXRayDaemonWriteAccess`と`CloudWatchAgentServerPolicy`を付ける。X-RayへはアプリではなくADOT Collectorが書き込むため、権限もタスクロール側に付ける
- ECRリポジトリに`force_delete = true`を指定。イメージが残っていても`terraform destroy`でリポジトリごと消える

![ECRリポジトリ・VPC内のECSサービスとタスク・X-Rayの関係と、実行ロールとタスクロールの使い分けを示した図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/f146e3b4-cdfc-40b6-abe4-2f1414926fd1.png)

アプリとADOT Collectorは1つのタスク定義にまとめます。

![1つのECSタスク定義にappとadot-collectorの2コンテナが入り、localhost:4317のOTLP/gRPCでつながることを示した図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/a192193c-bb86-41cd-8149-6c35af7bd489.png)

CloudWatch Logsのログ設定だけ省いたタスク定義です。

```hcl
# Apple Silicon でビルドした arm64 イメージをそのまま動かす
resource "aws_ecs_task_definition" "main" {
  family                   = local.name
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "ARM64"
  }

  container_definitions = jsonencode([
    {
      name         = "app"
      image        = "${aws_ecr_repository.app.repository_url}:latest"
      essential    = true
      portMappings = [{ containerPort = 8080, protocol = "tcp" }]
    },
    {
      name      = "adot-collector"
      image     = "public.ecr.aws/aws-observability/aws-otel-collector:latest"
      essential = true
      command   = ["--config=/etc/ecs/ecs-default-config.yaml"]
    }
  ])
}
```

Collector側の設定は`command`で指定している[ecs-default-config.yaml](https://github.com/aws-observability/aws-otel-collector/blob/main/config/ecs/ecs-default-config.yaml)だけです。ADOTイメージに同梱されたECS向けのデフォルト設定で、OTLPを4317番で受けたトレースをX-Rayへ送るパイプラインが書かれています。Collector側にportMappingsがないのは、アプリからの4317番が同じタスク内のlocalhost通信で届き、タスクの外へ開ける必要がないからです。

設定のうち、今回関係する部分を抜粋すると次のようになります。トレースは`otlp` Receiverから`awsxray` Exporterへ流れます。メトリクス用には`awsemf` Exporterも定義されており、タスクロールに`CloudWatchAgentServerPolicy`を付けているのはこのデフォルト設定に対応するためです。

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  awsxray:
  awsemf:

service:
  pipelines:
    traces:
      receivers: [otlp, awsxray]
      exporters: [awsxray]
    metrics:
      receivers: [otlp, statsd]
      exporters: [awsemf]
```

この例では検証を簡単にするため、ADOT Collectorに`:latest`タグを使っています。本番では更新による挙動の変化を避けるため、検証済みのバージョンまたはイメージダイジェストに固定します。

`runtime_platform`はApple Siliconで作成したイメージに合わせて`ARM64`にしています。x86_64環境でビルドする場合は`X86_64`に変更します。

applyは2回に分けます。ECSのタスク定義はECR上のイメージを参照するため、イメージをpushする前にサービスまで作るとタスクが起動に失敗し続けます。そこで`-target`でECRリポジトリだけ先に作ります。

```bash
cd rust-xray-handson/terraform
terraform init
terraform apply -target=aws_ecr_repository.app
```

`-target`を指定するとTerraformから警告が表示されますが、ここではイメージの保存先だけを先に作るために意図して使用しています。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REPO="$ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/rust-xray-handson"
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin "${REPO%/*}"
docker build -t "$REPO:latest" ../app
docker push "$REPO:latest"
```

イメージが入ったら残りをまとめて作ります。

```bash
terraform apply
```

## 3. curlでトレースを作る

Spanはリクエストを処理したときに作られるので、リクエストを送ってトレースを発生させます。

ALBを置いていない構成のため、送り先にはタスクのパブリックIPを直接使います。まずIPを調べます。

検証用に8080番のインバウンド通信を許可しますが、セキュリティグループのソースは自分のグローバルIPアドレス（`curl ifconfig.me`などで確認）を`/32`で指定してください。`0.0.0.0/0`には公開しません。

```bash
TASK_ARN=$(aws ecs list-tasks --cluster rust-xray-handson --query 'taskArns[0]' --output text)
ENI_ID=$(aws ecs describe-tasks --cluster rust-xray-handson --tasks "$TASK_ARN" \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
aws ec2 describe-network-interfaces --network-interface-ids "$ENI_ID" \
  --query 'NetworkInterfaces[0].Association.PublicIp' --output text
```

出てきたIPに対して3つのエンドポイントへ数回ずつリクエストを送ります。

```bash
IP=<上で出たIP>
curl http://$IP:8080/ok
curl http://$IP:8080/slow
curl http://$IP:8080/error
```

## 4. CloudWatchコンソールで見る

X-Rayに届いたトレースを見にいきましょう。AWSコンソールでCloudWatchを開き、左メニューの「Application Signals (APM)」から「トレースマップ」を選びます。Spanはバッチでまとめて送られるため、反映までに1分程度かかります。

![X-RayのトレースマップにクライアントとGET /okのノードが表示され、両者が100% OKの線で結ばれている画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/5f1b36ab-46e2-4076-a9b0-e1585ffa2463.png)

自分で書いたSpanがSDKからサイドカーを通ってマネージドサービスまで届いています。エラーを返したほうは赤いノードです。

![X-RayのトレースマップでGET /errorのノードが赤く表示され、障害100%と示されている画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/531a48b1-7684-42a8-ac6d-498767394c78.png)

次に、コードで設定したSpan名・属性・エラー状態がX-Ray上でどのように表現されているかを確認します。`batch-get-traces`にはトレースIDが必要なので、直近10分のトレースから1件取得します。次の`date`はmacOSの書式です。

```bash
TRACE_ID=$(aws xray get-trace-summaries \
  --start-time "$(date -v-10M +%s)" --end-time "$(date +%s)" \
  --query 'TraceSummaries[0].Id' --output text)
aws xray batch-get-traces --trace-ids "$TRACE_ID"
```

セグメントの中身は次のように対応しています。

| コードで書いたもの | X-Ray上の位置 |
| --- | --- |
| `KeyValue::new("app.exercise", "first-trace")` | `metadata.default.app.exercise` |
| `Status::error("simulated failure")` | `fault: true`と`cause.exceptions[0].message` |
| 子Spanの`wait_backend` | 親セグメントの`subsegments[]`（実測500.7ms） |
| `with_service_name("rust-xray-handson")` | `metadata.default.otel.resource.service.name` |

ADOT Collectorが変換したことはセグメントの`aws.xray`から分かります。

```json
"aws": { "xray": { "sdk": "opentelemetry for rust", "sdk_version": "0.32.1", "auto_instrumentation": false } }
```

属性はAnnotationsではなくMetadataに入ります。X-Rayはこの2つを区別していて、フィルタ式で検索できるのはAnnotationsだけです。`app.exercise = "first-trace"`でトレースを絞り込む、という使い方はこのままではできません。検索したい属性はCollector側でawsxrayエクスポーターの`indexed_attributes`に挙げ、Annotationsへ昇格させます。

今回のサービスマップでは、Span名がノード名になるため`GET /ok`・`GET /slow`・`GET /error`の3つに分かれます。`service.name`はmetadataとして保存され、このノード名には使われません。

## 自動計装との差分

ここまでのSpanはすべて手動で作成しています。では、自動計装を使った場合は何が変わるのでしょうか。

OpenTelemetry公式デモのRustサービス[`src/shipping`](https://github.com/open-telemetry/opentelemetry-demo/tree/main/src/shipping)と比較します。公式デモもこの記事と同じOpenTelemetry 0.32系を使用しています。

| 観点 | この記事（手書き） | 公式デモ（自動計装） |
| --- | --- | --- |
| 計装 | ハンドラごとにSpanを手書き | `opentelemetry-instrumentation-actix-web`のミドルウェア |
| 応答時間 | `span.end()`の位置しだいで0になる。`/ok`はX-Ray上2.1 µs（実測の応答は約20 ms） | リクエスト全体が1つのSpanになる |
| HTTP属性 | 付けていない。トレース一覧のメソッド・ステータスの列が空になる | セマンティック規約に沿って自動で付く |
| Propagator | 設定なし | `TraceContextPropagator`を明示設定 |
| Resource | `service_name`のみ | Host / OS / ProcessのDetectorで自動収集 |
| シグナル | トレースのみ | トレース・ログ・メトリクスの3本 |
| 終了処理 | `provider.shutdown()`を1回 | tracer → logger → meterの順に3つ |

自動計装は少ない実装でリクエスト全体を継続的に計測でき、HTTP属性も一定の形式で揃います。手動計装はSpanを作るコードが必要になる一方、計測する区間や属性を用途に合わせて決められます。どちらか一方を選ぶだけでなく、自動計装でリクエスト境界を捉え、`wait_backend`のような内部処理を手動計装で補う構成も取れます。

## ECSタスク停止時に未送信のSpanを送る

`with_batch_exporter`を使うと生成したSpanはメモリに一時保存され、既定で5秒ごとにまとめて送信されます。そのためECSタスクが停止するタイミングによっては、未送信のSpanがアプリケーション内に残ることがあります。

未送信のSpanを送ってから終了するには次の順序で処理します。

1. ECSから送られるSIGTERMを受け取る
2. Axumサーバーを終了する
3. `provider.shutdown()`を呼び出す
4. Spanの送信が完了してからプロセスを終了する

`main.rs`では、`with_graceful_shutdown`にSIGTERMを待つ`shutdown_signal()`を渡しています。

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await?;

provider.shutdown()?;
```

SIGTERMを受け取ると`shutdown_signal()`が完了し、`axum::serve`から次の行へ進みます。そこで`provider.shutdown()`が呼ばれ、バッチに残っているSpanが送信されます。

ECSはSIGTERMを送ったあと既定では30秒後にSIGKILLでプロセスを強制終了します。終了処理はこの時間内に完了させます。

![SIGTERMからSIGKILLまでの30秒の区間で、with_graceful_shutdownなしはその場で終了して未送信のSpanが消え、ありは残りを送り切ってから終了することを示した図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/e8c6c0cb-91d2-4cd8-ac1e-0eae8448154d.png)

### ECS上での確認

停止直前に`/ok`を3回呼び出し、バッチに未送信のSpanがある状態でECSタスクを停止しました。

| 時刻 | コンテナ | 出力 |
| --- | --- | --- |
| 12:19:57.971 | app | `shutdown signal received` |
| 12:19:57.972 | app | `tracer provider shut down` |
| 12:19:57.998 | adot-collector | `Received signal from OS` |
| 12:19:57.999 | adot-collector | `Shutdown complete.` |

アプリケーションの終了処理が実行され、停止直前の3リクエストもX-Rayに記録されていました。

この構成ではアプリケーションの送信完了からCollectorの終了まで27 msしかありませんでした。また、コンテナ間の依存関係を指定していないため、この停止順序は保証されません。

本番で同じサイドカー構成を使う場合は、タスク定義でアプリケーションからCollectorへの[コンテナ依存関係](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definition_dependson)を指定します。ECSは起動時の依存関係を停止時には逆順で適用するため、アプリケーションを停止してからCollectorを停止できます。あわせて`stopTimeout`を指定し、`provider.shutdown()`が完了するまでの時間を確保します。

```hcl
dependsOn  = [{ containerName = "adot-collector", condition = "START" }]
stopTimeout = 60
```

`stopTimeout`の既定値は30秒で、Fargateでは最大120秒です。

## X-RayのトレースをGrafanaから見る

同じトレースをGrafanaからも確認します。

Grafana自体はローカルのDockerコンテナで動かします。

AWS Application Signalsデータソースプラグインv2.17.1の追加画面では、「X-Ray」と検索しても表示されません。「Application Signals」で検索してください。プラグインのIDは`grafana-x-ray-datasource`のままですが、表示名は「AWS Application Signals」に変わっています。

![Grafanaのデータソース追加画面で「Application Signals」と検索し、AWS Application Signalsが1件表示されている画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/44835312-6466-47bd-8ebf-0153f37a48b9.png)

### Grafanaを起動する

GrafanaのX-RayデータソースにはAWS認証情報が必要です。ここでは、事前に設定したIAM Identity Centerのプロファイルから一時クレデンシャルを取得し、「Access & secret key」方式でデータソースに渡します。

一時クレデンシャルでは、アクセスキーとシークレットキーに加えてセッショントークンも必要です。プロビジョニングファイルの`secureJsonData`に3つの環境変数を指定します。

```yaml
# provisioning/datasources/xray.yaml
apiVersion: 1
datasources:
  - name: X-Ray
    type: grafana-x-ray-datasource
    isDefault: true
    jsonData:
      authType: keys
      defaultRegion: ap-northeast-1
    secureJsonData:
      accessKey: $AWS_ACCESS_KEY_ID
      secretKey: $AWS_SECRET_ACCESS_KEY
      sessionToken: $AWS_SESSION_TOKEN
```

`aws configure export-credentials`で現在のプロファイルから認証情報を取得し、Grafanaコンテナへ渡します。

```bash
eval "$(aws configure export-credentials --format env)"
docker run -d --rm --name grafana-xray -p 127.0.0.1:3001:3000 \
  -e GF_PLUGINS_PREINSTALL=grafana-x-ray-datasource@2.17.1 \
  -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e AWS_SESSION_TOKEN \
  -v "$(pwd)/provisioning:/etc/grafana/provisioning:ro" \
  grafana/grafana:13.2.0
```

ポートは`127.0.0.1:3001`にだけ公開し、AWS認証情報を持つGrafanaコンテナに他の端末からアクセスできないようにしています。初回起動ではプラグインのダウンロードに30秒ほどかかります。

一時クレデンシャルの有効期限が切れた場合は、`aws sso login`と`eval "$(aws configure export-credentials --format env)"`を再実行します。その後、既存のGrafanaコンテナを削除し、上の`docker run`コマンドで作り直します。コンテナを再起動するだけでは、作成時に渡した環境変数は更新されません。

### Grafanaでトレースを見る

`http://127.0.0.1:3001`を開き、admin / adminでログインします。X-Rayデータソースはプロビジョニングによって登録されています。

Exploreを開き、Query Typeに「Trace List」、期間に「Last 1 hour」を指定すると、X-Rayに保存されたトレースの一覧を確認できます。

![GrafanaのTrace Listに19件のトレースが並び、Response Time列に501 msと0 sが混在している画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/92a07c40-09e5-44fb-b984-0ececb42745c.png)

Method・Response・URL・Client IPの列がすべて空です。「自動計装との差分」で見たとおり、手動計装でHTTP関連のセマンティック規約属性を付けていないと一覧はこのように表示されます。

一覧からTrace IDを開くとタイムラインになります。

![Grafanaのタイムラインで、GET /slowの下にwait_backendが500.67 msの帯として入れ子で表示されている画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/a057cd35-fd3a-4c34-9221-46fe99eace6e.png)

`GET /slow`の下に`wait_backend`が入れ子になり、500.67 msを占めています。コードで`start_with_context`を使って親子にしたものがそのままこの形で出てきます。この表示ではGrafanaがセグメントの上にルートSpanを1つ追加するため、Span数は3になります。

`/error`のトレースも確認します。

![Grafanaのタイムラインで、GET /errorのSpanに赤いエラーアイコンが付いている画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4244797/fd5d65df-881c-497a-9bab-948f95983ee5.png)

`Status::error`を付けたSpanには赤い印が付きます。X-Rayのセグメントで`fault: true`になっていたものがGrafanaではこの表示になります。

## かかった費用

この検証で継続的に発生する費用の大半はFargateタスクとパブリックIPv4アドレスによるものです。これらに加えて、ECRのイメージ保存も使用量に応じて課金されます。

| 項目 | 内訳（東京リージョン） | 時間あたり |
| --- | --- | --- |
| Fargate ARM（vCPU） | 0.25 vCPU × $0.04045/h | $0.01011 |
| Fargate ARM（メモリ） | 0.5 GB × $0.00442/GB-h | $0.00221 |
| パブリックIPv4 | 1個 × $0.005/h | $0.00500 |
| 合計 | | $0.01732 |

1 USD = 150円で換算すると約2.6円/時で、4時間動かすと約10円、12時間では約31円です。今回のX-Ray、CloudWatch Logs、データ転送の使用量は小さいため、アカウント内の他用途を含めて無料枠の上限を超えなければ追加料金は発生しません。

単価はAWS Price List APIから取得した実測値です。パブリックIPv4を除いてFargate料金だけで見積もると、実際の合計より約29%安くなります。

Grafanaでの確認まで終えたら、検証環境を削除します。X-Rayのトレースは30日間保持されるため、削除後も確認できます。

```bash
terraform destroy
```

## おわりに

今回はRustアプリケーションでOpenTelemetryのSpanを手動で作成し、ECS Fargate上のADOT Collectorを経由してX-Rayへ送信しました。

どこをSpanとして切り出しどの情報を持たせるかによってX-RayやGrafanaでの見え方は変わります。今後はトレースを送る仕組みに加えて計装そのものの設計についても知見を深めていきたいです。

最後までお読みいただきありがとうございました。

## 参考

- [OpenTelemetry: 用語と概念](https://opentelemetry.io/docs/concepts/)
- [OpenTelemetry: Rustのライブラリを使った計装](https://opentelemetry.io/docs/languages/rust/libraries/)
- [opentelemetry-rust](https://github.com/open-telemetry/opentelemetry-rust)
- [opentelemetry-demo の shipping サービス](https://github.com/open-telemetry/opentelemetry-demo/tree/main/src/shipping)（Rust実装の参考）
- [ADOT: ECSでのセットアップ](https://aws-otel.github.io/docs/setup/ecs)
- [aws-otel-collector](https://github.com/aws-observability/aws-otel-collector)
- [AWS X-Ray 開発者ガイド](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
- [Grafana X-Rayデータソースプラグイン](https://grafana.com/grafana/plugins/grafana-x-ray-datasource/)
