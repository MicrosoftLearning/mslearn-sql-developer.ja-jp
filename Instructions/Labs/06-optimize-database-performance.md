---
lab:
  title: 演習 6 - クエリのパフォーマンスを最適化する
  module: Optimize database performance
  description: この演習は、実行プラン、動的管理ビュー (DMV)、クエリ ストアを使用して、Azure SQL Database の低速クエリを調査するのに役立ちます。
  level: 300
  duration: 45 minutes
  islab: true
  primarytopics:
    - Azure SQL Database
    - Query Performance
    - Query Store
---

# クエリ パフォーマンスの最適化

**予測される所要時間**: 45 分

この演習では、実行プラン、動的管理ビュー (DMV)、クエリ ストアを使用して、Azure SQL Database の低速クエリを調査します。 現実的なワークロードを構築し、実行プランを通じて不足しているインデックスを特定します。 その後、ストアド プロシージャと偏ったデータを使用して、パラメーター スニッフィング回帰をシミュレートします。 SSMS の [最もリソース消費量の多いクエリ] ビューを使用して回帰を検出し、より適切なプランを強制します。 また、クエリ ストアのヒントを適用し、2 つのライター間のブロック シナリオを診断します。

あなたは、Adventure Works のデータベース管理者です。 最近のデプロイ後、カスタマーサービス チームから、注文検索が以前よりも遅くなったという報告がありました。 実行プランを使用して根本原因を見つけ、クエリ ストアを使用して回帰を確認して修正し、DMV を使用して倉庫チームから報告された、障害となっている問題を調査します。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- [Azure サブスクリプション](https://azure.microsoft.com/free)
- クエリ ストア GUI セクションの SQL Server Management Studio (SSMS)
- Azure SQL Database と T-SQL に関する基本的な知識

---

## Azure SQL Database をプロビジョニングする

まず、サンプル データを含む Azure SQL Database を作成します。

> &#128221; AdventureWorksLT Azure SQL Database が既にプロビジョニングされている場合は、このセクションをスキップしてください。

1. [Azure SQL ハブ](https://aka.ms/azuresqlhub)に移動し、メッセージが表示されたら Azure アカウントでサインインします。 **[Azure SQL Database]** ペインで、**[オプションの表示]**、**[SQL データベースの作成]** の順に選択します。

    > &#128161; このページに**無料オファー**のバナーが表示される場合は、それを適用して Azure SQL Database を無料で使用できます。 この[無料オファー](https://learn.microsoft.com/azure/azure-sql/database/free-offer)では、1 か月あたり 100,000 vCore 秒のサーバーレス コンピューティングと 32GB のストレージが提供されます。 無料オファーを適用する場合は、ステップ 3 から 6 までをスキップしてください。

1. **[SQL データベースの作成]** ページの **[基本]** タブに、必要な情報を入力します。

    | 設定 | 値 |
    | --- | --- |
    | **サブスクリプション** | Azure サブスクリプションを選択します。 |
    | **リソース グループ** | リソース グループを選択または作成します。 |
    | **データベース名** | *AdventureWorksLT* |
    | **[サーバー]** | **[新規作成]** を選択し、一意の名前で新しいサーバーを作成します。 **[場所]** を選択します。 認証には、次のいずれかのオプションを選択して、**[OK]** を選択します: |

    > &#128221; **認証は省略できません。** 組織のセキュリティ ポリシーに適合する方法を選択する必要があります。 各オプションは、後でデータベースに接続する方法に影響します。
    > - **Microsoft Entra 認証のみを使用する** (推奨): 組織で Entra ベースのアクセスが必須とされている場合は、これを選択します。** 自分の Azure アカウントを **Microsoft Entra 管理者**として設定してください。データベースへの接続には Microsoft Entra アカウントを使用します (たとえば、SSMS で [認証] = **[Microsoft Entra MFA]** を選択します)。**
    > - **SQL と Microsoft Entra の両方の認証/SQL 認証を使用する**: SQL 管理者としてログインする場合、または組織で両方の方法が許可されている場合は、これを選択します。 **[サーバー管理者ログイン]** と **[パスワード]** を入力します。 接続にはこれらの認証情報が必要です。 また、**[Microsoft Entra 管理者]** を設定して、SQL 認証に加えて Entra ログインを有効にすることもできます。

    > &#128221; 使用できるテスト サーバーが既にある場合は、新規に作成する代わりにそのサーバーを選択します。

1. **[SQL エラスティック プールを使用しますか?]** を **[いいえ]** に設定したままにします。
1. **[ワークロード環境]** で、**[開発]** を選択します。 これで、コンピューティングは、自動一時停止を有効にして **[汎用サーバーレス]** に事前設定されます。これは、最も費用効果の高い有料オプションです。
1. **[コンピューティングとストレージ]** で、 **[データベースの構成]** を選択します。 サービス レベルを **[ハイパースケール]** に、コンピューティング レベルを **[サーバーレス]** に変更します。 **[適用]** を選択して確定します。
1. **[バックアップ ストレージの冗長性]** の下で **[ローカル冗長バックアップ ストレージ]** を選択します。
1. **[Next: Networking]\(次へ: ネットワーク\)** を選択します。
1. **[ネットワーク]** タブの **[接続方法]** で、 **[パブリック エンドポイント]** を選択します。

    > &#128221; 新しいサーバーを作成する代わりに既存のサーバーを選択した場合、**[接続方法]** オプションは、そのサーバーで既に構成されているため、表示されないことがあります。

1. **[ファイアウォール規則]** の下の **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** を **[はい]** に設定し、**[現在のクライアント IP アドレスを追加する]** を **[はい]** に設定します。
1. **[次へ: セキュリティ]**、**[次へ: 追加設定]** の順に選択します。
1. **[追加設定]** タブの **[データソース]** で、**[既存のデータを使用する]** を *[サンプル]* に設定して AdventureWorksLT サンプル データベースを作成します。 確認を求められたら **[OK]** を選択します。
1. **[確認および作成]** を選択し、設定を確認してから、**[作成]** を選択します。
1. デプロイが完了するのを待って、新しい Azure SQL Database リソースに移動します。

---

## テスト ワークロードを作成する

AdventureWorksLT のサンプル データベースには、製品データと顧客データが含まれていますが、テーブルは小さいです。 意味のある実行プランとクエリ統計を生成するには、より大きなテーブルが必要です。 このセクションでは、1 年分の注文データをシミュレートする 80,000 行の `OrderHistory` テーブルを作成します。

1. SSMS を使用して Azure SQL Database に接続します。

    > &#128161; **[接続方法]** は、組織がサポートし、サーバーの作成時に構成された認証方法によって異なります。
    > - **Microsoft Entra 認証**: [認証] を **[Microsoft Entra MFA]** に設定し、Azure アカウントでサインインします。**
    > - **SQL 認証**: サーバーの作成時に指定した **[サーバー管理者ログイン]** と **[パスワード]** を入力し、[認証] を **[SQL ログイン]** に設定します。**
    >
    > どちらの場合も、**[サーバー名]** を `<your-server-name>.database.windows.net` に、**[データベース]** を **AdventureWorksLT** に設定します。
1. 次のスクリプトを実行して、`OrderHistory` テーブルを作成して事前設定します。

    ```sql
    DROP TABLE IF EXISTS dbo.OrderHistory;

    CREATE TABLE dbo.OrderHistory (
        OrderID INT IDENTITY(1,1) PRIMARY KEY,
        CustomerID INT NOT NULL,
        ProductID INT NOT NULL,
        OrderDate DATETIME NOT NULL,
        Quantity INT NOT NULL,
        UnitPrice DECIMAL(10,2) NOT NULL,
        TotalAmount AS (Quantity * UnitPrice) PERSISTED,
        Status NVARCHAR(20) NOT NULL
    );

    -- Insert 80,000 rows referencing real AdventureWorksLT customers and products
    INSERT INTO dbo.OrderHistory (CustomerID, ProductID, OrderDate, Quantity, UnitPrice, Status)
    SELECT TOP 80000
        c.CustomerID,
        p.ProductID,
        DATEADD(DAY, -ABS(CHECKSUM(NEWID())) % 365, GETDATE()),
        ABS(CHECKSUM(NEWID())) % 10 + 1,
        p.ListPrice,
        CASE ABS(CHECKSUM(NEWID())) % 4
            WHEN 0 THEN N'Pending'
            WHEN 1 THEN N'Processing'
            WHEN 2 THEN N'Shipped'
            ELSE N'Delivered'
        END
    FROM SalesLT.Customer AS c
    CROSS JOIN SalesLT.Product AS p
    ORDER BY NEWID();
    ```

    > &#128221; このスクリプトは、AdventureWorksLT サンプル データベースの実際の顧客と製品を組み合わせて、現実的な注文データを作成します。 `TotalAmount` 列は計算列です。つまり、エンジンによって `Quantity * UnitPrice` から自動的に計算されます。

1. テーブルが作成され、事前入力されたことを確認します。

    ```sql
    SELECT COUNT(*) AS TotalOrders FROM dbo.OrderHistory;
    GO
    ```

    > &#128221; **80,000** 行が表示されます。

---

## クエリ ストアを有効にして構成する

クエリ ストアは、クエリ テキスト、実行プラン、ランタイム統計をデータベース内で直接キャプチャします。 クエリの実行を開始する前に、推奨設定で有効にします。これにより、この時点以降、すべてがキャプチャされます。

1. 次のスクリプトを実行して、クエリ ストアを有効にし、以前のデータを消去します。

    ```sql
    ALTER DATABASE CURRENT SET QUERY_STORE = ON (
        OPERATION_MODE = READ_WRITE,
        QUERY_CAPTURE_MODE = AUTO,
        WAIT_STATS_CAPTURE_MODE = ON
    );
    GO

    -- Clear any prior data so only this exercise's queries appear
    ALTER DATABASE CURRENT SET QUERY_STORE CLEAR;
    GO
    ```

    > &#128221; クエリ ストアをクリアすると、この演習のクエリのみが表示されるため、後で回帰を見つけやすくなります。 この sh

1. クエリ ストアがアクティブであることを確認します。

    ```sql
    SELECT actual_state_desc, desired_state_desc,
        current_storage_size_mb, max_storage_size_mb
    FROM sys.database_query_store_options;
    ```

    > &#128221; `actual_state_desc` には `READ_WRITE` が表示されます。 `READ_ONLY` が表示される場合は、クエリ ストアの領域が不足しています。 `ALTER DATABASE CURRENT SET QUERY_STORE (MAX_STORAGE_SIZE_MB = 200);` を使用して、最大ストレージ サイズを増やします。

---

## 実行プランを分析して不足しているインデックスを追加する

実行プランには、オプティマイザーがクエリに対して選択した正確な演算子、データ アクセス方法、コスト見積もりが表示されます。 このセクションでは、パフォーマンスの低いクエリを実行し、プランを読んで原因を理解し、カバリング インデックスを使用して修正します。

1. SSMS で、**[実際の実行プランを含める]** (Ctrl+M) を選択してプランのキャプチャを有効にします。

1. カスタマーサービス チームが顧客の最近の注文を検索するために使用する次のクエリを実行します。

    ```sql
    SET STATISTICS IO ON;

    SELECT
        oh.OrderID,
        oh.OrderDate,
        p.Name AS ProductName,
        oh.Quantity,
        oh.UnitPrice,
        oh.TotalAmount,
        oh.Status
    FROM dbo.OrderHistory AS oh
    INNER JOIN SalesLT.Product AS p
        ON oh.ProductID = p.ProductID
    WHERE oh.CustomerID = 29485
        AND oh.OrderDate >= DATEADD(MONTH, -3, GETDATE())
    ORDER BY oh.OrderDate DESC;

    SET STATISTICS IO OFF;
    ```

1. **[実行プラン]** タブを選択し、プランを調べます。 次に該当しないか確認してください。

    - `OrderHistory` の **クラスター化インデックス スキャン**。 `CustomerID` または `OrderDate` にインデックスがないため、エンジンはすべての行を読み取ります。
    - スキャン演算子の**推定行数と実際の行数**。 スキャン演算子にカーソルを合わせ、これらの値を比較します。 大きな差は、統計が古いことを示します。
    - プランの上部にある "**不足しているインデックス候補**" という緑色のテキスト。このメッセージは、オプティマイザーが、インデックスがこのクエリに役立つと考えていることを意味します。

1. 実行プラン上の任意の場所を右クリックし、**[不足しているインデックスの詳細...]** を選択します。SSMS で、次のようなスクリプト化された `CREATE INDEX` ステートメントを含む新しいクエリ ウィンドウが開きます。

    ```sql
    /*
    Missing Index Details from SQLQuery123.sql - <yourservername>.database.windows.net.AdventureWorksLT (<username>) (51)
    The Query Processor estimates that implementing the following index could improve the query cost by 95.0604%.
    */

    /*
    USE [AdventureWorksLT]
    GO
    CREATE NONCLUSTERED INDEX [<Name of Missing Index, sysname,>]
    ON [dbo].[OrderHistory] ([CustomerID],[OrderDate])
    INCLUDE ([ProductID],[Quantity],[UnitPrice],[TotalAmount],[Status])
    GO
    */
    ```

    > &#128221; オプティマイザーは、どの列をキー列とし、どの列をインクルード列とするかを正確に指示します。 `TotalAmount` は、保存される計算列であっても INCLUDE リストに含まれていることに注意してください。 このパーセンテージ推定値は、このインデックスを追加した場合にオプティマイザーが予想するクエリ コストの削減額を示します。

1. **[メッセージ]** タブに切り替えて、`OrderHistory` の `logical reads` 値をメモします。 この値は、クエリを実行するためにエンジンが読み取った 8 KB ページの数です。

1. カバリング インデックスを作成します。 プランの提案をそのまま使用するのではなく、`OrderDate` の `DESC` 並べ替えを追加して、追加の並べ替えなしで最新の注文を最初に読み込むようにします。

    ```sql
    CREATE NONCLUSTERED INDEX IX_OrderHistory_CustomerDate
    ON dbo.OrderHistory (CustomerID, OrderDate DESC)
    INCLUDE (ProductID, Quantity, UnitPrice, Status);
    ```

    > &#128221; このインデックスは `CustomerID` と `OrderDate DESC` で複合キーを使用します。 エンジンは顧客の行を直接検索でき、`DESC` 順は、追加の並べ替えを行わずに最新の注文が最初に読み込まれることを意味します。 インクルード列は、クラスター化インデックスへのキー参照を防止するため、これは**カバリング インデックス**になります。

1. 同じクエリをもう一度実行します。 実行プランで、クラスター化インデックス スキャンの代わりに `IX_OrderHistory_CustomerDate` の **インデックス シーク**が表示されるようになりました。 **[メッセージ]** タブで、新しい `logical reads` 値を前の値と比較します。

    > &#128221; 論理読み取り数は劇的に減少します。 読み取り数が少なくなれば、I/O が減少し、CPU 使用率が低下し、カスタマーサービス チームの応答時間が短縮されます。

---

## DMV を使用して最もコストが高いクエリを検出する

実行プランは一度に 1 つのクエリを調べます。 DMV を使用すると、データベース内のすべてのクエリが幅広く表示されるため、個々のプランを詳しく調べる前に、最もコストの高いクエリを検出するのに役立ちます。

1. 次のクエリを実行して、平均 CPU 時間で上位 5 つのクエリを検出します。

    ```sql
    SELECT TOP 5
        qs.total_worker_time / qs.execution_count AS avg_cpu_time_us,
        qs.execution_count,
        qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
        SUBSTRING(st.text, (qs.statement_start_offset / 2) + 1,
            ((CASE qs.statement_end_offset
                WHEN -1 THEN DATALENGTH(st.text)
                ELSE qs.statement_end_offset
            END - qs.statement_start_offset) / 2) + 1) AS query_text
    FROM sys.dm_exec_query_stats AS qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS st
    ORDER BY avg_cpu_time_us DESC;
    ```

    > &#128221; 結果は、キャッシュされたクエリ プランの集計パフォーマンス統計を示します。 `avg_logical_reads` が多いクエリを探します。 返される行数に対して論理読み込み数が多いクエリは、インデックス チューニングの有力な候補です。

1. 既存の `IX_OrderHistory_CustomerDate` インデックスは、`CustomerID` でフィルター処理するクエリを対象としますが、倉庫チームは `Status` でフィルター処理し、`Product` テーブルに結合するレポートも実行しています。 このクエリを数回実行して、不足しているインデックスをオプティマイザーが登録できるようにします。

    ```sql
    SELECT
        p.Name AS ProductName,
        p.ProductNumber,
        oh.OrderDate,
        oh.Quantity,
        oh.TotalAmount
    FROM dbo.OrderHistory AS oh
    INNER JOIN SalesLT.Product AS p
        ON oh.ProductID = p.ProductID
    WHERE oh.Status = N'Pending'
        AND oh.OrderDate >= DATEADD(MONTH, -1, GETDATE())
    ORDER BY oh.TotalAmount DESC;
    GO 5
    ```

    > &#128221; このクエリには `Status` と `OrderDate` のインデックスが必要です。これは、既存の `CustomerID` と `OrderDate` のインデックスとは異なります。 複数回実行すると、オプティマイザーは不足しているインデックス DMV に関する十分な統計を確実に蓄積できます。

1. 次に、不足しているインデックス DMV のクエリを実行して、オプティマイザーがワークロード全体に対して推奨する内容を確認します。

    ```sql
    SELECT TOP 10
        mid.statement AS table_name,
        mid.equality_columns,
        mid.inequality_columns,
        mid.included_columns,
        ROUND(migs.avg_total_user_cost * migs.avg_user_impact *
            (migs.user_seeks + migs.user_scans), 2) AS improvement_measure
    FROM sys.dm_db_missing_index_groups AS mig
    INNER JOIN sys.dm_db_missing_index_group_stats AS migs
        ON migs.group_handle = mig.index_group_handle
    INNER JOIN sys.dm_db_missing_index_details AS mid
        ON mig.index_handle = mid.index_handle
    ORDER BY improvement_measure DESC;
    ```

    > &#128221; `Status` を等値列、`OrderDate` を非等値列とする推奨事項が表示されます。 `improvement_measure` は、クエリの平均コスト、推定改善率、クエリの実行頻度を組み合わせたものです。 値が大きいほど、利点も大きくなります。 これらは指示ではなく推奨事項です。 新しいインデックスを運用環境に追加する前に、読み取りと書き込みの両方のパフォーマンスを必ずテストしてください。

---

## クエリ ストアを使用してプランの回帰をシミュレートして検出する

パラメーター スニッフィングは、プランの回帰の一般的な原因です。 オプティマイザーは、最初に検出したパラメーター値に基づいてプランをコンパイルし、そのプランをキャッシュします。 今後のすべての呼び出しでは、別のプランの方がパフォーマンスが優れている場合でも、キャッシュされたプランが再利用されます。 このセクションでは、ストアド プロシージャを使用してパラメーター スニッフィングの回帰を発生させ、クエリ ストア GUI でそれを検出し、より適切な実行プランを強制します。

### 回帰シナリオを設定する

1. クエリ ストアをクリアし、すべてのストアド プロシージャ呼び出しが記録されるように再構成します:

    ```sql
    ALTER DATABASE CURRENT SET QUERY_STORE CLEAR;
    GO

    ALTER DATABASE CURRENT SET QUERY_STORE (
        QUERY_CAPTURE_MODE = ALL
    );
    GO
    ```

    > &#128221; `QUERY_CAPTURE_MODE = ALL` により、既定の `AUTO` モードではスキップされる可能性のある短いストアド プロシージャ呼び出しも含め、すべての実行が確実に記録されます。 運用環境では、通常、既定の `AUTO` キャプチャ モードを維持して、よりコストの高いクエリに集中します。

1. カバリング インデックスを非カバリング インデックスに置き換えます。 INCLUDE 列がない場合、オプティマイザーは、キー参照によるインデックス シーク (行数が少ない場合は高速) とフル テーブル スキャン (キー参照が多い場合は低コスト) のどちらを実行するかを決定する必要があります。 この決定が、パラメーター スニッフィングを可能にする理由です:

    ```sql
    DROP INDEX IF EXISTS IX_OrderHistory_CustomerDate ON dbo.OrderHistory;
    ```

    > &#128221; インデックスを削除すると、新しい非カバリング インデックスを競合なく作成できることが保証されます。

    ```sql
    CREATE NONCLUSTERED INDEX IX_OrderHistory_CustomerDate
    ON dbo.OrderHistory (CustomerID, OrderDate DESC);
    ```

    > &#128221; 前のセクションのカバリング インデックスは、一致した行数に関係なく、常にインデックス シークを使用しました。 INCLUDE 列を削除すると、オプティマイザーはキー参照のコストを比較検討する必要が生じ、推定される行数の影響を受けやすくなります。

1. 偏ったデータを追加して、ある顧客の注文数が他の顧客よりもはるかに多くなるようにします。 この偏ったデータは、オプティマイザーが顧客ごとに異なるプランを選択する理由を与えることで、パラメーター スニッフィングの条件を作成します。

    ```sql
    INSERT INTO dbo.OrderHistory (CustomerID, ProductID, OrderDate, Quantity, UnitPrice, Status)
    SELECT TOP 50000
        1,
        p.ProductID,
        DATEADD(DAY, -ABS(CHECKSUM(NEWID())) % 365, GETDATE()),
        ABS(CHECKSUM(NEWID())) % 10 + 1,
        p.ListPrice,
        N'Pending'
    FROM SalesLT.Product AS p
    CROSS JOIN SalesLT.Product AS p2
    ORDER BY NEWID();
    ```

    > &#128221; CustomerID 1 は 50,000 行を超えるようになったのに対して、他のほとんどの顧客は 100 行未満です。 この極端な偏りによって、オプティマイザーは異なるパラメーター値に対して異なるアクセス方法を選択するようになります。

1. 統計情報を更新して、オプティマイザーが新しいデータ分布を認識できるようにします。

    ```sql
    UPDATE STATISTICS dbo.OrderHistory;
    ```

1. 顧客注文クエリをラップするストアド プロシージャを作成します。 ストアド プロシージャは実行プランをキャッシュするため、パラメーター スニッフィングが発生します。

    ```sql
    CREATE OR ALTER PROCEDURE dbo.GetCustomerOrders
        @CustomerID INT
    AS
    BEGIN
        SELECT
            oh.OrderID,
            oh.OrderDate,
            p.Name AS ProductName,
            oh.Quantity,
            oh.UnitPrice,
            oh.TotalAmount,
            oh.Status
        FROM dbo.OrderHistory AS oh
        INNER JOIN SalesLT.Product AS p
            ON oh.ProductID = p.ProductID
        WHERE oh.CustomerID = @CustomerID
        ORDER BY oh.OrderDate DESC;
    END;
    ```

### 回帰の原因

1. プロシージャ キャッシュをクリアし、**CustomerID 29485** (行数が 100 行未満) でプロシージャを実行して、**高速プラン**をコンパイルします。 オプティマイザーは**インデックス シーク**を含むプランをコンパイルします。

    ```sql
    ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE;
    GO

    EXEC dbo.GetCustomerOrders @CustomerID = 29485;
    GO 10
    ```

1. 次にキャッシュをクリアし、**CustomerID 1** (50,000 行以上) を指定して呼び出し、**低速プラン**をコンパイルします。 テーブル全体をスキャンする方が 50,000 回のキー参照を実行するよりも低コストであるため、オプティマイザーは**クラスター化インデックス スキャン**をコンパイルします。

    ```sql
    ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE;
    GO

    EXEC dbo.GetCustomerOrders @CustomerID = 1;
    ```

1. **CustomerID 29485** でプロシージャをもう一度実行します。 オプティマイザーは、この顧客にとってシーク プランの方が高速であったにもかかわらず、**キャッシュされたスキャン プランを再利用**します。

    ```sql
    EXEC dbo.GetCustomerOrders @CustomerID = 29485;
    GO 10
    ```

    > &#128221; これが回帰です。 50,000 行用にコンパイルされたプランは、データが 100 行未満の顧客に再利用されています。 クラスター化インデックス スキャンは、必要な行が少量でもテーブル内のすべての行を読み取ります。 本番環境では、多くの場合、これは、プラン キャッシュのリサイクル後に "昨日は速く、今日は遅い" クエリとして現れます。

1. クエリ ストアのデータをディスクにフラッシュして、GUI で結果が即座に表示できるようにします。

    ```sql
    EXEC sp_query_store_flush_db;
    ```

### クエリ ストア GUI を使用して回帰を検出し、修正する

1. SSMS オブジェクト エクスプローラーで、[データベース] ノードを展開します。
1. **[クエリ ストア]** フォルダーを展開します。
1. **[最もリソース消費量の多いクエリ]** を選択します。

    > &#128221; このビューには、最も多くのリソースを消費するクエリが、選択したメトリックで並べ替えて表示されます。 最初に調査するクエリがすぐにわかるため、これは、パフォーマンス チューニングの最も一般的な出発点です。

1. まだ設定していない場合は、上部のドロップダウンを使用して、メトリックを **[期間 (ミリ秒)]**、統計を **[合計]**、時間の範囲を **[過去 1 時間]** に設定します。 時間の間隔は、*[構成]* ボタンを選択すると見つかります。
1. 左側のペインにある最も高いバーを選択します。 このスクリプトは `GetCustomerOrders` クエリです。 右側のパネルには、**[プランの要約]** グラフが表示され、各円は異なる実行プランを表します。
1. 期間が非常に異なるプランが 2 つあることを確認します。 **期間が長い方**の円 (スキャン プラン) を選択します。 下部にプランの詳細が表示されます。 **Clustered Index Scan** 演算子に注意してください。
1. **持続時間が短い方**の円 (シーク プラン) を選択します。 **Index Seek** 演算子に注意してください。
1. 速い方のプランを選択し、ツールバーの **[プランの強制]** ボタンを選択します。 確認ダイアログが表示されます。 **[はい]** を選択します。

    > &#128221; プラン強制は、渡されたパラメーター値に関係なく、このクエリに対して常にシーク プランを使用するようにオプティマイザーに指示します。 インデックスは依然として存在しているため、強制されたプランは正常に実行されます。 これは、アプリケーション コードやストアド プロシージャを変更する必要のないクイック修正です。

1. 次に、クエリ ストア フォルダーの **[プランが強制されたクエリ]** に移動します。 クエリが、強制されたプランの一覧に表示されていることを確認します。

### T-SQL を使用してプラン強制を検証する

1. 強制されたプランを確認します。

    ```sql
    SELECT
        q.query_id,
        p.plan_id,
        p.is_forced_plan,
        p.force_failure_count,
        qt.query_sql_text
    FROM sys.query_store_plan AS p
    INNER JOIN sys.query_store_query AS q
        ON p.query_id = q.query_id
    INNER JOIN sys.query_store_query_text AS qt
        ON q.query_text_id = qt.query_text_id
    WHERE p.is_forced_plan = 1;
    ```

    > &#128221; `is_forced_plan` 列には **1** が表示されます。 プランが依存するインデックスがまだ存在するため、`force_failure_count` は **0** になります。 この値が 0 以外の場合、強制されたプランは実行できず、オプティマイザーは別のプランにフォールバックしていることを意味します。

1. プロシージャをもう一度実行して、強制されたプランをテストします。 オプティマイザーは、パラメーター値に関係なく、強制されたシーク プランを使用します。

    ```sql
    EXEC dbo.GetCustomerOrders @CustomerID = 29485;
    ```

1. そこでもシーク プランが使用されていることを確認するために、今度は別のパラメーター値でプロシージャを実行してみましょう。

    ```sql
    EXEC dbo.GetCustomerOrders @CustomerID = 1;
    ```

    > &#128221; CustomerID 1 には 50,000 以上の行がありますが、シーク プランが強制されているため、オプティマイザーはそれを使用します。 これにより、その顧客のパフォーマンスが低下する可能性がありますが、他のすべての顧客の回帰が防止されます。 運用環境では、通常、クエリの書き換えやインデックスの追加など、より永続的な修正に取り組んでいる間は、これを一時的な軽減策として使用します。

1. 両方のパラメーター値の実行プランを確認します。 @CustomerID = 29485 の場合、SELECT 演算子で、割り当てられたメモリが多すぎるという警告が表示されることがあります。 @CustomerID = 1 の場合、並べ替え演算子で、ディスクへの一時的なデータ流出が発生することがあります。


1. 残りの演習のために、カバリング インデックスを復元し、ストアド プロシージャをクリーンアップします:

    ```sql
    DROP PROCEDURE IF EXISTS dbo.GetCustomerOrders;

    DROP INDEX IX_OrderHistory_CustomerDate ON dbo.OrderHistory;

    CREATE NONCLUSTERED INDEX IX_OrderHistory_CustomerDate
    ON dbo.OrderHistory (CustomerID, OrderDate DESC)
    INCLUDE (ProductID, Quantity, UnitPrice, Status);
    ```

---

## クエリ ストアのヒントを適用する

アプリケーション コードを変更せずにクエリの実行を形成することが必要な場合があります。 クエリ ストアのヒントを使用すると、クエリ ストアを通じて特定のクエリにヒントを添付できます。 このセクションでは、並列化された負荷の高い集計クエリを実行し、その後、`MAXDOP 1` ヒントを適用してそれを単一のスレッドに強制します。

1. SSMS で、まだ有効になっていない場合は、**[実際の実行プランを含める]** (Ctrl + M キー) を選択して有効にし、次のクエリを実行します。 このクエリは、注文を顧客、製品、カテゴリ、状態別にグループ化し、異なるパーティションを持つ複数のウィンドウ関数を適用します。 各パーティションには個別の並べ替えが必要であるため、クエリ コストが上昇します。

    ```sql
    SELECT
        oh.CustomerID,
        p.Name AS ProductName,
        pc.Name AS CategoryName,
        oh.Status,
        COUNT(*) AS OrderCount,
        SUM(oh.TotalAmount) AS TotalRevenue,
        AVG(oh.TotalAmount) AS AvgOrderValue,
        STDEV(oh.TotalAmount) AS StdDevOrderValue,
        RANK() OVER (PARTITION BY oh.Status
            ORDER BY SUM(oh.TotalAmount) DESC) AS StatusRevenueRank,
        PERCENT_RANK() OVER (PARTITION BY pc.Name
            ORDER BY AVG(oh.TotalAmount)) AS CategoryPctRank,
        SUM(COUNT(*)) OVER (PARTITION BY oh.Status) AS StatusTotalOrders
    FROM dbo.OrderHistory AS oh
    INNER JOIN SalesLT.Product AS p
        ON oh.ProductID = p.ProductID
    INNER JOIN SalesLT.ProductCategory AS pc
        ON p.ProductCategoryID = pc.ProductCategoryID
    GROUP BY oh.CustomerID, p.Name, pc.Name, oh.Status
    ORDER BY TotalRevenue DESC;
    ```

1. **[実行プラン]** タブを選択します。ハッシュ一致、並べ替え、クラスター化インデックス スキャンなどの演算子で並列処理を示す矢印 (小さな黄色の矢印) を探します。 これらの矢印は、オプティマイザーが並列プランを選択したことを示します。 また、複数のスレッドからの結果を結合する **Gather Streams** 演算子も表示されます。

    > &#128221; `CustomerID, ProductName, CategoryName, Status` 別にグループ化すると、数千ものグループが生成され、3 つのウィンドウ関数が異なる `PARTITION BY` 列を使用するため、このクエリはコストが高くなります。 個々のパーティションごとに、結果に対して個別の並べ替えパスが必要です。 見積コストが並列処理のコストしきい値 (既定値は 5) を超えるため、オプティマイザーは並列プランを選択します。

1. クエリ ストアでこのクエリの `query_id` を見つけます。

    ```sql
    SELECT q.query_id, qt.query_sql_text
    FROM sys.query_store_query_text AS qt
    INNER JOIN sys.query_store_query AS q
        ON qt.query_text_id = q.query_text_id
    WHERE qt.query_sql_text LIKE '%RevenueRank%PARTITION%'
        AND qt.query_sql_text NOT LIKE '%query_store%';
    ```

    > &#128221; `query_id` 値をメモします。 この値は、次の手順でヒントを適用してクリアするために使用します。

1. `MAXDOP 1` ヒントを適用して、クエリを単一のスレッドで強制的に実行します。

    ```sql
    EXEC sp_query_store_set_hints
        @query_id = <query_id>,
        @query_hints = N'OPTION (MAXDOP 1)';
    ```

    > &#128221; `<query_id>` を前の手順の実際の値に置き換えます。 `MAXDOP 1` ヒントは、クエリを単一のスレッドで強制的に実行します。 これは、並列処理のオーバーヘッドが利点を上回るクエリや、共有サーバーでの CPU 消費を削減する場合に役立ちます。

1. 集計クエリをもう一度実行し、実行プランを確認します。 並列処理の矢印が消えます。 すべての演算子は単一のスレッドで実行され、Gather Streams 演算子は表示されなくなりました。

    > &#128221; 実行時間を比較します。 このクエリの場合、集計関数とウィンドウ関数で複数のスレッドを利用できるため、並列プランの方が高速になる可能性があります。 シングル スレッド プランの場合、全体的に使用される CPU は少なくなりますが、実時間は長くなります。 このトレードオフこそが、まさに`MAXDOP` で制御されるものです。

1. テストが完了したら、ヒントを削除します。

    ```sql
    EXEC sp_query_store_clear_hints @query_id = <query_id>;
    ```

    > &#128221; クエリ ストアのヒントは、ステートメント レベルのヒントとプラン ガイドをオーバーライドします。 クエリ ストアのヒントを使用すると、アプリケーション コードに手を加えることなく、クエリの動作を完全に制御できます。 本番環境では、これらのヒントを慎重に使用し、どのクエリにヒントが適用されているかを文書化してください。

---

## 障害となっている問題を特定して解決する

Azure SQL Database では、Read Committed Snapshot Isolation (RCSI) が既定で有効になっているため、読み取り操作によって書き込みがブロックされることはありません。 ただし、同じ行を対象とする 2 つの書き込み操作は、依然として相互にブロックし合います。 このセクションでは、倉庫チームから報告されたブロック シナリオをシミュレーションし、DMV を使用してそれを診断します。

1. SSMS で、すべて同じデータベースに接続された **3 つの個別のクエリ ウィンドウ**をそれぞれ開きます。 それらを頭の中でウィンドウ 1、ウィンドウ 2、ウィンドウ 3 とラベル付けします。

1. **ウィンドウ 1** で、注文の状態を更新するが、コミットはしないトランザクションを開始します。 このクエリは、トランザクションの途中でクラッシュした倉庫アプリケーションをシミュレートします。

    ```sql
    BEGIN TRANSACTION;
    UPDATE dbo.OrderHistory SET Status = N'Cancelled' WHERE OrderID = 1;
    -- Simulating an application that stopped without committing
    ```

1. **Window 2** で、同じ行を更新してみます。 このクエリは、別の倉庫作業員が同じ注文を処理する場合をシミュレートします。

    ```sql
    UPDATE dbo.OrderHistory SET Status = N'Shipped' WHERE OrderID = 1;
    ```

    > &#128221; このクエリはハングします。 ウィンドウ 1 がその行に対して排他ロックを保持しており、コミットしていないため、ブロックされます。

1. **ウィンドウ 3** で、ブロック チェーンを特定し、ヘッド ブロッカーの実行内容を調べます。

    ```sql
    -- Find blocked sessions
    SELECT
        r.session_id AS blocked_session,
        r.blocking_session_id AS head_blocker,
        r.wait_type,
        r.wait_time AS wait_time_ms,
        r.wait_resource,
        t.text AS blocked_query
    FROM sys.dm_exec_requests AS r
    CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
    WHERE r.blocking_session_id <> 0;
    ```

    > &#128221; `wait_type` は `LCK_M_X` になります (排他ロックを待っています)。 `wait_resource` は正確な行を識別します。 `head_blocker` 列は、調査するセッションを示します。

1. 引き続き**ウィンドウ 3** で、ヘッド ブロッカー セッションの実行内容を調べます。

    ```sql
    -- Investigate the head blocker
    SELECT
        s.session_id,
        s.status,
        s.login_time,
        s.program_name,
        s.host_name,
        t.text AS last_query,
        c.connect_time,
        s.last_request_start_time,
        s.last_request_end_time
    FROM sys.dm_exec_sessions AS s
    LEFT JOIN sys.dm_exec_connections AS c
        ON s.session_id = c.session_id
    CROSS APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) AS t
    WHERE s.session_id = <blocking_session_id>;
    ```

    > &#128221; `<blocking_session_id>` を、前の結果の値に置き換えます。 `status` 列に `sleeping` が表示されていることに注意してください。これは、セッションがステートメントを実行したが、トランザクションが開いたままアイドル状態になっていることを意味します。 これは最も一般的なブロック パターンの 1 つです。トランザクションをコミットしないままセッションがスリープ状態になっています。

1. **ウィンドウ 1** で、トランザクションをロールバックしてブロックを解決します:

    ```sql
    ROLLBACK TRANSACTION;
    ```

1. **ウィンドウ 2** に切り替えて、更新が完了したことを確認します。

    > &#128221; 運用環境では、必要最小限​​のステートメントのみを実行し、即座にコミットして、トランザクションの保持期間を短くしてください。 T-SQL で `TRY...CATCH` ブロックを使用し、`CATCH` ブロック内でロールバックして、ランタイム エラーが発生してもトランザクションが開いたままにならないようにします。 トランザクションを開始するストアド プロシージャで、`SET XACT_ABORT ON` を使用することを検討してください。これにより、ランタイム エラーが発生した場合に、開いているトランザクションが自動的にロールバックされます。 孤立したセッションが他のセッションをブロックしている場合は、`KILL <session_id>;` を使用して、そのセッションを終了することができます。

---

## クリーンアップ

この演習中に作成したテストテーブルとストアド プロシージャを削除します。

```sql
DROP PROCEDURE IF EXISTS dbo.GetCustomerOrders;
DROP TABLE IF EXISTS dbo.OrderHistory;
```

強制されたプランを強制解除し、クエリ ストアのヒントをクリアして、データベースをクリーンな状態のままにします。

```sql
-- Unforce all plans
DECLARE @qid INT, @pid INT;

SELECT TOP 1 @qid = q.query_id, @pid = p.plan_id
FROM sys.query_store_plan AS p
INNER JOIN sys.query_store_query AS q ON p.query_id = q.query_id
WHERE p.is_forced_plan = 1;

WHILE @@ROWCOUNT > 0
BEGIN
    EXEC sp_query_store_unforce_plan @query_id = @qid, @plan_id = @pid;

    SELECT TOP 1 @qid = q.query_id, @pid = p.plan_id
    FROM sys.query_store_plan AS p
    INNER JOIN sys.query_store_query AS q ON p.query_id = q.query_id
    WHERE p.is_forced_plan = 1;
END
```

> &#128221; この Azure SQL Database リソースは、コース内の他のラボでも使用できます。 すべてのラボが完了したら、データベースを削除できます。または、リソース グループをこのラボ用に作成し、その中にこのデータベースのみが含まれている場合は、リソース グループ全体を削除できます。 リソース グループを削除すると、コストが発生する可能性のあるリソースが稼働し続けるのを確実に防ぐことができます。

---

これでこの演習は完了です。

この演習では、Azure SQL Database でのクエリのパフォーマンス問題を調査しました。 実行プランを分析して、不足しているインデックスを特定し、論理読み取りを大幅に削減するカバリング インデックスを作成しました。 DMV を使用して、ワークロード全体で最もコストの高いクエリと不足しているインデックスの推奨事項を検出しました。 ストアド プロシージャと偏ったデータを使用してパラメーター スニッフィング回帰をシミュレートし、その後、SSMS の [最もリソース消費量の多いクエリ] ビューを使用して回帰を検出し、より適切な実行プランを強制しました。 アプリケーション コードを変更せずに並列処理を制御するためにクエリ ストアのヒントを適用しました。 最後に、DMV を使用してヘッド ブロッカーを特定し、ライター間のブロック シナリオを診断し、スリーピング状態のセッションを調査して、それを解決しました。
