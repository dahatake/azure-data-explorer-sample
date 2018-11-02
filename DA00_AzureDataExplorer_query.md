# Azure Data Explorer Tutrial

気象サンプルデータを使ったサンプルです。基本的なクエリやレンダリング手法を理解します。

## 事前準備

- Azure Data Explorer のクラスターを作成

  https://docs.microsoft.com/ja-jp/azure/data-explorer/create-cluster-database-portal

## コード実行前に

以下のチュートリアルがおススメ。

  - クイック スタート: Azure Data Explorer でデータのクエリを実行する:
  
    https://docs.microsoft.com/ja-jp/azure/data-explorer/web-query-data


## コード
Azure Portal もしくは Azure Data Explorer の Poral 上に下記コードを投入します。クエリ文字列を選択せずとも、[Shift] + [Enter] でクエリ単位で実行できます。


  - Azure Data Explorer Web UI: 

    https://dataexplorer.azure.com

```sql
.create table StormEvents (StartTime: datetime, EndTime: datetime, EpisodeId: int, EventId: int, State: string, EventType: string, InjuriesDirect: int, InjuriesIndirect: int, DeathsDirect: int, DeathsIndirect: int, DamageProperty: int, DamageCrops: int, Source: string, BeginLocation: string, EndLocation: string, BeginLat: real, BeginLon: real, EndLat: real, EndLon: real, EpisodeNarrative: string, EventNarrative: string, StormSummary: dynamic)

// Ingest Sample Data
.ingest into table StormEvents h'https://kustosamplefiles.blob.core.windows.net/samplefiles/StormEvents.csv?st=2018-08-31T22%3A02%3A25Z&se=2020-09-01T22%3A02%3A00Z&sp=r&sv=2018-03-28&sr=b&sig=LQIbomcKI8Ooz425hWtjeq6d61uEaq21UVX7YrM61N4%3D' with (ignoreFirstRecord=true)

// TableのDataを複製😊
// 20回くらい実行😎
.append StormEvents <| StormEvents

// 1) Simple

// 件数
StormEvents | count

StormEvents | take 100

StormEvents
| where StartTime >= datetime(2007-11-01) and StartTime < datetime(2007-12-01)
| where State == "FLORIDA"  
| count

// 集計
StormEvents
| summarize event_count=count(), mid = avg(BeginLat) by State
| sort by mid
| where event_count > 1800
| project State, event_count
| render columnchart

StormEvents
| extend hour= floor( StartTime % 1d , 1h)
| where State in ("GULF OF MEXICO","MAINE","VIRGINIA","WISCONSIN","NORTH DAKOTA","NEW JERSEY","OREGON")
| summarize event_count=count() by hour, State
| render timechart

// 2. 少し高度なもの

// ネスト
StormEvents
| top-nested 2 of State by sum(BeginLat),
top-nested 3 of Source by sum(BeginLat),
top-nested 1 of EndLocation by sum(BeginLat)

// 時系列
StormEvents
| make-series n=count() default=0 on StartTime in range(datetime(2007-01-01), datetime(2007-03-31), 1d) by State

StormEvents
| make-series n=count() default=0 on StartTime in range(datetime(2007-01-01), datetime(2007-03-31), 1d) by State
| extend series_stats(n)
| top 3 by series_stats_n_max desc
| render timechart

// Join
let LightningStorms =
StormEvents
| where EventType == "Lightning";
let AvalancheStorms =
StormEvents
| where EventType == "Avalanche";
LightningStorms
| join (AvalancheStorms) on State
| distinct State
StormEvents
| where State == "WASHINGTON" and StartTime >= datetime(2007-07-01) and StartTime <= datetime(2007-07-31)
| summarize StormCount = count() by EventType
| render piechart

```

Reference:
https://docs.microsoft.com/ja-jp/azure/data-explorer/write-queries#overview-of-the-query-language
