---
lab:
  title: ラボ 1 – SQL Server でデータベース オブジェクトを設計して実装する
  module: Design and implement database objects with SQL Server
  description: この演習は、テーブル、制約、テンポラル テーブル、JSON 列、インデックスなど、さまざまなデータベース オブジェクトを SQL Server で実装するのに役立ちます。
  duration: 30
  level: 300
  islab: true
  status: released
  targetDate: '2099-01-01'
---

# SQL でデータベース オブジェクトを設計して実装する

**推定時間**:30 分

この演習では、テーブル、制約、テンポラル テーブル、JSON 列、インデックスなど、さまざまなデータベース オブジェクトを SQL Server で実装します。 この演習は、堅牢で効率的なデータベース スキーマの作成方法を理解するのに役立ちます。

あなたは、ある eコマース プラットフォームのデータベース デザイナーです。 適切な制約を持つ標準テーブル、価格変更を追跡するためのテンポラル テーブル、製品メタデータの JSON ストレージ、効率的な履歴データ管理のためのパーティション分割を含むデータベース スキーマを作成する必要があります。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 環境をセットアップします

ラボ仮想マシンが提供され、事前に構成されている場合は、**C:\LabFiles** フォルダーにラボ ファイルが用意されているはずです。 *少し時間をとって確認してください。ファイルが既に存在存在している場合には、このセクションをスキップしてください*。 ただし、独自のマシンを使用している場合、またはラボ ファイルが見つからない場合は、 *GitHub* からそれらを複製して続行する必要があります。

> &#9888; **重要:** この演習では、**SQL Server 2025** 以降が必要です。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、Visual Studio Code のセッションを起動します。

1. コマンド パレット (Ctrl+Shift+P) を開き、「**Git Clone**」と入力します。 **[Git: Clone]** オプションを選択します。

1. **[Repository URL]** フィールドに次の URL を貼り付け、**Enter** キーを押します。

    ```url
    https://github.com/MicrosoftLearning/mslearn-sql-developer.git
    ```

1. リポジトリを、ラボ仮想マシン、または提供されていない場合はローカル コンピューターの **[C:\LabFiles]** フォルダーに保存してください（フォルダーが存在しない場合は作成します）。

---

## 新しいデータベースの作成

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。

1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、**[サーバー証明書を信頼する]** チェックボックスをオンにする必要があります。

    > &#128221; 独自の SQL Server インスタンスを使用している場合は、適切なサーバー インスタンス名と資格情報を使用して接続する必要があることに注意してください。

1. **Databases** フォルダーを選択し、**[新しいクエリ]** を選択します。

1. 次の T-SQL をコピーして、新しいクエリ ウィンドウに貼り付けます。 クエリを実行してデータベースを作成します。

    ```sql
    CREATE DATABASE EcommerceDB;
    GO

    USE EcommerceDB;
    GO
    ```

1. **[メッセージ]** タブに、コマンドが正常に完了したことを示すメッセージが表示されます。

---

## 制約を含むコア テーブルを作成する

eコマース システムの基礎となるテーブルを作成します。

1. **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Create Supplier table
    CREATE TABLE Supplier (
        SupplierID INT PRIMARY KEY IDENTITY(1,1),
        SupplierName NVARCHAR(100) NOT NULL UNIQUE,
        Country NVARCHAR(50) NOT NULL,
        Email NVARCHAR(100),
        Phone NVARCHAR(20),
        CreatedDate DATETIME2 DEFAULT GETUTCDATE()
    );

    -- Create Category table
    CREATE TABLE Category (
        CategoryID INT PRIMARY KEY IDENTITY(1,1),
        CategoryName NVARCHAR(100) NOT NULL UNIQUE,
        Description NVARCHAR(500)
    );

    -- Create Product table with constraints
    CREATE TABLE Product (
        ProductID INT PRIMARY KEY IDENTITY(1,1),
        ProductName NVARCHAR(100) NOT NULL,
        CategoryID INT NOT NULL,
        SupplierID INT NOT NULL,
        BasePrice DECIMAL(10,2) NOT NULL,
        StockQuantity INT NOT NULL DEFAULT 0,
        CreatedDate DATETIME2 DEFAULT GETUTCDATE(),
        CHECK (BasePrice > 0),
        CHECK (StockQuantity >= 0),
        FOREIGN KEY (CategoryID) REFERENCES Category(CategoryID),
        FOREIGN KEY (SupplierID) REFERENCES Supplier(SupplierID),
    );

    -- Create indexes
    CREATE INDEX IX_Category ON Product(CategoryID);
    CREATE INDEX IX_Supplier ON Product(SupplierID);

    GO
    ```

1. テーブルにサンプル データを挿入します。 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Insert sample suppliers
    INSERT INTO Supplier (SupplierName, Country, Email, Phone)
    VALUES 
        ('Contoso Supplies', 'USA', 'contact@contoso.com', '555-0100'),
        ('Fabrikam Inc', 'Canada', 'sales@fabrikam.com', '555-0200');

    -- Insert sample categories
    INSERT INTO Category (CategoryName, Description)
    VALUES 
        ('Electronics', 'Electronic devices and accessories'),
        ('Clothing', 'Apparel and fashion items');

    -- Insert sample products
    INSERT INTO Product (ProductName, CategoryID, SupplierID, BasePrice, StockQuantity)
    VALUES 
        ('Wireless Mouse', 1, 1, 29.99, 100),
        ('Cotton T-Shirt', 2, 2, 19.99, 250);
    GO
    ```

---

## 価格履歴用のテンポラル テーブルを作成する

テンポラル テーブルは、時間経過に伴う変化を自動的に追跡します。 このタスクでは、価格の変更の完全な監査証跡を保持する価格履歴テーブルを作成します。

1. **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Create Price History table with temporal versioning
    CREATE TABLE ProductPrice (
        PriceID INT PRIMARY KEY IDENTITY(1,1),
        ProductID INT NOT NULL,
        CurrentPrice DECIMAL(10,2) NOT NULL,
        EffectiveDate DATE,
        SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN,
        SysEndTime DATETIME2 GENERATED ALWAYS AS ROW END HIDDEN,
        PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime),
        FOREIGN KEY (ProductID) REFERENCES Product(ProductID)
    ) WITH (SYSTEM_VERSIONING = ON);
    GO

    -- Insert initial price data
    INSERT INTO ProductPrice (ProductID, CurrentPrice, EffectiveDate)
    VALUES (1, 99.99, '2025-01-01'), (2, 149.99, '2025-01-01');

    -- Update price (creates history entry)
    UPDATE ProductPrice SET CurrentPrice = 109.99 WHERE ProductID = 1;
    GO
    ```

1. 価格履歴のクエリを実行して、経時的な変化を確認します。 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Query price history
    SELECT ProductID, CurrentPrice, SysStartTime, SysEndTime
    FROM ProductPrice
    FOR SYSTEM_TIME ALL
    WHERE ProductID = 1;
    ```

    > &#128221; テンポラル テーブルでは、現在の価格と以前の価格の両方がそれぞれの時間範囲と共に示されることに注意してください。

---

## メタデータの JSON 列を追加する

JSON 列には、製品の種類によって異なる柔軟な可変データが格納されます。 このタスクでは、メタデータ ストレージを追加し、頻繁にクエリが実行されるプロパティにインデックスを作成します。

1. **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Add metadata column to Product (JSON type requires SQL Server 2025)
    ALTER TABLE Product ADD Metadata JSON;
    GO

    -- Add computed column for indexing
    ALTER TABLE Product ADD MetadataColor AS JSON_VALUE(Metadata, '$.color');
    GO

    -- Create index on the computed column
    CREATE NONCLUSTERED INDEX IX_Product_Metadata_Color
        ON Product (MetadataColor);
    GO

    -- Update products with metadata
    UPDATE Product SET Metadata = N'{"color":"blue","size":"large","material":"cotton"}'
    WHERE ProductID = 1;

    UPDATE Product SET Metadata = N'{"color":"red","size":"small","material":"silk"}'
    WHERE ProductID = 2;
    GO
    ```

1. JSON データのクエリを実行します。 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Query JSON data
    SELECT 
        ProductID,
        ProductName,
        JSON_VALUE(Metadata, '$.color') AS Color,
        JSON_VALUE(Metadata, '$.size') AS Size,
        JSON_VALUE(Metadata, '$.material') AS Material
    FROM Product
    WHERE JSON_VALUE(Metadata, '$.color') = 'blue';
    ```

---

## パーティション分割された注文テーブルを作成する

パーティション分割では、クエリを高速化し、メンテナンスを容易にするために、大きなテーブルをより小さなセグメントに分割します。 このタスクでは、パーティション分割された注文テーブルを作成します。

1. **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Create partition function for order dates
    -- Use RANGE RIGHT for date columns to keep same-day values together
    CREATE PARTITION FUNCTION PF_OrderDate (DATE)
        AS RANGE RIGHT FOR VALUES 
        ('2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01');

    -- Create partition scheme (single filegroup recommended)
    CREATE PARTITION SCHEME PS_OrderDate
        AS PARTITION PF_OrderDate ALL TO ([PRIMARY]);

    -- Create partitioned Order table
    -- Include OrderDate in primary key for clustered index alignment
    CREATE TABLE [Order] (
        OrderID BIGINT IDENTITY(1,1),
        OrderDate DATE NOT NULL,
        CustomerName NVARCHAR(100) NOT NULL,
        TotalAmount DECIMAL(12,2) NOT NULL,
        OrderStatus NVARCHAR(20) DEFAULT 'Pending',
        CONSTRAINT PK_Order PRIMARY KEY (OrderID, OrderDate),
        CHECK (TotalAmount > 0),
        CHECK (OrderStatus IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled'))
    ) ON PS_OrderDate(OrderDate);

    -- Create partitioned index
    CREATE NONCLUSTERED INDEX IX_Order_Customer
        ON [Order](CustomerName)
        ON PS_OrderDate(OrderDate);
    GO

    -- Insert sample orders
    INSERT INTO [Order] (OrderDate, CustomerName, TotalAmount, OrderStatus) VALUES
        ('2025-01-15', 'John Smith', 299.97, 'Delivered'),
        ('2025-02-20', 'Jane Doe', 149.99, 'Shipped'),
        ('2025-06-10', 'Bob Johnson', 449.95, 'Processing');
    GO
    ```

1. パーティションごとにクエリを実行します。 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Query by partition
    SELECT 
        $PARTITION.PF_OrderDate(OrderDate) AS PartitionNumber,
        COUNT(*) AS OrdersInPartition,
        MIN(OrderDate) AS MinDate,
        MAX(OrderDate) AS MaxDate
    FROM [Order]
    GROUP BY $PARTITION.PF_OrderDate(OrderDate);
    ```

---

## SEQUENCE を使用して注文の詳細を作成する

シーケンスでは、どのテーブルとも関係のない一意の番号が生成されます。 このタスクでは、注文品目の識別子にシーケンスを使います。

1. **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Create SEQUENCE for order line items
    CREATE SEQUENCE OrderLineSequence
        START WITH 1
        INCREMENT BY 1;

    -- Create OrderDetail table
    CREATE TABLE OrderDetail (
        OrderLineID INT PRIMARY KEY,
        OrderID BIGINT NOT NULL,
        OrderDate DATE NOT NULL,
        ProductID INT NOT NULL,
        Quantity INT NOT NULL,
        UnitPrice DECIMAL(10,2) NOT NULL,
        LineTotal AS (Quantity * UnitPrice),
        CHECK (Quantity > 0),
        CHECK (UnitPrice > 0),
        FOREIGN KEY (OrderID, OrderDate) REFERENCES [Order](OrderID, OrderDate),
        FOREIGN KEY (ProductID) REFERENCES Product(ProductID)
    );
    GO

    -- Insert order details using SEQUENCE
    INSERT INTO OrderDetail (OrderLineID, OrderID, OrderDate, ProductID, Quantity, UnitPrice)
    VALUES 
        (NEXT VALUE FOR OrderLineSequence, 1, '2025-01-15', 1, 2, 99.99),
        (NEXT VALUE FOR OrderLineSequence, 1, '2025-01-15', 2, 1, 149.99),
        (NEXT VALUE FOR OrderLineSequence, 2, '2025-02-20', 1, 3, 99.99);
    GO
    ```

1. データを検証します。 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    SELECT * FROM OrderDetail;
    ```

---

## データベース オブジェクトを検証する

検証クエリを実行して、すべてのデータベース オブジェクトが正しく作成されたことを確認します。

1. **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Verify constraints work
    -- This should fail: negative price
    INSERT INTO Product (ProductName, CategoryID, SupplierID, BasePrice, StockQuantity)
    VALUES ('Invalid', 1, 1, -50, 10);
    ```

    > &#128221; このクエリは CHECK 制約違反で失敗するはずであり、これは制約が正しく動作していることを示します。

1. JSON とパーティション分割クエリを確認します。 次の T-SQL コードをコピーして新しいクエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE EcommerceDB;
    GO

    -- Verify JSON queries work
    SELECT ProductName, JSON_VALUE(Metadata, '$.color') AS Color
    FROM Product
    WHERE Metadata IS NOT NULL;

    -- Verify partitioning
    SELECT $PARTITION.PF_OrderDate(OrderDate) AS Partition, COUNT(*) AS RecordCount
    FROM [Order]
    GROUP BY $PARTITION.PF_OrderDate(OrderDate);

    -- Verify temporal table
    SELECT ProductID, CurrentPrice, SysStartTime, SysEndTime
    FROM ProductPrice FOR SYSTEM_TIME ALL
    ORDER BY ProductID, SysStartTime;
    ```

---

## クリーンアップ

データベースまたはラボ ファイルを他の目的に使っていない場合、このラボで作成したオブジェクトをクリーンアップしてかまいません。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。
1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、**[サーバー証明書を信頼する]** チェックボックスをオンにする必要があります。
1. **オブジェクト エクスプローラー**で、**[データベース]** フォルダーを展開します。
1. **EcommerceDB** データベースを右クリックして、**[削除]** を選びます。
1. **[オブジェクトの削除]** ダイアログで、**[既存の接続を閉じる]** チェックボックスをオンにします。
1. **[OK]** を選択します。

---

このラボは以上で完了です。

この演習では、制約のあるテーブル、変更を追跡するためのテンポラル テーブル、柔軟なメタデータ格納用の JSON 列、大きなデータ セットのためのパーティション分割テーブル、一意の識別子を生成するためのシーケンスなど、SQL Server でさまざまなデータベース オブジェクトを設計して実装する方法を学びました。
