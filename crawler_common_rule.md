# Crawler common spec

Tập hợp spec chung của mỗi crawler

## ストレージサービス ( Storage service)

### パーティション設定 (Setting partition)

Spec

- 取り込み時間( thời gian import) : _PARTITIONTIME (UTCの擬似列) 。
    - パーティション単位 (đơn vị partition): DAY, MONTH
- 時間単位の列( cột đơn vị thời gian) : xoá toàn bộ data của table tồn tại,rồi import。
    - パーティション単位 (đơn vị partition): DAY, MONTH
    - setting column có kiểu là 1 trong các kiểu DATE, TIMESTAMP, DATETIME
- テーブル Suffix (table Suffix) : 日付でテーブルを分割する（指定したテーブル名に _yyyy-mm-dd がつく）
- なし( không có): import all data mà không có partition。Thay thề toàn bộ data mỗi lần。

- ** Use cases và các setting partition tưởng định**
    1. [通常利用] 定期的にファイルを取り込み、追加する。
        - パーティション設定：「取り込み時間」「テーブル Suffix」
            - 日付入りファイル名 (file_2024-08-20.csv など) の日付を指定して取り込む。
            - ファイルの中身のデータの日付とは関係なく、定期実行時に指定した日付のパーティション / テーブルでファイル内の全データを取り込む。
        - 「時間単位の列」は使用しないこと。（取り込み先のテーブルのデータが削除されるため）
    2. [通常利用] 定期実行で取り込んだファイルのデータに誤りがあったので、修正後のファイルを取り込んで修正する or 過去のデータもファイルから取り込む（バックフィル）。
        - パーティション設定：「取り込み時間」「テーブル Suffix」
            - 同設定で定期実行しているテーブルに対し、指定した日付のパーティションへの洗い替えの取り込みになる。
    3. [特別用途] 「時間単位の列」をパーティションとして指定して、1つのファイルから複数のパーティションに分けて取り込む。
        - 想定パーティション設定：「時間単位の列」
            - 指定したカラムでパーティションを作成して取り込む。
                - 複数の日付のデータが入っている場合はパーティションが分かれる。
            - 既存のテーブルに取り込む場合、データを全削除してから取り込む。
                
                ※ 本仕様の理由
                
                - ファイルの中のデータはCollectro取り込み時はわからないため、指定された「時間単位の列」には複数の日付のデータが入っている可能性がある。
                - 取り込み時にパーティションを指定するには、全データが同じ日付である必要があるが、タイムゾーンを考慮すると指定する日付が同一パーティションにならないケースが発生しうる。
                - 定期実行で指定する相対日付、ファイル名日付、データ内の日付が条件を満たさないと取り込めないため、仕様が複雑になり、ユーザーが適切に使用するのが困難になると判断。（特に、ファイル名日付と異なるデータが修正や遅延などで入ることは容易に起きうるため、実運用上、分離するのは困難だと思われる）
                - 一方で、複数の日付のデータが入ったファイルをその日付でパーティショニングして取り込むことができれば、有用であるため、そのため設定として用意する。（取り込み先のテーブルのデータが削除される前提）
        - 想定パーティション設定：「なし」
            - 既存のテーブルに取り込む場合、データを全削除してから取り込む。
            - 「時間単位の列」のパーティションなしバージョン。

### スキーマ自動設定（カラム名を指定）Setting schema tự động ( chỉ định column name)

Định nghĩa hành vi theo các điều kiện sau

- CSVファイル File CSV
- スキップする行の数（ユースケースに合わせて適切に設定する） Số dòng skip ( setting tương thích kết hợp với use case)
- スキーマ設定：自動検知  Setting schema : detect tự động
    - カラム名を手動で設定: ON   Setting column name : ON

- Crawler đối tượng
    
    
    | 7010005 | Google Drive |
    | --- | --- |
    | 7010006 | Box |
    | 7010007 | Dropbox |
- Use case
    - Use case1: CSV có dòng header、nhưng muốn thay đổi column name và import,do không thể sử dụng column name của BQ
    - Use case2: Muốn setting header và import vào file CSV không có header
    - Use case3: Trường hợp muốn import CSV file mà đã export từ Excel、thì muốn import mà skip 1 số dòng
    - Use case4: Muốn import 1 số dòng bên trong CSV

- Setting ở mỗi user case
    
    Bằng cách import theo các setting dưới đây thì ở tên column đã chỉ định 、có thể import toàn bộ data expected
    
    Sample：[https://drive.google.com/drive/folders/1LXQwrjsdwDgx_qLQMx8OjZoQTmXuDWP-?hl=ja](https://drive.google.com/drive/folders/1LXQwrjsdwDgx_qLQMx8OjZoQTmXuDWP-?hl=ja)
    
    - Use case1: CSV có header
        - スキップする行の数(số dòng skip) : デフォルト Default（1）
        - Sample file ：stock_2024-10-03_standard.csv
    - Use case2: CSV không có dòng header
        - スキップする行の数(số dòng skip): 0
        - Sample file: stock_2024-10-03_no_headers.csv
    - Use case3: CSV có header（1 vài dòng trống）
        - スキップする行の数(số dòng skip): số dòng trống +1
            - e.g.) trường hợp dòng trống: 3 → setting 4
        - Sample file: stock_2024-10-03_with_empty_rows.csv （dòng trống 3, dòng column header 1）
    - Use case4:  CSVファイル（import từ giữa chừng）
        - スキップする行の数(số dòng skip): số cho đến trước dòng data muốn import
            - e.g.) trường hợp muốn import từ dòng thứ 5 → setting 4
        - Sample file: stock_2024-10-03_with_dummys.csv （header column dòng 1, dòng dummy 3）
