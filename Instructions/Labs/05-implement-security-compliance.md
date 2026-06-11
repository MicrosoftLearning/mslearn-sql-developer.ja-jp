---
lab:
  title: ラボ 5 – SQL でセキュリティとコンプライアンスを実装する
  module: Implement security and compliance with SQL
  description: この演習は、動的データ マスクや行レベル セキュリティなどのセキュリティ機能を実装して SQL データベース内の機密データを保護するのに役立ちます。
  duration: 30
  level: 300
  islab: true
  status: released
  targetDate: '2099-01-01'
---

# SQL でセキュリティとコンプライアンスを実装する

**推定時間**:30 分

この演習では、セキュリティ機能を実装して SQL データベース内の機密データを保護します。 認可されていなユーザーから機密情報を隠蔽するために動的データ マスクを構成し、ユーザー ID に基づいてデータをフィルター処理するために行レベル セキュリティを実装します。

あなたは、認可されたユーザーが必要な情報に引き続きアクセスできるようにしながら、従業員と顧客のデータを保護する必要があるデータベース開発者です。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- [Azure サブスクリプション](https://azure.microsoft.com/free)
- Visual Studio Code と SQL Server (mssql) 拡張機能、または SQL Server Management Studio
- Azure SQL Database と T-SQL に関する基本的な知識

---

## Azure SQL Database をプロビジョニングする

最初に、セキュリティ演習用の Azure SQL Database を作成します。

1. [Azure portal](https://portal.azure.com) にサインインします。
1. **[Azure SQL]** ページに移動し、リソース メニューで **[Azure SQL Database]** を展開して、**[SQL データベース]** を選択します。
1. **[+ 作成]** を選択してから、**[SQL データベース]** を選択します。
1. **[SQL データベースの作成]** ページで必要な情報を入力します。

    | 設定 | 値 |
    | --- | --- |
    | **サブスクリプション** | Azure サブスクリプションを選択します。 |
    | **リソース グループ** | リソース グループを選択または作成します。 |
    | **データベース名** | *SecurityLabDB* |
    | **[サーバー]** | **[新規作成]** を選択し、**[SQL 認証]** で管理者ログインとパスワードを使い、一意の名前の新しいサーバーを作成します。 |
    | **ワークロード環境** | *開発* |
    | **バックアップ ストレージの冗長性** | *ローカル冗長バックアップ ストレージ* |

1. **[次へ: ネットワーク]** を選択し、次の設定を構成します。

    | 設定 | Value |
    | --- | --- |
    | **接続方法** | *パブリック エンドポイント* |
    | **Azure のサービスとリソースにこのサーバーへのアクセスを許可する** | *はい* |
    | **現在のクライアント IP アドレスを追加する** | *はい* |

1. **[確認および作成]** を選択し、設定を確認してから、**[作成]** を選択します。
1. デプロイが完了するのを待ってから、新しい Azure SQL Database リソースに移動します。

---

## サンプル テーブルを作成する

データベースに接続し、機密データを含むサンプル テーブルを作成します。

1. Visual Studio Code を開き、SQL Server 拡張機能を使って自分の Azure SQL Database に接続します。
1. 新しいクエリ ウィンドウを開き、次のスクリプトを実行してサンプル テーブルを作成します。

    ```sql
    -- Create tables for the exercise
    CREATE TABLE dbo.Employees (
        EmployeeID int PRIMARY KEY IDENTITY(1,1),
        FirstName nvarchar(50) NOT NULL,
        LastName nvarchar(50) NOT NULL,
        Email nvarchar(100) NOT NULL,
        SSN char(11) NOT NULL,
        Salary decimal(18,2) NOT NULL,
        Department nvarchar(50) NOT NULL
    );

    CREATE TABLE dbo.Customers (
        CustomerID int PRIMARY KEY IDENTITY(1,1),
        CompanyName nvarchar(100) NOT NULL,
        ContactName nvarchar(100) NOT NULL,
        Phone nvarchar(20) NOT NULL,
        CreditCardNumber nvarchar(19) NOT NULL,
        SalesRegion nvarchar(20) NOT NULL
    );

    -- Insert sample data
    INSERT INTO dbo.Employees (FirstName, LastName, Email, SSN, Salary, Department)
    VALUES 
        ('Sarah', 'Chen', 'sarah.chen@contoso.com', '123-45-6789', 95000.00, 'Engineering'),
        ('Marcus', 'Johnson', 'marcus.johnson@contoso.com', '234-56-7890', 75000.00, 'Engineering'),
        ('Emily', 'Williams', 'emily.williams@contoso.com', '345-67-8901', 82000.00, 'Sales'),
        ('David', 'Brown', 'david.brown@contoso.com', '456-78-9012', 68000.00, 'Sales'),
        ('Lisa', 'Garcia', 'lisa.garcia@contoso.com', '567-89-0123', 71000.00, 'HR');

    INSERT INTO dbo.Customers (CompanyName, ContactName, Phone, CreditCardNumber, SalesRegion)
    VALUES
        ('Northwind Traders', 'John Smith', '206-555-0100', '4111-1111-1111-1111', 'West'),
        ('Adventure Works', 'Jane Doe', '425-555-0150', '5500-0000-0000-0004', 'East'),
        ('Fabrikam Inc', 'Bob Wilson', '503-555-0175', '3400-0000-0000-009', 'West'),
        ('Contoso Ltd', 'Alice Brown', '360-555-0125', '6011-0000-0000-0004', 'East');
    ```

---

## 動的データ マスクを実装する

動的データ マスクは、機密データをクエリ結果でマスクして、特権を持たないユーザーにそれが表示されないようにします。

1. `Employees` テーブルにマスクを追加して、機密列を保護します。

    ```sql
    -- Mask SSN to show only last 4 digits
    ALTER TABLE dbo.Employees
    ALTER COLUMN SSN ADD MASKED WITH (FUNCTION = 'partial(0, "XXX-XX-", 4)');

    -- Mask Salary with a random value
    ALTER TABLE dbo.Employees
    ALTER COLUMN Salary ADD MASKED WITH (FUNCTION = 'random(50000, 150000)');

    -- Mask Email to show first character and domain
    ALTER TABLE dbo.Employees
    ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
    ```

1. `Customers` テーブルにマスクを追加します。

    ```sql
    -- Mask credit card to show only last 4 digits
    ALTER TABLE dbo.Customers
    ALTER COLUMN CreditCardNumber ADD MASKED WITH (FUNCTION = 'partial(0, "XXXX-XXXX-XXXX-", 4)');

    -- Mask phone number to show only last 4 digits
    ALTER TABLE dbo.Customers
    ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'partial(0, "XXX-XXX-", 4)');
    ```

1. UNMASK アクセス許可を持たないテスト ユーザーを作成し、マスクが機能することを確認します。

    ```sql
    -- Create a user without UNMASK permission
    CREATE USER MaskedViewer WITHOUT LOGIN;
    GRANT SELECT ON dbo.Employees TO MaskedViewer;
    GRANT SELECT ON dbo.Customers TO MaskedViewer;

    -- Query as the masked user (data appears masked)
    EXECUTE AS USER = 'MaskedViewer';
    SELECT FirstName, LastName, Email, SSN, Salary FROM dbo.Employees;
    SELECT CompanyName, ContactName, Phone, CreditCardNumber FROM dbo.Customers;
    REVERT;

    -- Query as admin (data appears unmasked)
    SELECT FirstName, LastName, Email, SSN, Salary FROM dbo.Employees;
    ```

    `MaskedViewer` として実行すると、SSN は `XXX-XX-6789` と表示され、メール アドレスは `sXXX@XXXX.com` と表示され、給与にはランダムな値が示されることに注意してください。 管理者には、実際のデータが表示されます。

---

## 行レベル セキュリティを実装する

行レベル セキュリティ (RLS) を使うと、クエリを実行しているユーザーの特性に基づいて、データベース テーブル内の行へのアクセスを制御できます。

1. さまざまな地域の営業担当者を表すユーザーを作成します。

    ```sql
    -- Create users for different sales regions
    CREATE USER WestSalesRep WITHOUT LOGIN;
    CREATE USER EastSalesRep WITHOUT LOGIN;

    -- Grant SELECT permission on Customers table
    GRANT SELECT ON dbo.Customers TO WestSalesRep;
    GRANT SELECT ON dbo.Customers TO EastSalesRep;
    ```

1. ユーザーの担当地域に基づいて行をフィルター処理するセキュリティ スキーマと述語関数を作成します。

    ```sql
    -- Create a schema for security objects
    CREATE SCHEMA Security;
    GO

    -- Create a function that determines which rows a user can see
    CREATE FUNCTION Security.fn_RegionFilter(@SalesRegion nvarchar(20))
    RETURNS TABLE
    WITH SCHEMABINDING
    AS
    RETURN SELECT 1 AS AccessGranted
        WHERE @SalesRegion = 
            CASE USER_NAME()
                WHEN 'WestSalesRep' THEN 'West'
                WHEN 'EastSalesRep' THEN 'East'
                ELSE @SalesRegion -- Admins see all regions
            END
           OR IS_MEMBER('db_owner') = 1;
    GO
    ```

1. フィルター関数を Customers テーブルに適用するセキュリティ ポリシーを作成します。

    ```sql
    -- Create security policy
    CREATE SECURITY POLICY CustomerRegionPolicy
    ADD FILTER PREDICATE Security.fn_RegionFilter(SalesRegion)
        ON dbo.Customers
    WITH (STATE = ON);
    ```

1. さまざまなユーザーとしてクエリを実行し、行レベル セキュリティをテストします。

    ```sql
    -- Test as WestSalesRep (should see only West region customers)
    EXECUTE AS USER = 'WestSalesRep';
    SELECT * FROM dbo.Customers;
    REVERT;

    -- Test as EastSalesRep (should see only East region customers)
    EXECUTE AS USER = 'EastSalesRep';
    SELECT * FROM dbo.Customers;
    REVERT;

    -- Test as admin (should see all customers)
    SELECT * FROM dbo.Customers;
    ```

    `WestSalesRep` ユーザーには Northwind Traders と Fabrikam Inc (西地域) のみが表示され、`EastSalesRep` には Adventure Works と Contoso Ltd (東地域) が表示されます。 管理者アカウントには、4 つの顧客がすべて表示されます。

---

## クリーンアップ

Azure SQL Database を他の目的に使っていない場合は、作成したリソースをクリーンアップしてかまいません。

1. 次のスクリプトを実行し、セキュリティ オブジェクトとサンプル データを削除します。

    ```sql
    -- Remove security policy and function
    DROP SECURITY POLICY IF EXISTS CustomerRegionPolicy;
    DROP FUNCTION IF EXISTS Security.fn_RegionFilter;
    DROP SCHEMA IF EXISTS Security;

    -- Remove test users
    DROP USER IF EXISTS MaskedViewer;
    DROP USER IF EXISTS WestSalesRep;
    DROP USER IF EXISTS EastSalesRep;

    -- Drop tables
    DROP TABLE IF EXISTS dbo.Customers;
    DROP TABLE IF EXISTS dbo.Employees;
    ```

1. Azure Portalで、リソース グループに移動します。
1. **[リソース グループの削除]** を選択し、リソース グループの名前を入力して削除を確認します。
1. **[削除]** を選択して、このラボで作成されたすべてのリソースを削除します。

---

この演習は無事に完了しました。

この演習では、SQL データベースにセキュリティとコンプライアンスの機能を実装する方法を学びました。 Azure SQL Database のプロビジョニング、機密性の高い列を保護するための動的データ マスクの実装、ユーザーの ID に基づいてデータをフィルター処理するための行レベル セキュリティの構成を練習しました。 これらの機能により、多層防御が提供され、複数のレイヤーでデータが保護されます。
