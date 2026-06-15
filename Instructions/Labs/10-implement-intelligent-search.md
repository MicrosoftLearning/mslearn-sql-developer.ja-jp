---
lab:
  title: ラボ 10 - インテリジェント検索をフルテキスト、ベクトル、ハイブリッド クエリで実装する
  module: Design and implement models and embeddings with SQL
  description: この演習では、フルテキスト、ベクトル、ハイブリッドという検索アプローチを Azure SQL Database で Reciprocal Rank Fusion (RRF) を使用して実装する方法を学習します。
  level: 300
  duration: 45 minutes
  islab: true
  primarytopics:
    - Azure SQL Database
    - Vector Search
    - Full-Text Search
    - Azure OpenAI
---

# インテリジェント検索をフルテキスト、ベクトル、ハイブリッド クエリで実装する

**予測される所要時間**: 45 分

この演習では、Azure SQL Database でのさまざまな検索アプローチを実装します。 フルテキスト インデックスを作成し、格納済みの埋め込みを使ってベクトル検索を実行し、さらに両方の手法を組み合わせたハイブリッド検索を Reciprocal Rank Fusion (RRF) を使用して行います。 その後で、同じクエリが各検索アプローチでどのように処理されるかを比較し、結果の違いを観察します。

あなたは Adventure Works のデータベース開発者であるとします。 あなたのチームは製品検索を改善したいと考えています。顧客が関連製品を見つけるときに、キーワード完全一致でも、求めているものを自然言語で説明するという方法でも検索できるようにするためです。 あなたが実装するのはキーワード マッチングのためのフルテキスト検索、セマンティック的類似性のためのベクトル検索、およびこの両方のアプローチを組み合わせたハイブリッド検索です。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- [Azure サブスクリプション](https://azure.microsoft.com/free) ([Azure OpenAI へのアクセス](/legal/cognitive-services/openai/limited-access)が承認されていること)
- Visual Studio Code と SQL Server (mssql) 拡張機能、または SQL Server Management Studio
- Azure SQL Database と T-SQL に関する基本的な知識

---

## Azure SQL Database をプロビジョニングする

最初に、Azure SQL データベースをサンプル データ付きで作成します。

> &#128221; 既に AdventureWorksLT という Azure SQL データベースがプロビジョニングされている場合は、このセクションをスキップしてください。

1. [Azure SQL ハブ](https://aka.ms/azuresqlhub)に移動して、Azure アカウントでサインインします (要求された場合)。 **[Azure SQL Database]** ペインの **[オプションの表示]** を選択し、**[SQL データベースの作成]** を選択します。

    > &#128161; このページに**無料オファー**のバナーがある場合は、これを適用して Azure SQL Database を無料で使用できます。 この[無料オファー](https://learn.microsoft.com/azure/azure-sql/database/free-offer)では、サーバーレス コンピューティング 100,000 仮想コア秒とストレージ 32 GB (1 か月あたり) を利用できます。 無料オファーを適用する場合は、ステップ 3 から 6 までをスキップしてください。

1. **[SQL データベースの作成]** ページの **[基本]** タブに、必要な情報を入力します。

    | 設定 | 値 |
    | --- | --- |
    | **サブスクリプション** | Azure サブスクリプションを選択します。 |
    | **リソース グループ** | リソース グループを選択または作成します。 |
    | **データベース名** | *AdventureWorksLT* |
    | **[サーバー]** | **[新規作成]** を選択して一意の名前を持つ新しいサーバーを作成します。 **[場所]** を選択します。 認証に関して、次のオプションの 1 つを選択してから **[OK]** を選択します。 |

    > &#128221; **認証は省略可能ではありません。** 所属する組織のセキュリティ ポリシーに適合する方法を選択する必要があります。 どのオプションを選ぶかに応じて、後でデータベースに接続する方法が異なります。
    > - **Microsoft Entra 認証のみを使用する** (推奨): これを選択するのは、所属組織が Entra ベースのアクセスを必須としている場合です。** ご自身の Azure アカウントを **Microsoft Entra 管理者**として設定してください。データベースへの接続には Microsoft Entra アカウントを使用します (たとえば SSMS では [認証]** = **[Microsoft Entra MFA]** を選択します)。
    > - **SQL と Microsoft Entra 両方の認証/SQL 認証を使用する**: これを選択するのは SQL 管理者としてログインしたい場合、または所属組織で両方の方法が許可されている場合です。 **[サーバー管理者ログイン]** と **[パスワード]** を入力します。 これらの認証情報が接続に必要です。 また、**[Microsoft Entra 管理者]** を設定しておくと、SQL 認証に加えて Entra ログインを有効にすることができます。
    >
    > ここで選ぶ認証方法は、あなたが開発者としてどのようにデータベースに接続するかを決定するものです。** Azure SQL が Azure OpenAI に接続する方法とは別のものであり、これについてはこのラボで後ほど構成します。

    > &#128221; あなたが使用できるテスト サーバーが既にある場合は、新規作成する代わりにそのサーバーを選択してください。

1. **[SQL エラスティック プールを使用しますか?]** を **[いいえ]** に設定したままにします。
1. **[ワークロード環境]** については、**[開発]** を選択します。 これでコンピューティングが **[General Purpose サーバーレス]** に事前設定され、自動一時停止が有効化されますが、これは有料オプションの中で最もコストを抑えられる構成です。
1. **[コンピューティングとストレージ]** で、 **[データベースの構成]** を選択します。 サービス レベルを **[Hyperscale]** に、コンピューティング レベルを **[サーバーレス]** に変更します。 **[適用]** を選択して確定します。
1. **[バックアップ ストレージの冗長性]** の下で **[ローカル冗長バックアップ ストレージ]** を選択します。
1. **[Next: Networking]\(次へ: ネットワーク\)** を選択します。
1. **[ネットワーク]** タブの **[接続方法]** で、 **[パブリック エンドポイント]** を選択します。

    > &#128221; 新規作成の代わりに既存のサーバーを選択した場合は、**[接続方法]** オプションが表示されないことがありますが、これはそのサーバー上で既に構成されているためです。

1. **[ファイアウォール規則]** の下の **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** を **[はい]** に設定し、**[現在のクライアント IP アドレスを追加する]** を **[はい]** に設定します。
1. **[次へ: セキュリティ]** を選択し、**[次へ: 追加設定]** を選択します。
1. **[追加設定]** タブの **[データソース]** の下で、AdventureWorksLT サンプル データベースを作成するために **[既存のデータを使用します]** を [サンプル] に設定します。** 確認を求められたら **[OK]** を選択します。
1. **[確認および作成]** を選択し、設定を確認してから、**[作成]** を選択します。
1. デプロイが完了したら、この新しい Azure SQL Database リソースに移動します。

---

## Foundry プロジェクトを作成して Azure OpenAI モデルを展開する

次に、Microsoft Foundry の中にプロジェクトを作成し、チャット モデルと埋め込みモデルを展開します。

> &#128221; 既に Azure OpenAI リソースがあり、**gpt-5.4-mini** と **text-embedding-3-small** のモデルが展開済みの場合は、このセクションをスキップしてください。

### Foundry プロジェクトを作成する

最初のステップは、Microsoft Foundry の中に新しいプロジェクトを作成することですが、これは Azure OpenAI モデルの展開と管理のためのワークスペースとなります。

1. [Microsoft Foundry](https://ai.azure.com) に移動して Azure アカウントでサインインします。

    > &#128221; **[新しい Foundry]** トグルがポータルの右上隅にある場合は、**[オン]** になっていることを確認してください。これは Foundry ポータルの最新バージョンを使用するためです。 以降のステップは、この新しいエクスペリエンスを前提としています。

1. 新しいプロジェクトを作成します。
    1. 左上隅にプロジェクト名が表示されている場合は、それを選択して **[新しいプロジェクトの作成]** を選択します。 プロジェクトが存在しない場合は、ホーム ページから **[+ プロジェクトの作成]** を選択します。
    1. プロジェクトの名前を入力します (たとえば *proj-sqlailab*)。
    1. **[詳細オプション]** を展開します。 **[サブスクリプション]** と **[リソース グループ]** は自分の Azure SQL データベースに使用したのと同じものを選択し、**[リージョン]** は Azure OpenAI モデルが利用可能なリージョンを選択します。
    1. **［作成］** を選択します **[始めましょう]** のプロンプトが表示された場合は、**[始めましょう]** を選択します。

### Azure OpenAI モデルのデプロイ
この演習では 2 つのモデルを展開します。**gpt-5.4-mini** (チャット補完用) と **text-embedding-3-small** (埋め込み用) です。 以降のステップで、両方のモデルを展開する方法を説明します。

最初に **gpt-5.4-mini** モデルを展開しましょう。

1. 上部のナビゲーション バーの **[検出]** を選択します。 ページの上部にある検索バーで **gpt-5.4-mini** を検索します。 別の方法として、**[注目のモデル]** セクションで **[すべてのモデルを表示]** を選択してカタログ全体を閲覧することもできます。 検索結果から **gpt-5.4-mini** を選択します。
1. モデル詳細ページで **[デプロイ]** ドロップダウンを選択して **[既定の設定]** を選択します。
1. **[デプロイするプロジェクトを選択する]** ダイアログが表示された場合は、プロジェクト作成時に選択したのと同じ **[リージョン]** を選択してから、自分のプロジェクトを **[プロジェクト]** ドロップダウンから選択します。 **[続行]** を選択します。
1. **[デプロイ名]** を **gpt-5.4-mini** に設定し、**[デプロイ]** を選択します。
1. デプロイが完了すると、モデルが展開された状態になります。

次に **text-embedding-3-small** モデルを展開しましょう。

1. ここでは埋め込みモデルを展開します。 上部のナビゲーションで **[検出]** を選択し、**text-embedding-3-small** を検索します。 それを検索結果から選択します。
1. モデル詳細ページで **[デプロイ]** ドロップダウンを選択して **[既定の設定]** を選択します。
1. **[展開するプロジェクトを選択する]** ダイアログが表示されたら、前と同じ **[リージョン]** と **[プロジェクト]** を選択して **[続行]** を選択します。
1. デプロイが完了すると、モデルが展開された状態になります。

1. 上部のナビゲーションで **[ビルド]** > **[モデル]** を選択して、両方の展開が表示されていることを確認します。

### Azure OpenAI エンドポイントを取得する

ここで、**Azure OpenAI エンドポイント**を取得します。これは後の T-SQL のステップに必要になります。 

1. [Azure portal](https://portal.azure.com/) で、自分のリソース グループに移動して、プロジェクトとともに作成された **Foundry** リソース (たとえば *proj-sqlailab-resource*) を選択します。 
1. 左側のメニューで **[リソース管理]** > **[キーとエンドポイント]** を選択します。 **[OpenAI]** タブを選択し、エンドポイントの URL (たとえば `https://proj-sqlailab-resource.openai.azure.com/`) に注目します。 このエンドポイント名 (`.openai.azure.com` より前の部分) が後で必要になります。

    > &#128221; **[キーとエンドポイント]** ページには **[Foundry]**、**[OpenAI]**、**[AI サービス]** の 3 つのタブがあります。 必ず **[OpenAI]** タブを選択してください。このタブには、Azure SQL Database の `CREATE EXTERNAL MODEL` と `sp_invoke_external_rest_endpoint` に必要な形式でエンドポイントが表示されます。 [Foundry] タブには、別の `.services.ai.azure.com` という URL が表示されますが、これらの T-SQL 機能には使用できません。

---

## マネージド ID アクセスを構成する

Azure SQL Database では Azure OpenAI での認証にシステム割り当てマネージド ID が使用されるため、あなたはこの ID を自分の SQL Server 上で有効にするとともに Azure OpenAI リソースへのアクセスを許可する必要があります。 この方法はセキュリティの点で API キーよりも優れており、シークレットの保存も必要ありません。

> &#128221; 既に自分の SQL Server でシステム割り当てマネージド ID が有効化されていて Azure OpenAI リソースに対する **Cognitive Services OpenAI ユーザー** ロールが付与されている場合は、このセクションをスキップしてください。

1. Azure portal で、先ほど作成した **SQL サーバー** (データベースではなく論理サーバー) に移動します。
1. 左側のメニューで、**[セキュリティ]** > **[ID]** を選択します。
1. **[システム割り当てマネージド ID]** の下の **[状態]** を **[オン]** に設定して **[保存]** を選択します。
1. マネージド ID が SQL Server 上で有効化された状態になったら、自分の **Azure OpenAI** リソース (たとえば *adventureworks-openai*) に移動します。
1. 左側のメニューで **[アクセス制御 (IAM)]** を選択します。
1. **[+ 追加]** を選択し、**[ロールの割り当ての追加]** を選択します。
1. **[ロール]** タブで、**Cognitive Services OpenAI ユーザー**を検索して選択し、**[次へ]** を選択します。
1. **[メンバー]** タブで、**[マネージド ID]** を選択してから **[+ メンバーを選択する]** を選択します。
1. **[マネージド ID の選択]** ペインで、**[マネージド ID]** を **[SQL サーバー]** に設定し、自分の SQL サーバーをリストから選択して **[選択]** を選択します。
1. **[確認と割り当て]** を 2 回選択して、ロールの割り当てを完了します。

    > &#128221; このロールの割り当てが有効となるまでに最大 5 分かかることがあります。 待つ間に次のステップに進んでもかまいません。

---

## ProductReview テーブルを作成する

AdventureWorksLT サンプル データベースには製品情報が格納されていますが、顧客レビューはありません。 このステップでは、あるスクリプトをダウンロードして実行することによって **ProductReview** テーブルを作成し、多数の製品カテゴリにわたる 140 件のリアルなレビューをここに格納します。 これらのレビューは、フルテキスト、ベクトル、ハイブリッド検索を使用して検索するのに適したリッチなテキスト データです。

1. 自分の Azure SQL Database に Visual Studio Code (SQL Server 拡張機能付き) または SQL Server Management Studio を使用して接続します。

    > &#128161; **接続方法**は、どの認証方法を所属組織がサポートしているか、およびどれがサーバー作成時に構成されたかによって決まります。
    > - **Microsoft Entra 認証**: SSMS では、[認証] を **[Microsoft Entra MFA]** に設定して Azure アカウントでサインインします。** VS Code では、接続プロファイルを作成するときに認証の種類として **[Microsoft Entra ID]** を選択します。
    > - **SQL 認証**: SSMS または VS Code で、**[サーバー管理者ログイン]** と **[パスワード]** にはサーバー作成時に指定したものを入力し、[認証] を **[SQL ログイン]** に設定します。**
    >
    > どちらの場合も、**[サーバー名]** を `<your-server-name>.database.windows.net` に、**[データベース]** を **AdventureWorksLT** に設定してください。
1. レビュー スクリプトを [**product-reviews-insert.sql**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-sql-developer/main/Allfiles/product-reviews-insert.sql) からダウンロードしてローカルに保存します。
1. ダウンロードしたファイルを開いて、このスクリプト全体を自分の **AdventureWorksLT** データベースに対して実行します。
1. テーブルが作成されてデータが事前設定されたことを確認するために、次のコマンドを実行します。

    ```sql
    SELECT COUNT(*) AS TotalReviews FROM dbo.ProductReview;
    GO
    ```

    > &#128221; 行数は **140** となるはずです。 レビューは自転車、タイヤ、ライト、ヘルメット、手袋、メンテナンス工具などについて書かれており、評価は 1 つから 5 つまでの星として付けられています。

---

## データベース スコープ資格情報と埋め込み用外部モデルを作成する

Azure OpenAI を T-SQL から呼び出すために必要な資格情報とモデル参照を設定します。 ここでは、あなたの Azure SQL Server のシステム割り当てマネージド ID を使用して資格情報を 1 つだけ作成します。 この資格情報が `CREATE EXTERNAL MODEL` (埋め込み用) と `sp_invoke_external_rest_endpoint` (チャット補完用) の両方で使用されるため、API キーは不要になります。

> &#128221; 既に Azure OpenAI エンドポイント用のデータベース スコープ資格情報と外部埋め込みモデルが構成されている場合は、このセクションをスキップしてください。

1. マネージド ID を使用してデータベース スコープ資格情報を作成するために、新しいクエリ ウィンドウを開いて次に示すスクリプトを実行します。 `<your-openai-endpoint>` を **[OpenAI]** タブに表示されているエンドポイント名に置き換えてください (たとえば、エンドポイントが `https://proj-sqlailab-resource.openai.azure.com/` の場合は *proj-sqlailab-resource* を使います)。

    ```sql
    -- Create a database master key if one doesn't exist
    IF NOT EXISTS (SELECT * FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
        CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<your strong password here>';
    GO

    -- Create a credential using Managed Identity
    CREATE DATABASE SCOPED CREDENTIAL [https://<your-openai-endpoint>.openai.azure.com]
    WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
    GO
    ```

    > &#128221; `<your strong password here>` を強力なパスワードに置き換えてください。 `<your-openai-endpoint>` を **[OpenAI]** タブに表示されているエンドポイント名に置き換えてください (たとえば *proj-sqlailab-resource*)。
    >
    > &#9888; **重要**: SECRET の中の `resourceid` の値を変更**しないでください**。 `https://cognitiveservices.azure.com` のままにしておく必要があります。 これは Azure Cognitive Services 用の固定 OAuth オーディエンスであり、あなたのエンドポイント URL 用ではありません。 変更すると、認証エラーが発生することになります。

1. ここで、埋め込みモデル用の外部モデル参照を作成します。 この参照を作成しておくと、`AI_GENERATE_EMBEDDINGS` を T-SQL で直接使用できるようになります。 `<your-openai-endpoint>` を実際のエンドポイント名で置き換えます。

    ```sql
    -- Create an external model reference for embeddings
    CREATE EXTERNAL MODEL my_embedding_model
    WITH (
        LOCATION = 'https://<your-openai-endpoint>.openai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-10-21',
        API_FORMAT = 'Azure OpenAI',
        MODEL_TYPE = EMBEDDINGS,
        MODEL = 'text-embedding-3-small',
        CREDENTIAL = [https://<your-openai-endpoint>.openai.azure.com]
    );
    GO
    ```

    > &#128221; `<your-openai-endpoint>` は、資格情報に使用したのと同じエンドポイント名に置き換えてください。 `api-version` の値 `2024-10-21` は、Azure OpenAI REST API の現在の GA バージョンです。 `MODEL` オプションは必須であり、モデル名に一致している必要があります。

---

## ベクトル列を追加して埋め込みを生成する

このセクションでは、ProductReview テーブルにベクトル列を追加し、レビュー テキストを表す埋め込みを生成し、効率的な類似性検索のためのベクトル インデックスを作成します。

> &#128221; `dbo.ProductReview` テーブルに既に `ReviewVector` 列があり、埋め込みを生成済みの場合は、このセクションをスキップしてください。

1. 最初に、レビューの埋め込みを格納するためのベクトル列を `dbo.ProductReview` テーブルに追加します。

    ```sql
    -- Add a vector column to store review text embeddings
    ALTER TABLE dbo.ProductReview
    ADD ReviewVector VECTOR(1536);
    GO
    ```

    > &#128221; `VECTOR(1536)` データ型には 1536 次元のベクトルが格納されますが、これは *text-embedding-3-small* モデルの出力に合わせたものです。

1. 各レビューの埋め込みを生成するために、製品名、レビュー タイトル、レビュー テキストを結合します。 このスクリプトではレビューを 30 件ずつバッチで処理し、API レート制限を避けるためにバッチ間に短い遅延を設けます。

    ```sql
    -- Generate embeddings in batches to avoid API rate limits
    DECLARE @batchSize INT = 30;
    DECLARE @rowsUpdated INT = 1;
    DECLARE @retryCount INT;
    DECLARE @maxRetries INT = 3;

    WHILE @rowsUpdated > 0
    BEGIN
        SET @retryCount = 0;

        RETRY:
        BEGIN TRY
            UPDATE TOP (@batchSize) r
            SET r.ReviewVector = AI_GENERATE_EMBEDDINGS(
                p.Name + ' - ' + r.ReviewTitle + ': ' + r.ReviewText
                USE MODEL my_embedding_model)
            FROM dbo.ProductReview r
            INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
            WHERE r.ReviewVector IS NULL;

            SET @rowsUpdated = @@ROWCOUNT;

            -- Brief pause between batches to respect API rate limits
            IF @rowsUpdated > 0
                WAITFOR DELAY '00:00:02';
        END TRY
        BEGIN CATCH
            SET @retryCount += 1;
            IF @retryCount <= @maxRetries
            BEGIN
                PRINT 'Rate limited. Retrying in 5 seconds... (Attempt ' 
                    + CAST(@retryCount AS NVARCHAR(10)) + ' of ' 
                    + CAST(@maxRetries AS NVARCHAR(10)) + ')';
                WAITFOR DELAY '00:00:05';
                GOTO RETRY;
            END
            ELSE
                THROW;
        END CATCH
    END
    GO
    ```

    > &#128221; このステップには数分かかることがあります。 このスクリプトは 1 バッチあたり 30 件のレビューを処理し、バッチ間には 2 秒の一時停止があります。 API からレート制限エラーが返された場合は、5 秒待って最大 3 回再試行します。 製品名、レビュー タイトル、レビュー テキストが一緒に埋め込まれているため、ベクトル検索時に、レビュー対象の製品と顧客の体験の両方について条件に一致するものを見つけることができます。

1. 埋め込みが生成されたことを確認します。

    ```sql
    -- Check how many reviews have embeddings
    SELECT 
        COUNT(*) AS TotalReviews,
        COUNT(ReviewVector) AS ReviewsWithEmbeddings
    FROM dbo.ProductReview;
    GO
    ```

1. プレビュー機能を有効にしてから、近似最近傍 (ANN) 検索を効率化するために列にベクトル インデックスを作成します。

    ```sql
    -- Enable preview features required for vector indexes
    ALTER DATABASE SCOPED CONFIGURATION SET PREVIEW_FEATURES = ON;
    GO

    -- Allow the table to remain writable after vector index creation
    ALTER DATABASE SCOPED CONFIGURATION SET ALLOW_STALE_VECTOR_INDEX = ON;
    GO

    -- Create a DiskANN vector index for fast approximate nearest neighbor search
    CREATE VECTOR INDEX IX_Review_ReviewVector
    ON dbo.ProductReview(ReviewVector)
    WITH (METRIC = 'cosine', TYPE = 'DISKANN');
    GO
    ```

    > &#128221; `CREATE VECTOR INDEX` で DiskANN ベースの近似最近傍 (ANN) インデックスが作成されますが、これは通常の非クラスター化インデックスとは根本的に異なります。 DiskANN はグラフ構造を構築するものであり、これでベクトルをたどって行くことによって似ているものを効率的に見つけます。 `ALLOW_STALE_VECTOR_INDEX` という設定で、テーブルが書き込み可能である状態を維持します。 これがない場合は、ベクトル インデックスが存在するときにテーブルは読み取り専用になります。

---

## フルテキスト インデックスを作成する

フルテキスト検索を行うには、検索対象のテキスト列にフルテキスト インデックスが存在する必要があります。 このセクションでは、フルテキスト カタログとインデックスを `ProductReview` テーブルの `ReviewTitle` 列と `ReviewText` 列に作成します。

1. フルテキスト カタログとフルテキスト インデックスを作成します。

    ```sql
    -- Create a full-text catalog
    CREATE FULLTEXT CATALOG ProductReviewCatalog AS DEFAULT;
    GO

    -- Create a full-text index on ReviewTitle and ReviewText
    CREATE FULLTEXT INDEX ON dbo.ProductReview
    (
        ReviewTitle LANGUAGE 1033,
        ReviewText LANGUAGE 1033
    )
    KEY INDEX PK_ProductReview
    ON ProductReviewCatalog
    WITH (CHANGE_TRACKING AUTO);
    GO
    ```

    > &#128221; `KEY INDEX` はテーブルの主キー上の一意なインデックスを参照する必要があります。 `LANGUAGE 1033` は英語を指定するものであり、これで語形変化を考慮したマッチングが可能になります (たとえば "ride" で "riding" が見つかります)。 `CHANGE_TRACKING AUTO` を指定すると、データの変化に合わせてインデックスが更新されます。

1. フルテキスト インデックスが作成されて内容が事前設定済みであることを確認します。

    ```sql
    -- Check full-text index status
    SELECT 
        OBJECTPROPERTY(OBJECT_ID('dbo.ProductReview'), 'TableFullTextPopulateStatus') AS PopulateStatus,
        OBJECTPROPERTY(OBJECT_ID('dbo.ProductReview'), 'TableHasActiveFulltextIndex') AS HasActiveIndex;
    GO
    ```

    > &#128221; `PopulateStatus` が **0** のときは、フルテキスト インデックスの内容が完全に事前設定済みでクエリに使用できる状態です。 値が **1** のときは、事前設定がまだ進行中です。 `HasActiveIndex` は **1** であることが必要です。

---

## フルテキスト述語を使用して検索する

フルテキスト検索では、`CONTAINS` や `FREETEXT` などの述語を使用してフルテキスト インデックスのクエリを実行します。 `CONTAINS` では、単語やフレーズが完全一致するものを探します。 `FREETEXT` では、語形変化したものも自動的に見つけます。

1. `CONTAINS` を使用して、ある単語に言及しているレビューを検索します。

    ```sql
    -- Find reviews that contain the word "puncture"
    SELECT 
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        p.Name AS ProductName
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    WHERE CONTAINS(r.ReviewText, 'puncture');
    GO
    ```

    > &#128221; `CONTAINS` は、フルテキスト インデックスを検索して "puncture" という単語がこのままの形で含まれているものを見つけることです。 レビューのうち、その特定の単語が出現するものだけが返されます。

1. `FREETEXT` を使用して、あるフレーズを検索します。 `FREETEXT` を指定すると自動的に検索語が拡張して、語形変化したものも対象になります。

    ```sql
    -- Find reviews about gloves and warmth
    SELECT TOP 10
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        p.Name AS ProductName
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    WHERE FREETEXT(r.ReviewText, 'warm gloves for cold winter commuting');
    GO
    ```

    > &#128221; `FREETEXT` は語形変化を扱うものであるため、"warm" を指定すると "warmth" や "warming" も見つかり、"commuting" を指定すると "commute" や "commuter" も見つかります。 また、ストップワード (たとえば "for" や "the") が除外されます。 これとは対照的に `CONTAINS` では、語形のそれぞれを明示的に指定する必要があります。 `TOP` を `FREETEXT` とともに使用しているのは、広義のフレーズを指定すると一致する行が多くなるからです。

    `FREETEXT` の方が柔軟ですが、検索フレーズが広義すぎる場合に、関連性の低い結果が返されることがあります。 `CONTAINS` の方がコントロールしやすくなりますが、より正確にクエリを書く必要があります。

1. `CONTAINSTABLE` を使用して、結果を関連性スコア付きでランク付けして取得します。

    ```sql
    -- Search with ranking scores
    SELECT TOP 10
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        p.Name AS ProductName,
        ft.[RANK] AS FullTextRank
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    INNER JOIN CONTAINSTABLE(dbo.ProductReview, (ReviewTitle, ReviewText), 
        'FORMSOF(INFLECTIONAL, mountain) AND FORMSOF(INFLECTIONAL, trail)') AS ft
        ON r.ReviewID = ft.[KEY]
    ORDER BY ft.[RANK] DESC;
    GO
    ```

    > &#128221; `CONTAINSTABLE` を指定すると、`KEY` 列 (主キーと一致) と `RANK` 列 (各行がどれだけ一致しているかを示す) を持つテーブルが返されます。 `FORMSOF(INFLECTIONAL, mountain)` と指定すると "mountain"、"mountains"、およびその他の語形変化したものが見つかります。 `AND` 演算子を指定すると、両方の単語が出現することが必須になります。

1. プレフィックス検索を使用して、特定の文字列で始まる単語が含まれるレビューを見つけます。

    ```sql
    -- Find reviews with words starting with "comfort"
    SELECT 
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        p.Name AS ProductName
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    WHERE CONTAINS(r.ReviewText, '"comfort*"');
    GO
    ```

    > &#128221; プレフィックス検索 `"comfort*"` を指定すると、"comfort"、"comfortable"、"comfortably"、およびその他の "comfort" で始まる単語が見つかります。 これは、ある語根のバリエーションを、個々の語形を列挙することなく捉えたい場合に便利です。

---

## ベクトル類似性を使用して検索する

ベクトル検索は、レビューを見つけるときに単なるキーワード一致ではなくテキストのセマンティック的な意味を基準とするものです。 たとえば、「ライド中の飲み物を冷たいままで」と質問すると、このとおりの語句が出現しないものも含めて、水筒に関するレビューを見つけることができます。

1. `VECTOR_DISTANCE` を使用して厳密な最近傍検索を実行します。 このオプションを指定すると、クエリ埋め込みと各レビューとのコサイン距離が計算されます。

    ```sql
    -- Exact vector search using VECTOR_DISTANCE
    DECLARE @searchText NVARCHAR(1000) = 'comfortable seat for long distance touring';
    DECLARE @searchVector VECTOR(1536);

    SELECT @searchVector = AI_GENERATE_EMBEDDINGS(@searchText USE MODEL my_embedding_model);

    SELECT TOP 5
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        VECTOR_DISTANCE('cosine', @searchVector, r.ReviewVector) AS Distance
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    ORDER BY Distance;
    GO
    ```

    > &#128221; `VECTOR_DISTANCE` を指定すると、2 つのベクトル間のコサイン距離が計算されます。 値が小さいほど、類似性が高くなります。 この関数はテーブルのすべての行をスキャンするものであり、小さいデータセットに適しています。 結果に含まれている、ツーリング自転車とサドルの快適さに関するレビューの中には、"comfortable seat" という語句が正確に含まれてはいないものもあることに注目してください。

1. `VECTOR_SEARCH` を DiskANN インデックスとともに使用して近似最近傍 (ANN) 検索を実行します。 このインデックスは大規模なデータセット向けに最適化されており、速く結果が得られます。

    ```sql
    -- Approximate vector search using VECTOR_SEARCH with DiskANN index
    DECLARE @searchText NVARCHAR(1000) = 'something to keep me visible when riding at night';
    DECLARE @searchVector VECTOR(1536);

    SELECT @searchVector = AI_GENERATE_EMBEDDINGS(@searchText USE MODEL my_embedding_model);

    SELECT
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        pc.Name AS Category,
        vs.distance AS Distance
    FROM VECTOR_SEARCH(
        TABLE = dbo.ProductReview AS r,
        COLUMN = ReviewVector,
        SIMILAR_TO = @searchVector,
        METRIC = 'cosine',
        TOP_N = 5
    ) AS vs
    INNER JOIN SalesLT.Product p 
        ON r.ProductID = p.ProductID
    INNER JOIN SalesLT.ProductCategory pc 
        ON p.ProductCategoryID = pc.ProductCategoryID
    ORDER BY vs.distance;
    GO
    ```

    > &#128221; `VECTOR_SEARCH` は DiskANN インデックスを使って近似最近傍を見つけるため、すべての行をスキャンすることはありません。 このクエリで述べているのは、夜間ライド中の視認性という概念 ("visible when riding at night") であり、具体的なキーワードではありません。 ベクトル検索では、レビューの中からライト、反射器材、視認性に関するものを、これらの語句が検索テキストに出現していなくても見つけることができます。

1. `VECTORPROPERTY` を使用してベクトル メタデータを調べます。

    ```sql
    -- Inspect vector metadata
    SELECT TOP 1
        VECTORPROPERTY(ReviewVector, 'Dimensions') AS VectorDimensions,
        VECTORPROPERTY(ReviewVector, 'BaseType') AS VectorBaseType
    FROM dbo.ProductReview
    WHERE ReviewVector IS NOT NULL;
    GO
    ```

    > &#128221; `VECTORPROPERTY` を指定すると 1 つのベクトル列に関するメタデータが返されます。 これは、ベクトルの次元数が期待したとおりであることの検証とデータ型の確認に便利です。

---

## フルテキスト検索とベクトル検索をハイブリッド検索として結合する

フルテキスト検索は、キーワードの完全一致を見つけるのに適していますが、同じ考えを別の形で表現する文書を見逃してしまいます。 ベクトル検索では、セマンティック的な意味が捉えられますが、ユーザーが指定した重要な用語を見逃すことがあります。 ハイブリッド検索とは、両方のアプローチを実行してその結果を Reciprocal Rank Fusion (RRF) を使用してマージすることです。

RRF でさまざまなソースからのランク付き結果を結合するときは、生のスコアではなくランク位置が使用されます。 式 `1/(k + rank)` でランクがスコアに変換され、`k` は平滑化定数 (一般的に 60) です。 両方の結果セットに現れる項目は総合スコアが高くなり、したがって幅広く高い関連性を持つ結果が上位に押し上げられます。

1. 次に示すハイブリッド検索クエリを実行します。これはフルテキスト検索とベクトル検索を RRF を使用して結合するものです。

    ```sql
    DECLARE @searchText NVARCHAR(1000) = 'durable tires that resist punctures on rough terrain';
    DECLARE @searchVector VECTOR(1536);
    DECLARE @topN INT = 50;
    DECLARE @rrfK INT = 60;

    -- Generate embedding for the search phrase
    SELECT @searchVector = AI_GENERATE_EMBEDDINGS(@searchText USE MODEL my_embedding_model);

    -- Run hybrid search with RRF
    WITH keyword_search AS (
        SELECT TOP(@topN)
            r.ReviewID,
            RANK() OVER (ORDER BY ft.[RANK] DESC) AS keyword_rank
        FROM dbo.ProductReview r
        INNER JOIN FREETEXTTABLE(dbo.ProductReview, (ReviewTitle, ReviewText), @searchText) AS ft
            ON r.ReviewID = ft.[KEY]
    ),
    vector_search AS (
        SELECT TOP(@topN)
            ReviewID,
            RANK() OVER (ORDER BY distance) AS vector_rank
        FROM (
            SELECT 
                r.ReviewID,
                vs.distance
            FROM VECTOR_SEARCH(
                TABLE = dbo.ProductReview AS r,
                COLUMN = ReviewVector,
                SIMILAR_TO = @searchVector,
                METRIC = 'cosine',
                TOP_N = 50
            ) AS vs
        ) AS similar_reviews
    ),
    combined AS (
        SELECT
            COALESCE(ks.ReviewID, vs.ReviewID) AS ReviewID,
            ks.keyword_rank,
            vs.vector_rank,
            COALESCE(1.0 / (@rrfK + ks.keyword_rank), 0.0) +
            COALESCE(1.0 / (@rrfK + vs.vector_rank), 0.0) AS rrf_score
        FROM keyword_search ks
        FULL OUTER JOIN vector_search vs ON ks.ReviewID = vs.ReviewID
    )
    SELECT TOP 10
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        c.keyword_rank,
        c.vector_rank,
        c.rrf_score
    FROM combined c
    INNER JOIN dbo.ProductReview r ON c.ReviewID = r.ReviewID
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    ORDER BY c.rrf_score DESC;
    GO
    ```

    > &#128221; このクエリでは、CTE を使用してフルテキスト検索とベクトル検索を並列で実行します。 `keyword_search` の CTE では `FREETEXTTABLE` を使用して BM25 関連性に基づいてレビューをランク付けします。 `vector_search` の CTE では `VECTOR_SEARCH` を使用して埋め込み類似性でレビューをランク付けします。 `combined` の CTE で両方の結果セットを `FULL OUTER JOIN` で結合し、RRF スコアを計算します。 両方のリストに出現するレビューは、総合スコアが高くなります。 最後の `SELECT` で結果の上位 10 件を RRF スコア順に返します。これは、どのレビューが両方の検索方法で高い関連性を示したかを表します。

1. 出力の列を調べます。 `keyword_rank` と `vector_rank` の両方の値を持つ行は両方の検索方法で見つかったものであり、RRF スコアが最高に近くなる傾向があります。 ランク列の 1 つに `NULL` がある行は、1 つの方法でしか検出されませんでした。

---

## 3 つの検索アプローチを比較する

各アプローチの強みを理解するために、同じ質問を使って 3 つの検索方法すべてを実行し、結果を比較します。

1. フルテキスト検索を実行して快適なファミリー自転車 (comfortable family bike) を見つけます。

    ```sql
    -- Full-text search only
    SELECT TOP 5
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        ft.[RANK] AS FullTextRank
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    INNER JOIN FREETEXTTABLE(dbo.ProductReview, (ReviewTitle, ReviewText), 
        'comfortable bike for long weekend rides with the family') AS ft
        ON r.ReviewID = ft.[KEY]
    ORDER BY ft.[RANK] DESC;
    GO
    ```

1. 同じ質問でベクトル検索を実行します。

    ```sql
    -- Vector search only
    DECLARE @searchText NVARCHAR(1000) = 'comfortable bike for long weekend rides with the family';
    DECLARE @searchVector VECTOR(1536);

    SELECT @searchVector = AI_GENERATE_EMBEDDINGS(@searchText USE MODEL my_embedding_model);

    SELECT
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        vs.distance AS Distance
    FROM VECTOR_SEARCH(
        TABLE = dbo.ProductReview AS r,
        COLUMN = ReviewVector,
        SIMILAR_TO = @searchVector,
        METRIC = 'cosine',
        TOP_N = 5
    ) AS vs
    INNER JOIN SalesLT.Product p 
        ON r.ProductID = p.ProductID
    ORDER BY vs.distance;
    GO
    ```

1. 同じ質問でハイブリッド検索を実行します。

    ```sql
    -- Hybrid search with RRF
    DECLARE @searchText NVARCHAR(1000) = 'comfortable bike for long weekend rides with the family';
    DECLARE @searchVector VECTOR(1536);
    DECLARE @topN INT = 50;
    DECLARE @rrfK INT = 60;

    SELECT @searchVector = AI_GENERATE_EMBEDDINGS(@searchText USE MODEL my_embedding_model);

    WITH keyword_search AS (
        SELECT TOP(@topN)
            r.ReviewID,
            RANK() OVER (ORDER BY ft.[RANK] DESC) AS keyword_rank
        FROM dbo.ProductReview r
        INNER JOIN FREETEXTTABLE(dbo.ProductReview, (ReviewTitle, ReviewText), @searchText) AS ft
            ON r.ReviewID = ft.[KEY]
    ),
    vector_search AS (
        SELECT TOP(@topN)
            ReviewID,
            RANK() OVER (ORDER BY distance) AS vector_rank
        FROM (
            SELECT 
                r.ReviewID,
                vs.distance
            FROM VECTOR_SEARCH(
                TABLE = dbo.ProductReview AS r,
                COLUMN = ReviewVector,
                SIMILAR_TO = @searchVector,
                METRIC = 'cosine',
                TOP_N = 50
            ) AS vs
        ) AS similar_reviews
    ),
    combined AS (
        SELECT
            COALESCE(ks.ReviewID, vs.ReviewID) AS ReviewID,
            ks.keyword_rank,
            vs.vector_rank,
            COALESCE(1.0 / (@rrfK + ks.keyword_rank), 0.0) +
            COALESCE(1.0 / (@rrfK + vs.vector_rank), 0.0) AS rrf_score
        FROM keyword_search ks
        FULL OUTER JOIN vector_search vs ON ks.ReviewID = vs.ReviewID
    )
    SELECT TOP 5
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating,
        c.keyword_rank,
        c.vector_rank,
        c.rrf_score
    FROM combined c
    INNER JOIN dbo.ProductReview r ON c.ReviewID = r.ReviewID
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    ORDER BY c.rrf_score DESC;
    GO
    ```

    > &#128221; 3 つの結果セットを並べて比較してみてください。 フルテキスト検索では、"comfortable"、"weekend"、"family" などの単語が含まれるレビューが返されます。 ベクトル検索では、レクリエーション用自転車、リラックスできるジオメトリ、レジャー ライドに関するレビューが、表現が異なるものも含めて返されます。 ハイブリッド検索は両者を結合するものであり、両方の結果セットに出現するレビューのスコアが高くなります。 この比較は、各アプローチがどのような場合に最適かを示しています。つまり、キーワード完全一致を見つけるにはフルテキスト、セマンティック的類似性で見つけるにはベクトル、両方を求める場合はハイブリッドとなります。

---

## クリーンアップ

Azure SQL Database や Azure OpenAI のリソースを他の目的に使用する予定がない場合は、この演習で作成したリソースをクリーンアップしてもかまいません。

> &#128221; これらのリソースはラボ 9、10、11 で使用されます。

新しいリソース グループをこのラボ用にプロビジョニングした場合は、そのリソース グループ全体を削除すると、すべてのリソースを一度に削除できます。 既存のリソース グループを使用していた場合は、Azure SQL Database と Azure OpenAI のリソースを個別に削除してください。

1. Azure Portalで、リソース グループに移動します。
1. **[リソース グループの削除]** を選び、削除を確定するためにリソース グループの名前を入力します。
1. **[削除]** を選択して、このラボで作成されたすべてのリソースを削除します。

---

これでこの演習は完了です。

この演習では、Azure SQL Database で 3 つの検索アプローチを実装して比較しました。具体的にはフルテキスト検索 (フルテキスト インデックスをキーワード述語と語形変化パターンとともに使用する)、ベクトル検索 (厳密および近似最近傍クエリを DiskANN インデックスとともに使用する)、およびハイブリッド検索 (両手法を結合し、Reciprocal Rank Fusion を使用してキーワードとセマンティックの結果をマージする) です。 3 つのアプローチすべてを同じクエリで実行して比較し、それぞれの強みを理解しました。