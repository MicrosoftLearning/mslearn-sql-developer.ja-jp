---
lab:
  title: ラボ 8 – SQL ソリューションと Azure サービスを統合する
  module: Integrate SQL solutions with Azure services
  description: この演習は、製品カタログ データベースのデータ API ビルダー構成を作成し、REST および GraphQL エンドポイントを使用して Azure にデプロイするのに役立ちます。
  duration: 30
  level: 300
  islab: true
  status: released
  targetDate: '2099-01-01'
---

# SQL ソリューションと Azure サービスの統合

**推定時間**:30 分

この演習では、製品カタログ データベースのデータ API ビルダー構成を作成し、Azure にデプロイします。 テーブルとビューのエンティティを構成し、REST と GraphQL エンドポイントを設定し、API が正しく機能することを確認します。

あなたは、最新の API を通じて製品カタログ データを公開する必要があるデータベース開発者です。 チームのフロントエンド開発者は、単純な操作用の REST エンドポイントと、リレーションシップを持つ柔軟なクエリ用の GraphQL の両方を必要としています。

> &#128221; これらの演習では、構成コードをコピーして貼り付けるように求められます。 次の手順に進む前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- [Azure サブスクリプション](https://azure.microsoft.com/free)
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) がインストールされ、サブスクリプションにサインインしている
- [Visual Studio Code](https://code.visualstudio.com/download) などのコード エディターがあり、[SQL Server (mssql) 拡張機能](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql)がインストールされている
- Azure SQL Database または SQL Server インスタンスへのアクセス
- Dotnet SDK 8.0 以降がインストールされている。

---

## サンプル データベースを設定する

データ API ビルダーを構成する前に、テーブルとサンプル データを含むデータベースが必要です。

1. SQL Server Management Studio (SSMS)、または SQL Server (mssql) 拡張機能がある Visual Studio Code を使用して、SQL Server または Azure SQL Database に接続します。 データベースについては、**[master]** を選択します (まだ選択されていない場合)。

1. この演習用の新しいデータベースを作成します。

    ```sql
    CREATE DATABASE ProductCatalog;
    GO
    ```

    SQL Server (Azure SQL Database ではなく) でデータベースを作成した場合は、新しいデータベースに切り替えます。

    ```sql
    USE ProductCatalog;
    GO
    ```

    Azure SQL Database でデータベースを作成した場合は、SSMS または Visual Studio Code のデータベース ドロップダウンから新しいデータベースを選択するか、**[新しいクエリ]** ウィンドウを開き、コンテキストを新しいデータベースに設定します。

1. 製品カタログのテーブルを作成します。

    ```sql
    -- Create Categories table
    CREATE TABLE dbo.Categories (
        CategoryID INT IDENTITY(1,1) PRIMARY KEY,
        CategoryName NVARCHAR(50) NOT NULL,
        Description NVARCHAR(200)
    );
    
    -- Create Products table
    CREATE TABLE dbo.Products (
        ProductID INT IDENTITY(1,1) PRIMARY KEY,
        ProductName NVARCHAR(100) NOT NULL,
        CategoryID INT NOT NULL,
        UnitPrice DECIMAL(10,2) NOT NULL,
        UnitsInStock INT NOT NULL DEFAULT 0,
        Discontinued BIT NOT NULL DEFAULT 0,
        CONSTRAINT FK_Products_Categories 
            FOREIGN KEY (CategoryID) REFERENCES dbo.Categories(CategoryID)
    );
    GO
    
    -- Create a view combining product and category information
    CREATE VIEW dbo.ProductCatalogView AS
    SELECT 
        p.ProductID,
        p.ProductName,
        c.CategoryName,
        p.UnitPrice,
        p.UnitsInStock,
        CASE 
            WHEN p.UnitsInStock = 0 THEN 'Out of Stock'
            WHEN p.UnitsInStock < 10 THEN 'Low Stock'
            ELSE 'Available'
        END AS StockStatus
    FROM dbo.Products p
    INNER JOIN dbo.Categories c ON p.CategoryID = c.CategoryID
    WHERE p.Discontinued = 0;
    GO
    ```

1. サンプル データを挿入します:

    ```sql
    -- Insert categories
    INSERT INTO dbo.Categories (CategoryName, Description) VALUES
    ('Electronics', 'Electronic devices and accessories'),
    ('Clothing', 'Apparel and fashion items'),
    ('Home & Garden', 'Products for home improvement');
    
    -- Insert products
    INSERT INTO dbo.Products (ProductName, CategoryID, UnitPrice, UnitsInStock) VALUES
    ('Wireless Headphones', 1, 79.99, 50),
    ('USB-C Cable', 1, 12.99, 200),
    ('Laptop Stand', 1, 45.00, 5),
    ('Cotton T-Shirt', 2, 24.99, 100),
    ('Running Shoes', 2, 89.99, 0),
    ('Garden Hose', 3, 34.99, 25);
    GO
    ```

    カテゴリ、製品、およびそれらを結合するビューを含むデータベースが作成されました。

---

## データ API ビルダーの CLI をインストールする

DAB CLI は、構成ファイルの作成と管理に役立ちます。

1. ターミナルまたはコマンド プロンプトを開きます。

1. .NET を使用して、データ API ビルダーの CLI をインストールします。

    ```bash
    dotnet tool install --global Microsoft.DataApiBuilder
    ```

1. 次のようにしてインストールを検証します。

    ```bash
    dab --version
    ```

    バージョン番号が表示されるはずです。これにより、CLI がインストールされていることが確認されます。

---

## 初期構成を作成する

DAB CLI を使用して、ベースライン構成ファイルを作成します。

1. プロジェクト用の新規ディレクトリを作成します。

    ```bash
    mkdir product-catalog-api
    cd product-catalog-api
    ```

1. 新しい構成ファイルを初期化します。

    ```bash
    dab init --database-type mssql --connection-string "@env('DATABASE_CONNECTION_STRING')" --host-mode development
    ```

    このコマンドでは、基本設定を使用して `dab-config.json` ファイルを作成します。 `@env()` 構文は、DAB が実行時に環境変数から接続文字列を読み取ることを意味します。

1. エディターで `dab-config.json` を開きます。 次のような構造が表示されるはずです。

    ```json
    {
      "$schema": "https://github.com/Azure/data-api-builder/releases/latest/download/dab.draft.schema.json",
      "data-source": {
        "database-type": "mssql",
        "connection-string": "@env('DATABASE_CONNECTION_STRING')"
      },
      "runtime": {
        "rest": { "enabled": true, "path": "/api" },
        "graphql": { "enabled": true, "path": "/graphql" },
        "host": { "mode": "development" }
      },
      "entities": {}
    }
    ```

    > &#128161; `dab-config.json` ファイルでは、API を介して公開されるデータ ソース、ランタイム設定、エンティティなど、データ API ビルダーの構成を定義します。 現在の設定は、環境や接続しているデータベースによって前の例と異なる場合があります。 設定には、少なくとも @env('DATABASE_CONNECTION_STRING') 変数、`runtime` セクションの /api と /graphql パス、開発モードに設定されたモード、空の `entities` オブジェクトが含まれます。

---

## テーブルのエンティティを追加する

ターミナルまたはコマンド プロンプトで、Categories および Products テーブルを API エンティティとして追加します。

1. CLI を使用して Categories エンティティを追加します。

    ```bash
    dab add Category --source dbo.Categories --permissions "anonymous:read"
    ```

1. 匿名読み取りで Product エンティティを追加します

    ```bash
    dab add Product --source dbo.Products --permissions "anonymous:read"
    ```

1. 認証された完全な CRUD を追加します

    ```bash
    dab update Product --permissions "authenticated:*"
    ```

1. `dab-config.json` を開いて、生成されたエンティティを確認します (環境によっては、ファイルの外観が若干異なる場合があります)。

    ```json
    "entities": {
      "Category": {
        "source": { "object": "dbo.Categories", "type": "table" },
        "permissions": [
          { "role": "anonymous", "actions": ["read"] }
        ]
      },
      "Product": {
        "source": { "object": "dbo.Products", "type": "table" },
        "permissions": [
          { "role": "anonymous", "actions": ["read"] },
          { "role": "authenticated", "actions": ["*"] }
        ]
      }
    }
    ```

---

## フィールド マッピングとリレーションシップを構成する

フィールド マッピングと Category へのリレーションシップを使用して、Product エンティティを強化します。

1. `dab-config.json` を編集して、Product エンティティにフィールド マッピングを追加します。 Product エンティティ セクションを次のように置き換えます。

    ```json
    "Product": {
      "source": { "object": "dbo.Products", "type": "table" },
      "mappings": {
        "ProductID": "id",
        "ProductName": "name",
        "UnitPrice": "price",
        "UnitsInStock": "stockQuantity",
        "Discontinued": "isDiscontinued"
      },
      "relationships": {
        "category": {
          "cardinality": "one",
          "target.entity": "Category",
          "source.fields": ["CategoryID"],
          "target.fields": ["CategoryID"]
        }
      },
      "permissions": [
        { "role": "anonymous", "actions": ["read"] },
        { "role": "authenticated", "actions": ["*"] }
      ]
    }
    ```

    `mappings` セクションでは、データベース列の名前をより API フレンドリな名前に変更します。 `relationships` セクションを使用すると、GraphQL クライアントは製品からそのカテゴリに移動できます。

1. 逆のリレーションシップを Category エンティティに追加し、`dab-config.json` の既存の Category エンティティ セクションを次のように置き換えます。

    ```json
    "Category": {
      "source": { "object": "dbo.Categories", "type": "table" },
      "relationships": {
        "products": {
          "cardinality": "many",
          "target.entity": "Product",
          "source.fields": ["CategoryID"],
          "target.fields": ["CategoryID"]
        }
      },
      "permissions": [
        { "role": "anonymous", "actions": ["read"] }
      ]
    }
    ```

1. `dab-config.json` に対する変更を保存します。

---

## 読み取り専用エンティティとしてビューを追加する

組み合わされた製品とカテゴリの情報が必要なクライアントのために ProductCatalogView を公開します。

1. CLI を使用してビュー エンティティを追加します。

    ```bash
    dab add ProductCatalog --source dbo.ProductCatalogView --source.type view --source.key-fields "ProductID" --permissions "anonymous:read"
    ```

1. エンティティが `dab-config.json` で正しく追加されたことを確認します。

    ```json
    "ProductCatalog": {
      "source": {
        "object": "dbo.ProductCatalogView",
        "type": "view",
        "key-fields": ["ProductID"]
      },
      "graphql": { "operation": "query" },
      "permissions": [
        { "role": "anonymous", "actions": ["read"] }
      ]
    }
    ```

    DAB は主キーを自動的に検出できないため、ビューには `key-fields` プロパティが必要です。

---

## ローカルでテストする

データ API ビルダーをローカルで実行して、構成を確認します。 `dab start` コマンドでは、.NET ランタイムを使用してローカル Web サーバーを起動し、指定した接続文字列を使ってデータベースに接続します。

1. **PowerShell** または **Bash ** ターミナルを開きます。 このセクションの残りのコマンドは、PowerShell または Bash (CMD ではなく) で実行する必要があります。 現在、CMD プロンプトが表示されている場合は、「`pwsh`」と入力し、Enter キーを押して PowerShell に切り替えます (プロンプトが `PS` に変わることに注目してください)。

1. データベース接続文字列を環境変数として設定します。 SQL Server が認証用に構成された方法に応じて、次の 2 つのオプションのうちの **1 つ**を使用します。

    > &#128221; あなた (または管理者) は Azure SQL サーバーを作成したときに、組織のセキュリティ ポリシーに基づいて認証方法を選択しました。 同じ選択によって、ここで使用する接続文字列が決まります。 どの方法が構成されたかわからない場合は、Azure portal を確認してください。SQL Server > **[設定]** > **[Microsoft Entra ID]** に移動します。 Entra 管理者が設定されていて、**[このサーバーの Microsoft Entra 専用認証をサポートする]** チェックボックスがオンになっている場合は、オプション A を使用します。

    **オプション A: Microsoft Entra 認証** (ユーザー名/パスワードは不要。サインインした Azure ID を使用)

    | Shell | コマンド |
    | --- | --- |
    | **PowerShell** | `$env:DATABASE_CONNECTION_STRING="Server=your-server.database.windows.net;Database=ProductCatalog;Authentication=Active Directory Default;TrustServerCertificate=true"` |
    | **Bash** | `export DATABASE_CONNECTION_STRING="Server=your-server.database.windows.net;Database=ProductCatalog;Authentication=Active Directory Default;TrustServerCertificate=true"` |

    **オプション B: SQL 認証** (サーバー作成時の管理者ログインとパスワードを使用)

    | Shell | コマンド |
    | --- | --- |
    | **PowerShell** | `$env:DATABASE_CONNECTION_STRING="Server=your-server.database.windows.net;Database=ProductCatalog;User Id=your-user;Password=your-password;TrustServerCertificate=true"` |
    | **Bash** | `export DATABASE_CONNECTION_STRING="Server=your-server.database.windows.net;Database=ProductCatalog;User Id=your-user;Password=your-password;TrustServerCertificate=true"` |

    > &#128161; `your-server` は実際のサーバー名に置き換えてください。 オプション B の場合は、`your-user` と `your-password` も、サーバーの作成時に設定した管理者資格情報に置き換えます。

1. データ API ビルダーを起動します。

    ```bash
    dab start
    ```

1. ブラウザーを開き、`http://localhost:5000/api/Product` に移動して REST エンドポイントをテストします。

    マップされたフィールド名 (id、name、price、stockQuantity) を使用して、製品データを含む JSON 応答が表示されるはずです。

    > &#128221; **オプション A (Microsoft Entra) を使用し**、`DefaultAzureCredential failed to retrieve a token` に関するエラーが発生した場合は、Azure CLI セッションの有効期限が切れている可能性があります。 ターミナルで `az login --scope https://database.windows.net//.default` を実行して再認証してから、`dab start` をもう一度実行します。

1. `http://localhost:5000/graphql` に移動して、GraphQL プレイグラウンドにアクセスします。

1. **[ドキュメントの作成]** を選択して、新しいクエリ タブを開きます。次の GraphQL クエリを貼り付け、**Ctrl + Alt + Enter** キーを押して (または **[実行]** ボタンを選択して) 実行します。

    ```graphql
    query {
      products {
        items {
          id
          name
          price
          category {
            CategoryName
          }
        }
      }
    }
    ```

    応答には、構成されたリレーションシップを示す、関連するカテゴリ名を含む製品情報が含まれています。

1. ターミナルに戻り、`Ctrl+C` キーを押してデータ API ビルダーを停止します。

---

## クリーンアップ

他の目的でデータベースを使用していない場合は、作成したリソースをクリーンアップできます。

1. 次のスクリプトを実行して、データベース オブジェクトを削除します。

    ```sql
    USE master;
    GO
    DROP DATABASE IF EXISTS ProductCatalog;
    GO
    ```

1. プロジェクト ディレクトリを削除します。

    ```bash
    cd ..
    rm -rf product-catalog-api
    ```

---

この演習は正常に完了しました。

この演習では、データ API ビルダー構成を最初から作成する方法を学びました。 テーブルとビューのエンティティの作成、フィールド マッピングとリレーションシップの構成、およびローカルでの API のテストを実践しました。 これらのスキルを使用すると、最新の REST と GraphQL API を通じて SQL データベースを迅速に公開できます。
