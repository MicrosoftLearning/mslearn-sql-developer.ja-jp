---
lab:
  title: ラボ 2 – SQL でプログラミング オブジェクトを実装する
  module: Implement programmability objects with SQL
  description: この演習は、ビュー、ストアド プロシージャ、関数、トリガーなど、主要な SQL Server プログラミング オブジェクトを作成して使用するのに役立ちます。
  level: 300
  duration: 45 minutes
  islab: true
  primarytopics:
    - SQL Server
    - Stored Procedures
    - T-SQL
---

# SQL でプログラミング オブジェクトを実装する

**予測される所要時間**: 45 分

この演習では、主要な SQL Server プログラミング オブジェクトを作成して使用し、ロジックを一カ所にまとめて、いっそう保守しやすくします。

- ビューを作成して複雑なクエリを簡単にする
- ストアド プロシージャを記述してビジネス操作をカプセル化する
- 再利用可能な計算のためのスカラー関数を実装する
- パラメーター化された結果セットのためのインライン テーブル値関数 (`TVF`) を作成する
- トリガーを追加してデータの変更に自動的に応答する

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- SQL Server 2019 以降または Azure SQL Database
- SQL Server Management Studio などのクエリ ツール
- `CREATE` アクセス許可を持つ接続
- [AdventureWorks ライトウェイト サンプル データベース](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure) (SQL Server または Azure SQL)

---

## AdventureWorksLT に接続する

*AdventureWorksLT* サンプル データベースが復元され、SQL インスタンスで使用できることを確認します。 接続といくつかの重要なテーブルを確認します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    -- Verify key tables in AdventureWorksLT
    SELECT TOP (5) CustomerID, FirstName, LastName 
    FROM SalesLT.Customer;
    
    SELECT TOP (5) SalesOrderID, OrderDate, CustomerID 
    FROM SalesLT.SalesOrderHeader;
    
    SELECT TOP (5) ProductID, Name, ListPrice 
    FROM SalesLT.Product;
    ```

    各クエリからは、最大 5 行のサンプル データが返されるはずです。 1 番目の結果セットには顧客名が表示され、2 番目には最近の注文と日付および顧客の参照が表示され、3 番目には製品とその価格の一覧が表示されます。 いずれかのクエリで行が返されないか失敗する場合は、*AdventureWorksLT* データベースが正しく復元されていることと、ご自分に読み取りアクセス権があることを確認してください。

---

## クエリを簡単にするためのビューを作成する

AdventureWorksLT 内の顧客とその注文を結合するビューを作成します (*SalesLT*スキーマ)。 これにより、アプリケーションのコードから `JOIN` の複雑さが隠されます。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選んでビューを作成します。

    ```sql
    CREATE OR ALTER VIEW SalesLT.vCustomerOrders AS
    SELECT 
        c.CustomerID,
        CONCAT(c.FirstName, ' ', c.LastName) AS CustomerName,
        h.SalesOrderID,
        h.OrderDate
    FROM SalesLT.Customer c
    INNER JOIN SalesLT.SalesOrderHeader h ON c.CustomerID = h.CustomerID;
    ```

1. ビューを検証します。 次のクエリを実行します。

    ```sql
    SELECT TOP (5) * 
    FROM SalesLT.vCustomerOrders 
    ORDER BY OrderDate DESC;
    ```

    このクエリからは、最新の注文を示す最大 5 行が返されます。 各行には、`CustomerID`、顧客のフル ネーム、`SalesOrderID`、`OrderDate` が含まれており、ビューを使うと顧客と注文が結合されたデータへのアクセスがどれほど簡単になるかわかります。

---

## 注文を処理するストアド プロシージャを作成する

AdventureWorksLT の既存の注文に注文品目を追加してヘッダーの小計を更新するビジネス操作をカプセル化します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してストアド プロシージャを作成します。

    ```sql
    CREATE OR ALTER PROCEDURE dbo.AddOrderLineItem
        @SalesOrderID INT,
        @ProductID    INT,
        @Quantity     INT
    AS
    BEGIN
        SET NOCOUNT ON;
        BEGIN TRANSACTION;
    
        -- Use Product ListPrice as UnitPrice
        DECLARE @UnitPrice DECIMAL(18,2);
        SELECT @UnitPrice = CAST(ListPrice AS DECIMAL(18,2))
        FROM SalesLT.Product
        WHERE ProductID = @ProductID;
    
        IF @UnitPrice IS NULL
        BEGIN
            ROLLBACK TRANSACTION;
            THROW 50010, 'Invalid ProductID specified.', 1;
        END
    
        -- Ensure SalesOrderID exists
        IF NOT EXISTS (SELECT 1 FROM SalesLT.SalesOrderHeader WHERE SalesOrderID = @SalesOrderID)
        BEGIN
            ROLLBACK TRANSACTION;
            THROW 50011, 'Invalid SalesOrderID specified.', 1;
        END
    
        -- Insert line item (no discount)
        INSERT INTO SalesLT.SalesOrderDetail (SalesOrderID, OrderQty, ProductID, UnitPrice, UnitPriceDiscount)
        VALUES (@SalesOrderID, @Quantity, @ProductID, @UnitPrice, 0);
    
        -- Update header subtotal based on current line totals
        UPDATE h
        SET SubTotal = d.SumLineTotal,
            ModifiedDate = SYSUTCDATETIME()
        FROM SalesLT.SalesOrderHeader h
        INNER JOIN (
            SELECT SalesOrderID, SUM(LineTotal) AS SumLineTotal
            FROM SalesLT.SalesOrderDetail
            WHERE SalesOrderID = @SalesOrderID
            GROUP BY SalesOrderID
        ) d ON d.SalesOrderID = h.SalesOrderID;
    
        COMMIT TRANSACTION;
    END;
    ```

    このストアド プロシージャは、新しい品目のトランザクション挿入を実行します。 最初に、製品と注文が存在することを検証し、どちらかが無効な場合はロールバックしてエラーをスローします。 製品の定価を含む詳細行を挿入した後、すべての品目から注文の小計を再計算して、ヘッダーを更新します。 トランザクション内のすべてをラップすると、操作が確実にアトミックになります。つまり、すべての変更が成功するか、何も適用されないかのどちらかです。

1. 次の T-SQL コードを実行して、ストアド プロシージャをテストします。

    ```sql
    -- Add a line item to an existing order (choose a valid SalesOrderID)
    DECLARE @SalesOrderID INT = (SELECT TOP 1 SalesOrderID 
                                FROM SalesLT.SalesOrderHeader 
                                ORDER BY SalesOrderID DESC);
    EXEC dbo.AddOrderLineItem @SalesOrderID = @SalesOrderID,         
                                @ProductID = 680, 
                                @Quantity = 1; -- adjust ProductID as needed
    
    SELECT TOP (5) * 
    FROM SalesLT.SalesOrderDetail 
    WHERE SalesOrderID = @SalesOrderID 
    ORDER BY SalesOrderDetailID DESC;

    SELECT SalesOrderID, SubTotal, TaxAmt, Freight, TotalDue 
    FROM SalesLT.SalesOrderHeader 
    WHERE SalesOrderID = @SalesOrderID;
    ```

---

## 再利用可能な計算のためのスカラー関数を作成する

AdventureWorksLT の行の合計を使って注文の合計値を返すスカラー関数を作成します。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選んで関数を作成します。

    ```sql
        CREATE OR ALTER FUNCTION dbo.fnOrderTotal (@OrderID INT)
        RETURNS DECIMAL(18,2)
        AS
        BEGIN
            DECLARE @Total DECIMAL(18,2);

            SELECT @Total = SUM(LineTotal)
            FROM SalesLT.SalesOrderDetail
            WHERE SalesOrderID = @OrderID;

            RETURN ISNULL(@Total, 0.00);
        END;
    ```

1. 次の T-SQL コードを実行して、関数を使用します。

    ```sql
    SELECT d.SalesOrderID, dbo.fnOrderTotal(d.SalesOrderID) AS OrderTotal
    FROM SalesLT.SalesOrderDetail d
    GROUP BY d.SalesOrderID
    ORDER BY d.SalesOrderID DESC;
    ```

---

## インライン テーブル値関数 (TVF) を作成する

AdventureWorksLT から特定の顧客の注文を返す `TVF` を作成します。 `TVF` は、`SELECT` と `JOIN` 句で使うと便利です。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選んで TVF を作成します。

    ```sql
    CREATE OR ALTER FUNCTION dbo.GetCustomerOrders (@CustomerID INT)
    RETURNS TABLE
    AS
    RETURN
    (
        SELECT 
            h.SalesOrderID,
            h.OrderDate
        FROM SalesLT.SalesOrderHeader h
        WHERE h.CustomerID = @CustomerID
    );
    ```

1. 次の T-SQL コードを実行して、関数のクエリを実行します。

    ```sql
    SELECT * 
    FROM dbo.GetCustomerOrders(29929)
    ORDER BY OrderDate DESC;
    ```

1. 次の T-SQL コードを実行して、関数を顧客に結合します。

    ```sql
    SELECT CONCAT(c.FirstName, ' ', c.LastName) AS CustomerName, o.SalesOrderID, o.OrderDate
    FROM SalesLT.Customer c
        CROSS APPLY dbo.GetCustomerOrders(c.CustomerID) o
    WHERE c.CustomerID = 29929;
    ```

---

## 変更をログするトリガーを作成する

*SalesLT* の注文の詳細が変化したら注文合計への更新をログするトリガーを追加します。 トリガーは、ルールを適用したり、監査証跡を自動的にキャプチャしたりするのに役立ちます。

1. 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選んで、監査テーブルとトリガーを作成します。

    ```sql
    -- Audit table
    IF OBJECT_ID('dbo.OrderAudit') IS NULL
    BEGIN
        CREATE TABLE dbo.OrderAudit (
            AuditID     INT IDENTITY(1,1) PRIMARY KEY,
            OrderID     INT NOT NULL,
            OldTotal    DECIMAL(18,2) NULL,
            NewTotal    DECIMAL(18,2) NULL,
            ChangedAt   DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
        );
    END
    GO

    -- Trigger on order details updates
    CREATE OR ALTER TRIGGER SalesLT.trg_LogOrderTotalChange
    ON SalesLT.SalesOrderDetail
    AFTER INSERT, UPDATE
    AS
    BEGIN
        SET NOCOUNT ON;

        ;WITH AffectedOrders AS (
            SELECT SalesOrderID FROM inserted
            UNION
            SELECT SalesOrderID FROM deleted
        ),
        -- New totals from the base table (already reflects changes)
        NewTotals AS (
            SELECT d.SalesOrderID, SUM(d.OrderQty * d.UnitPrice) AS Total
            FROM SalesLT.SalesOrderDetail d
            INNER JOIN AffectedOrders a ON d.SalesOrderID = a.SalesOrderID
            GROUP BY d.SalesOrderID
        ),
        -- Contribution of the newly inserted/updated rows
        InsertedTotals AS (
            SELECT SalesOrderID, SUM(OrderQty * UnitPrice) AS Total
            FROM inserted
            GROUP BY SalesOrderID
        ),
        -- Contribution of the previous row versions (empty on INSERT)
        DeletedTotals AS (
            SELECT SalesOrderID, SUM(OrderQty * UnitPrice) AS Total
            FROM deleted
            GROUP BY SalesOrderID
        )
        INSERT INTO dbo.OrderAudit (OrderID, OldTotal, NewTotal)
        SELECT
            n.SalesOrderID,
            n.Total - ISNULL(i.Total, 0) + ISNULL(d.Total, 0) AS OldTotal,
            n.Total AS NewTotal
        FROM NewTotals n
        LEFT JOIN InsertedTotals i ON n.SalesOrderID = i.SalesOrderID
        LEFT JOIN DeletedTotals d ON n.SalesOrderID = d.SalesOrderID;
    END;
    ```

1. 次の T-SQL コードを実行して、トリガーをテストします。

    ```sql
    -- Update an order detail to change the total
    UPDATE d
    SET OrderQty = OrderQty + 1
    FROM SalesLT.SalesOrderDetail d
    WHERE d.SalesOrderID = (SELECT TOP 1 SalesOrderID FROM SalesLT.SalesOrderHeader ORDER BY SalesOrderID DESC);
    
    SELECT TOP (5) * 
    FROM dbo.OrderAudit 
    ORDER BY AuditID DESC;
    ```

    `SELECT` ステートメントは、監査テーブルから最近の行を返します。 各行では、影響を受けた `OrderID`、以前の合計 (`OldTotal`)、数量変更後の新しい合計 (`NewTotal`)、タイムスタンプが示されます。 これにより、トリガーによって変更が自動的にログされたことが確認されます。

---

## クリーンアップ

データベースまたはラボ ファイルを他の目的に使っていない場合、このラボで作成したオブジェクトをクリーンアップしてかまいません。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。
1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、**[サーバー証明書を信頼する]** チェックボックスをオンにする必要があります。
1. **オブジェクト エクスプローラー**で、**[データベース]** フォルダーを展開します。
1. **AdventureWorksLT** データベースを右クリックして、**[削除]** を選びます。
1. **[オブジェクトの削除]** ダイアログで、**[既存の接続を閉じる]** チェックボックスをオンにします。
1. **[OK]** を選択します。

## 次のステップ

1. インデックスを追加して、`JOIN` 操作とフィルター述語のパフォーマンスを向上させることを検討します。
1. 注文ごとに複数の品目を処理するよう、ストアド プロシージャを拡張します。
1. アクセス許可を追加して、エンド ユーザーにビューを安全に公開します。

---

この演習は無事に完了しました。

この演習では、複雑なクエリを簡単にするビュー、トランザクション ビジネス ロジックをカプセル化するストアド プロシージャ、再利用可能な計算のためのスカラー関数、パラメーター化された結果セットのためのインライン テーブル値関数、データが変更されたときに自動的に監査証跡をキャプチャするトリガーなど、SQL Server の主要なプログラミング オブジェクトの実装方法を学びました。
