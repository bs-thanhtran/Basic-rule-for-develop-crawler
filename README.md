# Basic-rule-for-develop-crawler
[Hogetic] Basic rule for develop crawler

# Bối cảnh

各クローラーで同じ処理が異なるコードに配置されているため、コードレビューの手間がかかり、品質が確保できない問題を解決したい。

標準的な機能配置と開発ルールを定めることで、開発・レビューの効率化を図る。

# Khái quát

Tập hợp như sau

- 各モジュール(cmd, pkg, lib)の責務
- ページネーションとチャネル送信のタイミング

# テンプレートを使用したクローラ開発

テンプレートを使用して開発する場合は、下記の「各モジュールの責務」以降の部分は適宜読み替える。

[テンプレートを使用したクローラ開発](https://www.notion.so/65bbcc69654443849f463dc62cdc4bed?pvs=21)

# 各モジュールの責務

## cmd

- configの読み取り
    - 共通コンポーネントReadInputConfigを使用します。
- BQInput(, Google SpreadSheet)の場合のデータの取得(確認中)
- periodsの設定
    - 共通コンポーネントのGetPeriodを使用します。
- BQの設定
    - config, periodsなどを元にパーティションなどを設定します。
- BQのスキーマの定義

## pkg

- request paramの設定
    - partitionで使用しているtimezoneとAPIリクエストに使用するtimezoneが異なる場合は、ここで変換します。
- iteratorの取得
- iteratorを使用してのAPIリクエスト
    - ページネーション処理が入る場合は、メモリ使用量を抑えるために1ページごとにチャネルに流して処理をします。
    - ページネーション処理自体はAPIによって実装方法が変わりますが、API固有のロジックはpkgではなくlibに実装しpkgからは隠蔽します。
    - どのAPIでもpkgではiteratorの同じメソッドを使用して呼び出せるようにします。
- BQ用にデータ整形
    - リクエストパラメータとレスポンスを使用してBQのデータ形式に沿うように整形します

## lib

- iteratorの定義
    - iteratorを返す関数を実装し、iteratorでのデータ取得はpkg/collectから行います。
    - iteratorの処理自体をlib内で呼び出してしまうと、大量のページネーションがある場合に全てをオンメモリで処理することになってしまうため、1ページ毎の結果をチャネルに流せるようにcollectから呼び出します。
- apiリクエスト
- request paramのstruct定義
- responseデータのstruct定義
    - APIによってはresponseのフィールドが非常に多くBQのカラムに使用しないものが多数あるパターンもあるので、BQに必要なフィールドのみstruct定義する。
    - response jsonのシンプルなunmarshallのみを行います。

# ページネーションとチャネル送信のタイミング

- クローラーには最大3種類(階層)のループがあります。

- 日付ごとのループ
    - キーワードごとにループ
        - ページネーションごとのループ

ページネーションのループ処理1回ごとにデータをチャネルに送信をしてください。

理由としては以下です。

- 大量のページネーションがあったときにデータがメモリに乗り切らず落ちてしまうことを避けるため
- 使用するメモリ量を抑えることでデプロイに必要なリソースサイズを小さく設定することができ、費用削減できるため

cmd, pkg, libの処理をまとめると以下のような構造になっています。

```go
for {　// 日付ごとのループ
	for {　// キーワードごとのループ
		for { // ページネーションごとのループ
			// iteratorでAPIリクエスト
			// チャネルに流す
		}
	}
}
```

# エラーハンドリング

### 基本的な エラー時の対応

- Function 内で 発生したエラーは main に return する
- エラー時はシステムログに出力する。
- 外部のライブラリ含め、 error を return する実装は、省略せず受け取りエラーハンドリングする。
    
    ```bash
    
    # NG
    val1, _ := func()
    
    # OK
    val1, err := func()
    if err != nil {
        // エラー時の処理
    
    }
    ```
    
- 可能な限りエラーを分類して定義する。
    - Config設定のバリデーションエラー、API通信のエラー、その他ミドルウェアのエラーなどは区別して実行ログおよびシステムログに出力する。
    - 「予期せぬエラーが発生しました」は分類できない時のみ出力する。

### API応答

- APIの応答でHTTP Status Code 200, 429以外の場合、エラーと判定する。
    - 実行ログにエラーの原因を出力し、403のようにパラメータの問題の場合で、具体的なエラー項目がわかる場合はそこまで表示するのが望ましい。
- HTTP Status Code 429(Too Many Requests)の場合、
    - retryablehttpのデフォルト処理で1,2,4,8秒待って計4回リトライする。
    - それでも429が続く場合、以下のようにハンドリングする。
        - HeaderにRetry-Afterが入っている場合：その時間だけsleepしてリトライする。
        - HeaderにRetry-Afterが入っていない場合
            1. Retry-Afterに相当する値がAPI独自定義でHeader または bodyに含まれる場合：
                - その値を抽出してsleep timeを算出、Client作成時にOptionでBackoffに渡す関数を指定することでretryablehttpに渡してsleepを実現する。
            2. 何もない場合：
                1. エラーで終了する。
    - 429(Too Many Requests)を発生させにくくするためにAPIコールの間にsleepを入れることも有効。

# 実行ログ(Log thực thi)

Message của 実行ログ(log thực thi) thì được cấu trúc ở 2 cái sau

- 結果 kết quả（phía trên khung）
- 詳細 chi tiết（bên trong khung）
    
    Ex）
  
    ![success](https://github.com/bs-thanhtran/Basic-rule-for-develop-crawler/assets/100660888/76dea5ab-c8b7-43dd-a9a6-0283ffcda458)

    ![fail](https://github.com/bs-thanhtran/Basic-rule-for-develop-crawler/assets/100660888/8403c80b-8628-420f-b68f-f86ede8667b5)
    
- câu văn và hàm trả về crawl.JobError của lỗi sử dụng thông thường thì định nghĩa ở pkg/crawling/error.go 
    - đối với các lỗi của riêng API thì sử dụng NewJobError、định nghĩa bên trong mỗi crawling

### 正常(normal)

- 結果(kết quả)：「ジョブが正常終了しました」
- 詳細(chi tiết)：xuất ra các nội dung sau
    - 開始 start：xuất ra khi khởi động crawler
    - 取得 get：ngày đối tượng get、input parameter、số record get được
        - đơn vị call API chẳng hạn ngày、search keyword（ngoại trừ pagination）
            - những tham số mà cùng setting khi gọi API thì không cần thiết hiển thị ra
        - kiểu mà không có ngày ở tham số của API thì không cần hiển thị ngày
    - 終了 finish：xuất ra khi kết thúc save lên BQ
    
    <aside>
    💡 処理の進捗をユーザーが確認できることで、全体進捗のどこまでが終わっているか、また、完了にどれくらいかかりそうか計画できるようにするために必要な情報です。
    標準的な内容と合わないクローラーでは、この目的を考慮して出力する内容を検討すること。
    
    </aside>
    

### 準正常 (semi-normal)

Trường hợp không phải là lỗi chẳng hạn như bị gián đoạn xử lý 1 thời gian thì xuất nội dung sau ra chi tiết

- アクセス過多 Too many access(429):  “単位時間のアクセス上限を超えました。%d分待ってアクセスします。”

### エラー(error)

- 結果(kết quả)：mô tả ngắn gọn nguyên nhân error
    
    ex）
    
    - 認証に失敗しました   thất bại xác thực
    - ワークフロー設定に問題があります có vấn đề ở setting workflow
    - 一時的なエラーが発生しました  đã phát sinh lỗi tạm thời
- 詳細(chi tiết)：trường hợp kết thúc không bình thường thì xuất lý do đó

<aside>
💡 nếu có vấn đề setting thì xuất ra thông tin để mà tự bản thân user giải quyết
  
Nếu có vấn đề service bên ngoài tạm thời thì hướng dẫn user chờ 1 lát

Bằng cách này thì xuất ra thông tin nhắc user hành động tiếp theo

</aside>

**Error case tiêu biểu**

Bên dưới là các error message chung. Tuy nhiên、đối với các lỗi cần giải thích chi tiết cụ thể hơn thì điều tra riêng biệt rồi setting 

- lỗi API(HTTP Status Code tiêu biểu)
    - 認証エラー lỗi xác thực(401):
        - 結果：認証に失敗しました
        - 詳細：認証に失敗しました。APIキーや認証情報をご確認ください。
    - lỗi parameter(hệ 400):
        - 結果：ワークフロー設定に問題があります
        - 詳細：ワークフロー設定に問題があります。設定をご確認ください。
    - lỗi server kết nối tới(hệ 500):
        - 結果：一時的なエラーが発生しました
        - 詳細：一時的なエラーが発生しました。しばらく経ってから再実行してください。
- lỗi input BigQuery
    - dataset ,table, key đối tượng không tồn tại. Or không có quyền
        - 結果：指定した対象にアクセスできません
        - 詳細：指定した対象が存在しないか、アクセス権がありません。設定をご確認ください。
- Lỗi setting input ở chẳng hạn Google Sheets、Google Drive
    - File ở URL đối tượng không tồn tại、không thể ccess đến file ở URL đối tượng、có sai sót ở chỗ chỉ định range
        - 結果(kết quả)：指定した対象にアクセスできません
        - 詳細(chi tiết)：指定した対象が存在しないか、アクセス権がありません。設定をご確認ください。
    - Format error
        - 結果(kết quả)：指定した対象に問題があります
        - 詳細(chi tiết)：指定した対象（カラム名等）に問題があります。ご確認ください。
- Lỗi validation、lỗi format
    - lỗi format, tham số API không thể combined
        - 結果(kết quả)：ワークフロー設定に問題があります
        - 詳細(chi tiết)：ワークフロー設定に問題があります。設定をご確認ください。
- データ準備前のエラー lỗi trước khi chuẩn bị data
    - ở trường hợp như là giá trị thống kê、chưa tồn tại data sau khi thống kê
        - 結果：ワークフロー設定に問題があります
        - 詳細：指定した日にちのデータが存在しません。設定をご確認ください。
- lỗi abnormal phát sinh mà không thể phân loại vào trong bất cứ loại nào đã có
    - 結果：予期せぬエラーが発生しました
    - 詳細：（không có）

# システムログ system log

<aside>
⚠️ Chú ý tuyệt đối không xuất ra các thông tin cá nhân và thông tin credential

</aside>

- xuất ra ở info level
    - xuất để biết được tham số của API request
    	- tuy nhiên credential chẳng hạn API Key thì không show ra. Xoá khỏi log OR thay thế = chuỗi ký tự đặc biệt như là xxxx 	
    - lọc ra các thông tin hữu ích rồi xuất ra khi phân tích lỗi
- response khi phát sinh error/HTTP status khác 200 thì xuất ra ở error level
    - response API thì cứ để vậy mà xuất ra
- text log sử dụng ở thông thường thì định nghĩa ở pkg/crawling/error.go

# モジュールの配置

構成

```
pkg
├── crawling
│ ├── ads
│ ├── cloudservice
│ ├── config.go
│ ├── crawling.go
│ ├── crawlingtest
│ ├── error.go
│ ├── instagram
│ ├── rakuten
│ ├── storageexport
│ ├── storageservice
│ ├── twitter
│ ├── utility
│ ├── yahoo
│ └── youtube
└── myhttp
│   └── client.go
└── mybigquery
    └── bq.go
```

## 配置ルール

### 共通モジュール

- crawling内のすべてのパッケージで使用されるもの(error.go, config.goなど)はcrawlingパッケージに配置します。
- すべてのパッケージで使用されるわけではないものはpkg配下にディレクトリ(myhttp, mybigquery)を作り別々のパッケージとして配置し、必要に応じてimportします。

### 各クローラーのcollector、libモジュール

- crawling直下のディレクトリはメニュー単位で分離する。
- 「クラウドサービス」のようにメニューの単位が大きい（複数の異なる連携サービスが含まれる）場合、その下にサービス単位でディレクトリを作成、その下にクローラー単位でのディレクトリを作成する。

例1）ヤフーの場合
メニュー単位=サービス単位なのでpkg/crawling/yahoo直下のフォルダをクローラー単位＋lib＋service.goにする。

```
pkg/crawling/yahoo
├── mapmetrics/
├── searchjourney/
├── searchranking/
├── searchtrend/
├── searchvolume/
├── service.go
└── yahoolib/
```

例2）xxの場合

1メニューに複数のサービスが含まれるため、サービスの階層を一つ設ける。

```

```

## メジャーバージョンアップデート時の構成

- メニューを分離するメジャーバージョンアップはR*n(*n=2,3, …)とする。
- サービス単位でのフォルダを新規に作成し、既存クローラーから分離する。

例1）ヤフーの場合
メニュー単位=サービス単位なのでcrawlingの下のyahooをサービスとして扱い、yahoo_r2 を作成する。

```
pkg/crawling
├── yahoo
└── yahoo_r2
     ├── mapmetrics/
     ├── searchjourney/
     ├── searchranking/
     ├── searchtrend/
     ├── searchvolume/
     ├── service.go
     └── yahoolib/
```

- 初版
    
    メニュー直下、lib、serviceでそれぞれ、r2(リリース2)ディレクトリを切って、その中に配置していきます。
    (r1のものはディレクトリを切っていないので、一旦そのまま残します。使われなくなったタイミングでr1ディレクトリを作成して移行するのが良さそうです。)
    
    ```go
    pkg/crawling/yahoo
    ├── mapmetrics
    ├── r2
    │   ├── mapmetrics
    │   ├── searchjourney
    │   ├── searchranking
    │   ├── searchtrend
    │   └── searchvolume
    ├── searchjourney
    ├── searchranking
    ├── searchtrend
    ├── searchvolume
    ├── service
    │   └── r2
    └── yahoolib
        └── r2
    ```
