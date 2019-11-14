---
aliases:
  - /ja/integrations/wmi
assets:
  dashboards: {}
  monitors: {}
  service_checks: assets/service_checks.json
categories:
  - monitoring
creates_events: false
ddtype: check
dependencies:
  - 'https://github.com/DataDog/integrations-core/blob/master/wmi_check/README.md'
display_name: WMI
git_integration_title: wmi_check
guid: d70f5c68-873d-436e-bddb-dbb3e107e3b5
integration_id: wmi
integration_title: WMI チェック
is_public: true
kind: integration
maintainer: help@datadoghq.com
manifest_version: 1.0.0
metric_prefix: wmi.
name: wmi_check
public_title: Datadog-WMI チェックインテグレーション
short_description: WMI メトリクスを収集してグラフを作成
support: コア
supported_os:
  - windows
---
![WMI メトリクス][1]

## 概要

WMI (Windows Management Instrumentation) で Windows アプリケーション/サーバーからメトリクスをリアルタイムに取得して、以下のことができます。

* アプリケーションのパフォーマンスを視覚化できます。
* アプリケーションのアクティビティを他のアプリケーションと関連付けることができます。

**注:** オーバーヘッドが大幅に少なく、したがってスケーラビリティに優れるため、どの場合も代わりに [PDH チェック][14]を使用することをお勧めします。

## セットアップ
### インストール

Microsoft Windows および他のパッケージアプリケーションから標準メトリクスのみを収集している場合、インストール手順はありません。新しいメトリクスを定義してアプリケーションから収集する必要がある場合は、いくつかオプションがあります。

1.  .NET の System.Diagnostics を使用してパフォーマンスカウンターを送信し、次に WMI からそれらにアクセスします。
2.  アプリケーションで COM ベースの WMI プロバイダーを実装します。通常、これは、.NET 以外の言語を使用している場合にのみ行います。

System.Diagnostics の詳細については、[MSDN のドキュメント][2]を参照してください。メトリクスを追加した後に、WMI でそれを検索できる必要があります。WMI ネームスペースを参照するには、このツール [WMI Explorer][3] が役立ちます。[こちら][4]の Powershell で同じ情報を見つけることもできます。また、[Datadog ナレッジベースの記事][5]で情報を確認してください。

新しいメトリクスに My_New_Metric というカテゴリを割り当てる場合、WMI パスは 
`\\<ComputerName>\ROOT\CIMV2:Win32_PerfFormattedData_My_New_Metric` になります

メトリクスが WMI に表示されない場合は、`winmgmt /resyncperf` を実行して、WMI に強制的にパフォーマンスライブラリを登録してみてください。


## コンフィグレーション

1.  WMI インテグレーションタイルの **Install Integration** ボタンをクリックします。
2.  Windows サーバーで Datadog Agent Manager を開きます。
3.  `Wmi Check` 構成を編集します。

        init_config:

        instances:
          - class: Win32_OperatingSystem
            metrics:
              - [NumberOfProcesses, system.proc.count, gauge]
              - [NumberOfUsers, system.users.count, gauge]

          - class: Win32_PerfFormattedData_PerfProc_Process
            metrics:
              - [ThreadCount, proc.threads.count, gauge]
              - [VirtualBytes, proc.mem.virtual, gauge]
              - [PercentProcessorTime, proc.cpu_pct, gauge]
            tag_by: Name

          - class: Win32_PerfFormattedData_PerfProc_Process
            metrics:
              - [IOReadBytesPerSec, proc.io.bytes_read, gauge]
            tag_by: Name
            tag_queries:
              - [IDProcess, Win32_Process, Handle, CommandLine]

<div class="alert alert-info">
デフォルトの構成は、フィルター節を使用して、プルされるメトリクスを制限しています。フィルターに有効な値を設定するか、上のようにフィルターを削除して、メトリクスを収集してください。
</div>

メトリクス定義は、次の 3 つの要素からなります。

* WMI のクラスプロパティ
* Datadog に表示されるメトリクス名
* メトリクスタイプ

#### 構成オプション
この機能は、バージョン 5.3 の Agent から使用できます。

各 WMI クエリには、2 つの必須オプション `class` と `metrics`、および 6 つの任意指定オプション `host`、`namespace`、`filters`、`provider`、`tag_by`、`constant_tags`、`tag_queries` があります。

* `class` は、`Win32_OperatingSystem`、`Win32_PerfFormattedData_PerfProc_Process` などの WMI クラスの名前です。標準クラス名の多くは、[MSDN ドキュメント][6]に記載されています。`Win32_FormattedData_*` クラスは、デフォルトで多数の有用なパフォーマンスカウンターを提供しています。

* `metrics` は、キャプチャするメトリクスのリストです。リスト内の各項目は、
`[<WMI_PROPERTY_NAME>, <METRIC_NAME>, <METRIC_TYPE>]` 形式のセットです。
  *  `<WMI_PROPERTY_NAME>` は、`NumberOfUsers` や `ThreadCount` などです。標準プロパティは、各クラスの MSDN ドキュメントにも記載されています。
  *  `<METRIC_NAME>` は、Datadog に表示する名前です。
  * `<METRIC_TYPE>` は、gauge、rate、histogram、counter などの標準的な Agent チェックタイプです。

* `host` は、WMI クエリのターゲットです (オプション)。デフォルトでは、`localhost` が想定されます。このオプションを設定する場合は、ターゲットホストで Remote Management が有効になっていることを確認してください。詳細については、[こちらをご覧ください][7]。

* `namespace` は、接続先の WMI ネームスペースです (オプション)。デフォルトは `cimv2` です。

* `filters` は、WMI クエリに対するフィルターのリストです。たとえば、プロセスベースの WMI クラスの場合は、マシン上で実行されている特定のプロセスのメトリクスだけが必要なことがあります。この場合は、各プロセス名に対応するフィルターを追加します。ワイルドカードとして '%' 文字を使用することもできます。

* `provider` は、WMI プロバイダーです (オプション)。デフォルトは `32` (Datadog Agent 32 ビットの場合) または `64` です。これは、デフォルト以外のプロバイダーに WMI データをリクエストするために使用されます。使用できるオプションは `32` または `64` です。
詳細については、[MSDN][8] をご覧ください。

* `tag_by` は、使用している WMI クラスのプロパティで各メトリクスをタグ付けします (オプション)。これは、WMI クエリに複数の値がある場合にのみ役立ちます。

* `tags` は、固定値のセットで各メトリクスをタグ付けします (オプション)。

* `tag_queries` は、クエリのリストを指定して、ターゲットクラスのプロパティでメトリクスをタグ付けします (オプション)。リスト内の各項目は、`[<LINK_SOURCE_PROPERTY>, <TARGET_CLASS>, <LINK_TARGET_CLASS_PROPERTY>, <TARGET_PROPERTY>]` 形式のセットです。
  * `<LINK_SOURCE_PROPERTY>` は、リンク値です。
  * `<TARGET_CLASS>` は、リンク先のクラスです。
  * `<LINK_TARGET_CLASS_PROPERTY>` は、リンク先のターゲットクラスのプロパティです。
  * `<TARGET_PROPERTY>` は、タグ付けに使用する値です。

  これは、WMI クエリ
  `SELECT '<TARGET_PROPERTY>' FROM '<TARGET_CLASS>' WHERE '<LINK_TARGET_CLASS_PROPERTY>' = '<LINK_SOURCE_PROPERTY>'` に変換されます。

<div class="alert alert-info">
これを設定すると、name:process#1 => name:process のように tag_by 値からインスタンス番号が削除されます。
</div>

#### メトリクスの収集
WMI チェックでは[カスタムメトリクス][9]を送信することができますが、これはお客様の[課金][10]に影響します。

### 検証

[Agent の `status` サブコマンドを実行][11]し、Checks セクションで `wmi_check` を探します。

## 収集データ
### メトリクス
このインテグレーションによって提供されるメトリクスのリストについては、[metadata.csv][12] を参照してください。

### イベント
WMI チェックには、イベントは含まれません。

### サービスのチェック
WMI チェックには、サービスのチェック機能は含まれません。

## トラブルシューティング
ご不明な点は、[Datadog のサポートチーム][13]までお問合せください。

[1]: https://raw.githubusercontent.com/DataDog/integrations-core/master/wmi_check/images/wmimetric.png
[2]: https://msdn.microsoft.com/en-us/library/system.diagnostics.performancecounter(v=vs.110.aspx)
[3]: https://wmie.codeplex.com
[4]: https://msdn.microsoft.com/en-us/powershell/scripting/getting-started/cookbooks/getting-wmi-objects--get-wmiobject-
[5]: https://docs.datadoghq.com/ja/integrations/faq/how-to-retrieve-wmi-metrics
[6]: https://msdn.microsoft.com/en-us/library/windows/desktop/aa394084.aspx
[7]: https://technet.microsoft.com/en-us/library/Hh921475.aspx
[8]: https://msdn.microsoft.com/en-us/library/aa393067.aspx
[9]: https://docs.datadoghq.com/ja/developers/metrics/custom_metrics
[10]: https://docs.datadoghq.com/ja/account_management/billing/custom_metrics
[11]: https://docs.datadoghq.com/ja/agent/guide/agent-commands/?tab=agentv6#agent-status-and-information
[12]: https://github.com/DataDog/integrations-core/blob/master/wmi_check/metadata.csv
[13]: https://docs.datadoghq.com/ja/help
[14]: https://docs.datadoghq.com/ja/integrations/pdh_check/


{{< get-dependencies >}}