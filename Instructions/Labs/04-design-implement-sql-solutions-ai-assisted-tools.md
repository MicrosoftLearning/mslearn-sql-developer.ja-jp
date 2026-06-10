---
lab:
  title: ラボ 4 – AI 支援ツールを使用して SQL ソリューションを実装する
  module: Implement SQL solutions by using AI-assisted tools
  description: この演習は、GitHub Copilot のような AI 支援開発ツールを使用する一貫した標準での SQL ソリューションの設計と実装に役立ちます。
  level: 300
  duration: 45 minutes
  islab: true
  primarytopics:
    - Azure SQL Database
    - GitHub Copilot
    - AI-Assisted Development
---

# AI 支援ツールを使用して SQL ソリューションを実装する

**予測される所要時間**: 45 分

この演習では、AI 支援開発ツールを使用した SQL ソリューションの設計と実装を実践します。 Visual Studio Code で GitHub Copilot を設定し、一貫した T-SQL コード生成のためのカスタム指示ファイルを作成し、Copilot でデータベース オブジェクトを生成します。

あなたは AI 支援ツールを使って開発ワークフローを迅速化したいデータベース開発者です。 あなたのチームは、一貫した標準とベスト プラクティスに沿った T-SQL コードの記述に活用するため、GitHub Copilot を採用しました。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- [Azure サブスクリプション](https://azure.microsoft.com/free)
- [Copilot のアクセスがある GitHub アカウント](https://github.com/features/copilot)
- ご利用のコンピューターに Visual Studio Code がインストールされていること
- Azure SQL Database と T-SQL に関する基本的な知識

---

## Azure SQL Database をプロビジョニングする

まず、GitHub Copilot で使用する Azure SQL Database を作成する必要があります。

1. [Azure portal](https://portal.azure.com) にサインインします。
1. **[Azure SQL]** ページに移動し、リソースメニューで **[Azure SQL Database]** を展開し、**[SQL データベース]** を選択します。
1. **[+ 作成]** を選択してから、**[SQL データベース]** を選択します。
1. **[SQL データベースの作成]** ページで必要な情報を入力します。

    | 設定 | 値 |
    | --- | --- |
    | **サブスクリプション** | Azure サブスクリプションを選択します。 |
    | **リソース グループ** | リソース グループを選択または作成します。 |
    | **データベース名** | *AdventureWorksLT* |
    | **[サーバー]** | **[新規作成]** を選択し、**[SQL 認証]** で管理者ログインとパスワードを使い、一意の名前の新しいサーバーを作成します。 |
    | **ワークロード環境** | *開発* |
    | **バックアップ ストレージの冗長性** | *ローカル冗長バックアップ ストレージ* |

1. **[次へ: ネットワーク]** を選択し、次の設定を構成します。

    | 設定 | Value |
    | --- | --- |
    | **接続方法** | *パブリック エンドポイント* |
    | **Azure のサービスとリソースにこのサーバーへのアクセスを許可する** | *はい* |
    | **現在のクライアント IP アドレスを追加する** | *はい* |

1. **[次へ: セキュリティ]**、**[次へ: 追加設定]** の順に選択します。
1. **[追加設定]** ページで **[既存データを使用]** を [サンプル] に設定して AdventureWorksLT サンプル データベースを作成します。**
1. **[確認および作成]** を選択し、設定を確認してから、**[作成]** を選択します。
1. デプロイが完了するのを待ってから、新しい Azure SQL Database リソースに移動します。

---

## Visual Studio Code と GitHub Copilot を設定する

次に、Visual Studio Code に AI 支援による SQL 開発に必要な拡張機能を構成します。

1. コンピューターで **Visual Studio Code** を開きます。
1. アクティビティバーの **[拡張機能]** アイコンを選択します (または **Ctrl + Shift + X** キーを押します)。
1. 次の拡張機能を検索してインストールします:
    - **GitHub Copilot Chat** (GitHub)
    - **SQL Server (mssql)** (Microsoft)
1. インストール後、アクティビティ バーで **[アカウント]** アイコンを選択します。
1. **[サインインして GitHub Copilot を使用する]** を選択して、Copilot へのアクセスがある GitHub アカウントでサインインします。
1. ステータス バーに、Copilot がアクティブであることを示す Copilot のアイコンが表示されていることを確認します。

---

## Azure SQL Database に接続する

次に、Visual Studio Code を Azure SQL Database に接続します。

1. Visual Studio Code のアクティビティ バーで **[SQL Server]** アイコンを選択します。
1. **[接続を追加]** を選択し、次の接続の詳細を入力します:

    | 設定 | 値 |
    | --- | --- |
    | **サーバー名** | お使いの Azure SQL Server 名 (例:yourserver.database.windows.net)** |
    | **データベース名** | *AdventureWorksLT* |
    | **認証の種類** | *SQL Login* |
    | **ユーザー名** | SQL 管理者ユーザー名** |
    | **パスワード** | SQL 管理者パスワード** |
    | **サーバー証明書を信頼する** | *True* |

1. **[接続する]** を選択し、**[接続]** ペインに接続が表示されていることを確認します。
1. 接続を展開してデータベース オブジェクト (テーブル、ビュー、ストアド プロシージャ) を表示します。

---

## Copilot のカスタム指示ファイルを作成する

カスタム指示ファイルは、Copilot にチームの標準に沿ったコードを生成するように指示します。 T-SQL 開発用の指示ファイルを作成します。

1. Visual Studio Code で、データベース プロジェクトを格納するフォルダーを開きます (または新しいフォルダーを作成します)。
1. プロジェクトのルートに `.github` という名前の新しいフォルダーを作成します。
1. `.github` フォルダーに、`copilot-instructions.md` という名前の新しいファイルを作成します。
1. 次の内容を指示ファイルに追加します。

    ```markdown
    # T-SQL Development Guidelines for Copilot

    ## Naming Conventions
    - Tables: PascalCase, singular form (Customer, Product, SalesOrder)
    - Columns: PascalCase (FirstName, OrderDate, UnitPrice)
    - Stored procedures: usp_ActionEntity (usp_GetCustomerOrders, usp_InsertProduct)
    - Views: vw_EntityDescription (vw_ActiveCustomers, vw_ProductInventory)
    - Indexes: IX_TableName_ColumnName

    ## T-SQL Style Guidelines
    - Always use explicit column lists in SELECT statements (avoid SELECT *)
    - Include schema prefix for all objects (SalesLT.Product, SalesLT.Customer)
    - Use ANSI JOIN syntax (INNER JOIN, LEFT JOIN) instead of comma-separated tables
    - Include SET NOCOUNT ON at the beginning of stored procedures
    - Use TRY...CATCH blocks for error handling in stored procedures

    ## Security Requirements
    - Use parameterized queries, never concatenate user input
    - Never include actual credentials or connection strings in code
    - Use least-privilege principles for GRANT statements

    ## Comments
    - Include a header comment with procedure name, purpose, and author
    - Add inline comments for complex logic
    ```

1. ファイルを保存します。 Copilot は、今後 T-SQL コードを生成する際にこれらのガイドラインを考慮します。

---

## Copilot を使用してストアド プロシージャを生成する

次に GitHub Copilot を使って、カスタム ガイドラインに従ってストアド プロシージャを生成します。

1. Visual Studio Code で、`usp_GetCustomerOrderSummary.sql` という名前の新しいファイルを作成します。
1. **Ctrl + Alt + I** キーを押して (または Copilot Chat アイコンを選択して) **Copilot Chat** パネルを開きます。
1. Copilot Chat パネルの左下にある **[モード]** が **[Ask]** に設定されていることを確認します。
1. 次のプロンプトをチャットに入力します。

    ```text
    Create a stored procedure named usp_GetCustomerOrderSummary that retrieves customer order information from the AdventureWorksLT database. The procedure should:
    - Accept a @CustomerID parameter (optional, if NULL return all customers)
    - Return customer name, total number of orders, total order amount, and last order date
    - Join SalesLT.Customer, SalesLT.SalesOrderHeader, and SalesLT.SalesOrderDetail tables
    - Include error handling with TRY...CATCH
    - Follow the T-SQL guidelines in the instruction file
    ```

1. 生成されたコードを確認します。 Copilot がご自分の命名規則やスタイル ガイドラインに従っていることに注目します。
1. 必要に応じて、Copilot にコードの修正を依頼します。

    ```text
    Add a comment header and ensure the procedure uses SET NOCOUNT ON
    ```

1. 最終的なストアド プロシージャ コードを SQL ファイルにコピーします。

---

## Copilot を使ってビューを生成する

Copilot を使って、アプリケーションのニーズをサポートするビューを作成します。

1. `vw_ProductSalesAnalysis.sql` という名前で新しいファイルを作成します。
1. [Copilot Chat] パネルに次のプロンプトを入力します。

    ```text
    Create a view named vw_ProductSalesAnalysis that shows:
    - Product name and category
    - Total quantity sold
    - Total revenue
    - Average sale price
    - Number of orders containing this product
    
    Use the SalesLT schema tables and follow the T-SQL guidelines.
    ```

1. 生成されたコードを確認し、命名規則やスタイル ガイドラインに従っていることを確認します。
1. コードを SQL ファイルにコピーします。

---

## Copilot を使用して既存のコードを説明する

Copilot は既存のデータベース コードの理解にも役立ちます。

1. **[SQL Server]** 接続ペインで、データベースの下の **[ビュー]** を展開します。
1. `SalesLT.vGetAllCategories` を右クリックして **Script as Create** を選択します。
1. エディターでビューのコード全体を選択します。
1. Copilot Chat を開き、次のように入力します:

    ```text
    Explain what this view does and how the recursive CTE works
    ```

1. 説明を確認します。 Copilot でコードが分析され、その機能について明確な説明が提供されます。

---

## クエリ最適化の提案に Copilot を使用する

Copilot にクエリの最適化を依頼します。

1. `query_optimization.sql` という名前で新しいファイルを作成します。
1. 次のように入力するかこれを貼り付けます。

    ```sql
    SELECT *
    FROM SalesLT.SalesOrderHeader h, SalesLT.SalesOrderDetail d, SalesLT.Product p
    WHERE h.SalesOrderID = d.SalesOrderID
    AND d.ProductID = p.ProductID
    AND h.OrderDate > '2008-01-01'
    ```

1. クエリを選択して Copilot Chat を開きます。
1. 次のプロンプトを入力します。

    ```text
    Review this query and suggest optimizations following best practices. Explain each improvement.
    ```

1. Copilot の提案を確認します。次の内容が含まれているはずです。
    - 明示的な ANSI JOIN 構文に変換する
    - SELECT * を特定の列名に置き換える
    - スキーマ プレフィックスを追加する
    - インデックスを追加する可能性がある

---

## クリーンアップ

Azure SQL Database またはラボ ファイルを他の目的で使用していない場合は、この演習で作成したリソースをクリーンアップできます。

1. Azure Portalで、リソース グループに移動します。
1. **[リソース グループの削除]** を選択し、リソース グループの名前を入力して削除を確認します。
1. **[削除]** を選択して、このラボで作成されたすべてのリソースを削除します。
1. Visual Studio Code で、必要に応じて **[アカウント]** アイコンを選択し、**[サインアウト]** を選択すると GitHub Copilot からサインアウトできます。

---

この演習は正常に完了しました。

この演習では、AI 支援ツールを使用して SQL ソリューションを設計および実装する方法を学習しました。 Azure SQL Database のプロビジョニング、Visual Studio Code での GitHub Copilot の構成、コード生成をガイドするカスタム指示ファイルの作成、Copilot を使用したストアド プロシージャとビューの生成、既存のデータベース コードの説明、クエリの最適化の提案の取得を実践しました。
