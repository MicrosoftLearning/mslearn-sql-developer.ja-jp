---
lab:
  title: ラボ 3 – 高度な T-SQL クエリを記述する
  module: Write advanced T-SQL queries
  description: この演習は、JSON 関数、CTE、ウィンドウ関数を使って AdventureWorksLT データベースからデータを構築してクエリを実行する練習をするのに役立ちます。
  level: 300
  duration: 30 minutes
  islab: true
  primarytopics:
    - T-SQL
    - JSON
    - SQL Server
---

# 高度な T-SQL クエリを記述する

**推定時間**:30 分

この演習では、JSON 関数を使って AdventureWorksLT データベースから JSON データを構築してクエリを実行する練習をします。 また、JSON の出力を CTE やウィンドウ関数と組み合わせて、実用的なレポートを作成します。

あなたは、eコマース企業のデータベース開発者です。 マーケティング チームは Web カタログの製品データを JSON 形式で必要としており、あなたはカテゴリ内の製品のランクを付けたレポートを作成する必要があります。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- SQL Server 2022 以降または Azure SQL Database
- SQL Server Management Studio などのクエリ ツール
- 読み取りアクセス許可を持つ接続
- [AdventureWorks ライトウェイト サンプル データベース](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure) (SQL Server または Azure SQL)

---

## AdventureWorksLT に接続する

*AdventureWorksLT* サンプル データベースが復元され、SQL インスタンスで使用できることを確認します。 接続を検証する:

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    -- Verify key tables in AdventureWorksLT
    SELECT TOP (5) ProductID, Name, ListPrice 
    FROM SalesLT.Product;
    
    SELECT TOP (5) ProductCategoryID, Name 
    FROM SalesLT.ProductCategory;
    ```

    各クエリからは、最大 5 行のサンプル データが返されるはずです。 いずれかのクエリで行が返されないか失敗する場合は、*AdventureWorksLT* データベースが正しく復元されていることと、ご自分に読み取りアクセス権があることを確認してください。

---

## 製品データから JSON 出力を作成する

マーケティング チームは、Web カタログの製品情報を JSON 形式で必要としています。 最初に、Product テーブルから簡単な JSON オブジェクトを作成します。

### 製品ごとに JSON オブジェクトを作成する

`FOR JSON PATH` を使って、製品の行を JSON に変換します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    SELECT 
        ProductID,
        Name,
        Color,
        ListPrice
    FROM SalesLT.Product
    WHERE Color IS NOT NULL
    ORDER BY ListPrice DESC
    FOR JSON PATH;
    ```

    このクエリは、色の値を含む製品を選択し、結果を JSON 配列として書式設定します。 各行は、列名と一致するプロパティを持つ JSON オブジェクトになります。 `FOR JSON PATH` 句は、変換を自動的に処理します。

### 製品カテゴリで入れ子になった JSON を作成する

入れ子になったオブジェクトとしてカテゴリ情報を追加します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    SELECT 
        p.ProductID,
        p.Name AS ProductName,
        p.ListPrice,
        JSON_OBJECT(
            'CategoryID': pc.ProductCategoryID,
            'CategoryName': pc.Name
        ) AS Category
    FROM SalesLT.Product AS p
    INNER JOIN SalesLT.ProductCategory AS pc
        ON p.ProductCategoryID = pc.ProductCategoryID
    ORDER BY p.ListPrice DESC
    FOR JSON PATH;
    ```

    このクエリは、`JSON_OBJECT` を使って入れ子構造を作成します。 Category プロパティには、CategoryID と CategoryName を含む独自の JSON オブジェクトが含まれます。 この方法では、関連するデータがグループ化されて出力で維持されます。

---

## JSON と CTE およびウィンドウ関数を組み合わせる

次に、各カテゴリ内で価格によって製品のランクを付け、結果を JSON として出力する、もう少し便利なレポートを作成します。

### ウィンドウ関数のランク付けを含む CTE を作成する

最初に、CTE と `ROW_NUMBER()` を使ってクエリのロジックを作成します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    WITH RankedProducts AS (
        SELECT 
            p.ProductID,
            p.Name AS ProductName,
            pc.Name AS CategoryName,
            p.ListPrice,
            ROW_NUMBER() OVER (
                PARTITION BY pc.ProductCategoryID 
                ORDER BY p.ListPrice DESC
            ) AS PriceRank
        FROM SalesLT.Product AS p
        INNER JOIN SalesLT.ProductCategory AS pc
            ON p.ProductCategoryID = pc.ProductCategoryID
        WHERE p.ListPrice > 0
    )
    SELECT 
        ProductID,
        ProductName,
        CategoryName,
        ListPrice,
        PriceRank
    FROM RankedProducts
    WHERE PriceRank <= 3
    ORDER BY CategoryName, PriceRank;
    ```

    この CTE は、カテゴリ内の製品ごとに価格ランクを計算します。 `PARTITION BY` 句はカテゴリごとに番号付けを開始し直し、`ORDER BY ListPrice DESC` は最も高価な製品にランク 1 を割り当てます。 外側のクエリでは、カテゴリごとに上位 3 つの製品のみを表示するようにフィルター処理が行われます。

### ランク付けされた製品を JSON として出力する

`FOR JSON PATH` を追加して、結果を API 用に書式設定します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    WITH RankedProducts AS (
        SELECT 
            p.ProductID,
            p.Name AS ProductName,
            pc.Name AS CategoryName,
            p.ListPrice,
            ROW_NUMBER() OVER (
                PARTITION BY pc.ProductCategoryID 
                ORDER BY p.ListPrice DESC
            ) AS PriceRank
        FROM SalesLT.Product AS p
        INNER JOIN SalesLT.ProductCategory AS pc
            ON p.ProductCategoryID = pc.ProductCategoryID
        WHERE p.ListPrice > 0
    )
    SELECT 
        ProductID,
        ProductName,
        CategoryName,
        ListPrice,
        PriceRank
    FROM RankedProducts
    WHERE PriceRank <= 3
    ORDER BY CategoryName, PriceRank
    FOR JSON PATH, ROOT('TopProducts');
    ```

    `ROOT('TopProducts')` を追加すると、JSON 配列全体が名前付きプロパティを持つオブジェクト内にラップされます。 これにより、ルート要素が必要なアプリケーションで出力を操作しやすくなります。

---

## OPENJSON を使用して JSON データを解析する

次に、`OPENJSON` を使って JSON データを行に読み戻す練習をします。

### JSON 配列を行として解析する

製品の更新を JSON として受け取ったとします。 それをテーブルに変換するには `OPENJSON` を使います。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    DECLARE @ProductUpdates NVARCHAR(MAX) = N'[
        {"ProductID": 680, "NewPrice": 1250.00},
        {"ProductID": 706, "NewPrice": 1450.00},
        {"ProductID": 707, "NewPrice": 38.99}
    ]';

    SELECT 
        ProductID,
        NewPrice
    FROM OPENJSON(@ProductUpdates)
    WITH (
        ProductID INT '$.ProductID',
        NewPrice DECIMAL(10,2) '$.NewPrice'
    );
    ```

    `WITH` 句は、出力のスキーマを定義します。 各 JSON プロパティは、指定されたデータ型を持つ列にマップされます。 `$.PropertyName` 構文は、列ごとに読み取る JSON パスを SQL Server に指示します。

### 解析された JSON を既存のデータと結合する

JSON データと Product テーブルを組み合わせて、現在の価格と新しい価格を確認します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    DECLARE @ProductUpdates NVARCHAR(MAX) = N'[
        {"ProductID": 680, "NewPrice": 1250.00},
        {"ProductID": 706, "NewPrice": 1450.00},
        {"ProductID": 707, "NewPrice": 38.99}
    ]';

    SELECT 
        p.ProductID,
        p.Name,
        p.ListPrice AS CurrentPrice,
        updates.NewPrice,
        updates.NewPrice - p.ListPrice AS PriceDifference
    FROM SalesLT.Product AS p
    INNER JOIN OPENJSON(@ProductUpdates)
    WITH (
        ProductID INT '$.ProductID',
        NewPrice DECIMAL(10,2) '$.NewPrice'
    ) AS updates
        ON p.ProductID = updates.ProductID;
    ```

    このクエリは、解析された JSON を Product テーブルと直接結合します。 `WITH` 句を含む `OPENJSON` 関数はテーブルのように機能するため、他のデータ ソースと同様に結合できます。 結果では、各製品の現在の価格と提案された新しい価格が、並べて表示されます。

---

## クリーンアップ

データベースまたはラボ ファイルを他の目的に使っていない場合、このラボで作成したオブジェクトをクリーンアップしてかまいません。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。
1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、**[サーバー証明書を信頼する]** チェックボックスをオンにする必要があります。
1. **オブジェクト エクスプローラー**で、**[データベース]** フォルダーを展開します。
1. **AdventureWorksLT** データベースを右クリックして、**[削除]** を選びます。
1. **[オブジェクトの削除]** ダイアログで、**[既存の接続を閉じる]** チェックボックスをオンにします。
1. **[OK]** を選択します。

---

この演習は無事に完了しました。

この演習では、JSON 関数とウィンドウ関数を使って高度な T-SQL クエリを記述する方法を学びました。 `FOR JSON PATH` を使ってクエリの結果から JSON 出力を生成し、`JSON_OBJECT` で入れ子になった JSON 構造を作成し、JSON 出力と CTE およびウィンドウ関数を組み合わせ、`OPENJSON` とスキーマを使って JSON 配列を行に解析する方法を練習しました。
