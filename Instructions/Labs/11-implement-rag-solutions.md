---
lab:
  title: ラボ 11 - RAG ソリューションを実装する
  module: Design and implement RAG with SQL
  description: この演習では、Azure SQL Database と Azure OpenAI を使用して完全な検索拡張生成 (RAG) ソリューションを実装する方法を学習します。
  duration: 45
  level: 300
  islab: true
  status: released
  targetDate: '2099-01-01'
---

# RAG ソリューションを実装する

**予測される所要時間**: 45 分

この演習では、Azure SQL Database を使用して完全な検索拡張生成 (RAG) ソリューションを実装します。 製品レビュー テーブルを作成し、埋め込みを格納するためのベクトル列を追加し、顧客レビューを表す埋め込みを生成し、ベクトル検索を使用して関連性の高いレビューを取得し、JSON コンテキストとして書式設定し、拡張プロンプトを組み立て、Azure OpenAI エンドポイントを呼び出し、応答を抽出します。

あなたは Adventure Works のデータベース開発者であるとします。 あなたのチームは、顧客からの質問に実際の顧客レビューを使って回答できる AI 搭載の製品アシスタントを構築したいと考えています。 モデルを微調整する代わりに、RAG を使用して、あなたのデータベースをモデルの応答の典拠にさせます。 ベクトル検索を使用して、各質問への関連性が最も高いレビューを見つけます。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 前提条件

- [Azure サブスクリプション](https://azure.microsoft.com/free) ([Azure OpenAI へのアクセス](/legal/cognitive-services/openai/limited-access)が承認されていること)
- Visual Studio Code と SQL Server (mssql) 拡張機能、または SQL Server Management Studio
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
    >
    > ここで選ぶ認証方法は、あなたが開発者としてどのようにデータベースに接続するかを決定するものです。** Azure SQL が Azure OpenAI に接続する方法とは別のものであり、これについてはこのラボで後ほど構成します。

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

AdventureWorksLT サンプル データベースには製品情報が格納されていますが、顧客レビューはありません。 このステップでは、あるスクリプトをダウンロードして実行することによって **ProductReview** テーブルを作成し、多数の製品カテゴリにわたる 140 件のリアルなレビューをここに格納します。 これらのレビューは、RAG ソリューションでの検索に適した、豊富で多様なコンテンツとなります。

1. 自分の Azure SQL Database に Visual Studio Code (SQL Server 拡張機能付き) または SQL Server Management Studio を使用して接続します。

    > &#128161; **[接続方法]** は、組織がサポートし、サーバーの作成時に構成された認証方法によって異なります。
    > - **Microsoft Entra 認証**: SSMS では、[認証] を **[Microsoft Entra MFA]** に設定して Azure アカウントでサインインします。** VS Code では、接続プロファイルを作成するときに認証の種類として **[Microsoft Entra ID]** を選択します。
    > - **SQL 認証**: SSMS または VS Code で、**[サーバー管理者ログイン]** と **[パスワード]** にはサーバー作成時に指定したものを入力し、[認証] を **[SQL ログイン]** に設定します。**
    >
    > どちらの場合も、**[サーバー名]** を `<your-server-name>.database.windows.net` に、**[データベース]** を **AdventureWorksLT** に設定します。
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

    > &#128221; `CREATE VECTOR INDEX` で DiskANN ベースの近似最近傍 (ANN) インデックスが作成されますが、これは通常の非クラスター化インデックスとは根本的に異なります。 DiskANN はグラフ構造を構築するものであり、これでベクトルをたどって行くことによって似ているものを効率的に見つけます。 `ALLOW_STALE_VECTOR_INDEX` という設定で、テーブルが書き込み可能である状態を維持します。 これがない場合は、ベクトル インデックスが存在するときにテーブルは読み取り専用になります。 テーブルが大きい場合は、ANN インデックスを作成しておくと類似性検索の速度が大幅に向上しますが、これはすべての行をフル スキャンすることがなくなるからです。

---

## ベクトル検索を使用してデータを取得し、JSON コンテキストとして書式設定する

このセクションでは、RAG の**検索** (retrieval) ステップを実践します。 ハードコーディングされた `WHERE` 句を使う代わりに、ユーザーの質問を埋め込みに変換し、`VECTOR_SEARCH` を使用して最も関連性の高いレビューを見つけます。 その結果を JSON として書式設定します。

1. 次のクエリを実行し、`VECTOR_SEARCH` 関数を使用して近似最近傍探索を行います。 この関数では DiskANN ベクトル インデックスが使用されるため、取得が速くなります。

    ```sql
    -- Convert a question to an embedding and find the closest matching reviews
    DECLARE @userQuestion NVARCHAR(1000) = 'What mountain bike can handle really technical rocky trails?';
    DECLARE @questionVector VECTOR(1536);

    -- Generate embedding for the question
    SELECT @questionVector = AI_GENERATE_EMBEDDINGS(@userQuestion USE MODEL my_embedding_model);

    -- Find the top 5 most relevant reviews using ANN vector search
    SELECT
        p.Name AS ProductName,
        p.ListPrice,
        pc.Name AS Category,
        r.Rating,
        r.ReviewTitle,
        r.ReviewText,
        vs.distance AS Distance
    FROM VECTOR_SEARCH(
        TABLE = dbo.ProductReview AS r,
        COLUMN = ReviewVector,
        SIMILAR_TO = @questionVector,
        METRIC = 'cosine',
        TOP_N = 5
    ) AS vs
    INNER JOIN SalesLT.Product p 
        ON r.ProductID = p.ProductID
    INNER JOIN SalesLT.ProductCategory pc 
        ON p.ProductCategoryID = pc.ProductCategoryID
    FOR JSON PATH;
    GO
    ```

    > &#128221; `VECTOR_SEARCH` では DiskANN ベクトル インデックスを使用して近似最近傍 (ANN) 検索が実行されます。 `VECTOR_DISTANCE` と `ORDER BY` (すべての行をスキャンする) とは異なり、`VECTOR_SEARCH` ではグラフ インデックスをたどることによって最も一致度が高いものを効率的に見つけます。 `distance` 列は自動的に結果に含まれます。 距離値が小さいほど、関連性が高くなります。 各レビューの独自性に注目してください。 製品説明では、サイズが異なっていても同じ自転車ならば同じ文章が使われますが、レビューはそれぞれ異なる顧客の体験を表しています。

---

## 拡張プロンプトをデータベース コンテキスト付きで構築する

ここで、取得したデータにシステム メッセージとユーザー質問を組み合わせて**拡張された** (augmented) プロンプトを作成します。 このプロンプトが RAG の "A" です。

1. 次のスクリプトを実行して完全な RAG プロンプトを T-SQL で作成します。 このスクリプトでは、ベクトル検索を使用して関連性の高いレビューを取得してから、拡張プロンプトを構築します。

    ```sql
    DECLARE @userQuestion NVARCHAR(1000) = 'What is the best bike for commuting to work in rainy weather?';
    DECLARE @questionVector VECTOR(1536);
    DECLARE @context NVARCHAR(MAX);
    DECLARE @payload NVARCHAR(MAX);

    -- Step 1: Convert the question to an embedding
    SELECT @questionVector = AI_GENERATE_EMBEDDINGS(@userQuestion USE MODEL my_embedding_model);

    -- Step 2: Retrieve relevant reviews using ANN vector search
    SET @context = (
        SELECT
            p.Name AS ProductName,
            p.ListPrice,
            pc.Name AS Category,
            r.Rating,
            r.ReviewTitle,
            r.ReviewText
        FROM VECTOR_SEARCH(
            TABLE = dbo.ProductReview AS r,
            COLUMN = ReviewVector,
            SIMILAR_TO = @questionVector,
            METRIC = 'cosine',
            TOP_N = 5
        ) AS vs
        INNER JOIN SalesLT.Product p 
            ON r.ProductID = p.ProductID
        INNER JOIN SalesLT.ProductCategory pc 
            ON p.ProductCategoryID = pc.ProductCategoryID
        FOR JSON PATH
    );

    -- Step 3: Build augmented prompt using JSON_OBJECT and JSON_ARRAY
    SET @payload = JSON_OBJECT(
        'messages': JSON_ARRAY(
            JSON_OBJECT(
                'role': 'system', 
                'content': 'You are an Adventure Works product assistant. Answer questions using only the provided product reviews and data. Mention specific customer experiences from the reviews when relevant. Be concise and helpful. If the data does not contain enough information, say so.'
            ),
            JSON_OBJECT(
                'role': 'user', 
                'content': 'Product reviews: ' + ISNULL(@context, '[]') + CHAR(10) + CHAR(10) + 'Customer question: ' + @userQuestion
            )
        ),
        'max_tokens': CAST(500 AS INT),
        'temperature': 0.5
    );

    -- Display the constructed payload
    SELECT @payload AS AugmentedPrompt;
    GO
    ```

    > &#128221; 出力の JSON を精査してください。 `VECTOR_SEARCH` 関数は、最もセマンティック的に関連性の高い顧客レビューを DiskANN ベクトル インデックスを使用して見つけます。 レビューは一つ一つ異なり、その中には実際の顧客の体験、評価、意見が含まれているため、製品説明文だけの場合に比べて豊かなコンテキストをモデルに与えることができます。 システム メッセージはモデルに対して、具体的な顧客体験を回答の中で参照するよう指示するものです。

---

## Azure OpenAI エンドポイントを呼び出して応答を生成する

このステップは RAG の G、つまり生成 (generation) のステップです。 拡張プロンプトを Azure OpenAI に送信し、回答を抽出します。

1. 次のスクリプトを実行して RAG パイプライン全体を完成させます。 `<your-openai-endpoint>` を実際のエンドポイント名で置き換えます。

    ```sql
    DECLARE @userQuestion NVARCHAR(1000) = 'Which tires last the longest and resist punctures?';
    DECLARE @questionVector VECTOR(1536);
    DECLARE @context NVARCHAR(MAX);
    DECLARE @payload NVARCHAR(MAX);
    DECLARE @response NVARCHAR(MAX);
    DECLARE @returnValue INT;

    -- Step 1: Convert the question to an embedding
    SELECT @questionVector = AI_GENERATE_EMBEDDINGS(@userQuestion USE MODEL my_embedding_model);

    -- Step 2: Retrieve relevant reviews using ANN vector search
    SET @context = (
        SELECT
            p.Name AS ProductName,
            p.ListPrice,
            pc.Name AS Category,
            r.Rating,
            r.ReviewTitle,
            r.ReviewText
        FROM VECTOR_SEARCH(
            TABLE = dbo.ProductReview AS r,
            COLUMN = ReviewVector,
            SIMILAR_TO = @questionVector,
            METRIC = 'cosine',
            TOP_N = 5
        ) AS vs
        INNER JOIN SalesLT.Product p 
            ON r.ProductID = p.ProductID
        INNER JOIN SalesLT.ProductCategory pc 
            ON p.ProductCategoryID = pc.ProductCategoryID
        FOR JSON PATH
    );

    -- Step 3: Build augmented prompt
    SET @payload = JSON_OBJECT(
        'messages': JSON_ARRAY(
            JSON_OBJECT(
                'role': 'system', 
                'content': 'You are an Adventure Works product assistant. Answer questions using only the provided product reviews and data. Mention specific customer experiences from the reviews when relevant. Be concise and helpful. If the data does not contain enough information, say so.'
            ),
            JSON_OBJECT(
                'role': 'user', 
                'content': 'Product reviews: ' + ISNULL(@context, '[]') + CHAR(10) + CHAR(10) + 'Customer question: ' + @userQuestion
            )
        ),
        'max_tokens': CAST(500 AS INT),
        'temperature': 0.5
    );

    -- Step 4: Call Azure OpenAI
    EXECUTE @returnValue = sp_invoke_external_rest_endpoint
        @url = N'https://<your-openai-endpoint>.openai.azure.com/openai/deployments/gpt-5.4-mini/chat/completions?api-version=2024-10-21',
        @method = 'POST',
        @payload = @payload,
        @credential = [https://<your-openai-endpoint>.openai.azure.com],
        @response = @response OUTPUT;

    -- Step 5: Extract and display the answer
    IF @returnValue = 0
    BEGIN
        DECLARE @answer NVARCHAR(MAX);
        SET @answer = JSON_VALUE(@response, '$.result.choices[0].message.content');
        SELECT @answer AS AssistantResponse;
    END
    ELSE
    BEGIN
        SELECT 
            @returnValue AS HttpStatus,
            JSON_VALUE(@response, '$.response.status.http.description') AS ErrorDescription;
    END
    GO
    ```

    > &#128221; `<your-openai-endpoint>` は、資格情報に使用したのと同じエンドポイント名に置き換えてください。 `api-version` の値 `2024-10-21` は、Azure OpenAI REST API の現在の GA バージョンです。

---

## RAG ストアド プロシージャを作成する

ここで、すべてを一つにまとめて、自分のアプリケーションで呼び出せるように再利用可能なストアド プロシージャを作成します。

1. 次のスクリプトを実行してストアド プロシージャを作成します。 `<your-openai-endpoint>` を実際のエンドポイント名で置き換えます。

    ```sql
    CREATE OR ALTER PROCEDURE dbo.AskProductQuestion
        @Question NVARCHAR(1000),
        @Answer NVARCHAR(MAX) OUTPUT
    AS
    BEGIN
        SET NOCOUNT ON;

        DECLARE @questionVector VECTOR(1536);
        DECLARE @context NVARCHAR(MAX);
        DECLARE @payload NVARCHAR(MAX);
        DECLARE @response NVARCHAR(MAX);
        DECLARE @returnValue INT;

        -- Step 1: Convert the question to an embedding
        SELECT @questionVector = AI_GENERATE_EMBEDDINGS(@Question USE MODEL my_embedding_model);

        -- Step 2: Retrieve relevant reviews using ANN vector search
        SET @context = (
            SELECT
                p.Name AS ProductName,
                p.ListPrice,
                pc.Name AS Category,
                r.Rating,
                r.ReviewTitle,
                r.ReviewText
            FROM VECTOR_SEARCH(
                TABLE = dbo.ProductReview AS r,
                COLUMN = ReviewVector,
                SIMILAR_TO = @questionVector,
                METRIC = 'cosine',
                TOP_N = 5
            ) AS vs
            INNER JOIN SalesLT.Product p 
                ON r.ProductID = p.ProductID
            INNER JOIN SalesLT.ProductCategory pc 
                ON p.ProductCategoryID = pc.ProductCategoryID
            FOR JSON PATH
        );

        -- Check if context was retrieved
        IF @context IS NULL
        BEGIN
            SET @Answer = 'No reviews found matching your query. Please try a different question.';
            RETURN;
        END

        -- Step 3: Build the augmented prompt
        SET @payload = JSON_OBJECT(
            'messages': JSON_ARRAY(
                JSON_OBJECT(
                    'role': 'system', 
                    'content': 'You are an Adventure Works product assistant. Follow these rules:
    1. Answer only using the provided product reviews and data
    2. Reference specific customer experiences from the reviews when relevant
    3. Include star ratings to help the customer assess product quality
    4. Keep responses under 150 words
    5. Suggest related products when relevant'
                ),
                JSON_OBJECT(
                    'role': 'user', 
                    'content': 'Product reviews: ' + @context + CHAR(10) + CHAR(10) + 'Customer question: ' + @Question
                )
            ),
            'max_tokens': CAST(500 AS INT),
            'temperature': 0.5
        );

        -- Step 4: Call the model
        EXECUTE @returnValue = sp_invoke_external_rest_endpoint
            @url = N'https://<your-openai-endpoint>.openai.azure.com/openai/deployments/gpt-5.4-mini/chat/completions?api-version=2024-10-21',
            @method = 'POST',
            @payload = @payload,
            @credential = [https://<your-openai-endpoint>.openai.azure.com],
            @response = @response OUTPUT;

        -- Step 5: Extract the answer or handle errors
        IF @returnValue = 0
            SET @Answer = JSON_VALUE(@response, '$.result.choices[0].message.content');
        ELSE IF @returnValue = 429
            SET @Answer = 'The service is currently busy. Please try again in a moment.';
        ELSE IF @returnValue IN (401, 403)
            SET @Answer = 'Authentication failed. Please check the credential configuration.';
        ELSE
            SET @Answer = 'Unable to process your question at this time. HTTP status: ' + CAST(@returnValue AS NVARCHAR(10));
    END;
    GO
    ```

    > &#128221; `<your-openai-endpoint>` は、資格情報に使用したのと同じエンドポイント名に置き換えてください。 `api-version` の値 `2024-10-21` は、Azure OpenAI REST API の現在の GA バージョンです。

1. ストアド プロシージャをテストするために、夜間ライドの安全性について質問します。

    ```sql
    DECLARE @response NVARCHAR(MAX);

    EXEC dbo.AskProductQuestion
        @Question = 'I ride before sunrise and after dark. What lights actually work well?',
        @Answer = @response OUTPUT;

    SELECT @response AS AssistantResponse;
    GO
    ```

1. 自転車のメンテナンスに関する質問を試してみます。

    ```sql
    DECLARE @response NVARCHAR(MAX);

    EXEC dbo.AskProductQuestion
        @Question = 'How do I keep my bike maintained between shop visits?',
        @Answer = @response OUTPUT;

    SELECT @response AS AssistantResponse;
    GO
    ```

1. よくある不満についての質問でテストしてみます。

    ```sql
    DECLARE @response NVARCHAR(MAX);

    EXEC dbo.AskProductQuestion
        @Question = 'What are the most common problems or complaints people have with their bikes?',
        @Answer = @response OUTPUT;

    SELECT @response AS AssistantResponse;
    GO
    ```

    > &#128221; 各呼び出しは同じ RAG パターンに従っています。つまり、質問を埋め込みに変換し、`VECTOR_SEARCH` を DiskANN インデックスとともに使用して最も関連性の高いレビューをデータベースから取得し、そのデータでプロンプトを拡張し、典拠のある応答を生成します。 モデルの回答は、レビューに書かれている具体的な顧客体験を参照しているはずです。 ベクトル検索では関連性の高いレビューがセマンティック的意味に基づいて見つけられていることに注目してください。 たとえば、"問題" について質問すると、具体的な不具合に言及している低評価のレビューが取得されます。

独自の質問を考えて試してみてください。 テストに使用するクエリの種類が多いほど、RAG パイプラインの仕組みについて、および取得されたコンテキストをモデルがどのように使用して回答を生成しているかについて、よりよく理解できるようになります。

---

## クリーンアップ

Azure SQL Database や Azure OpenAI のリソースを他の目的に使用する予定がない場合は、この演習で作成したリソースをクリーンアップしてもかまいません。

> &#128221; これらのリソースはラボ 9、10、11 で使用されます。

新しいリソース グループをこのラボ用にプロビジョニングした場合は、そのリソース グループ全体を削除すると、すべてのリソースを一度に削除できます。 既存のリソース グループを使用していた場合は、Azure SQL Database と Azure OpenAI のリソースを個別に削除してください。

1. Azure Portalで、リソース グループに移動します。
1. **[リソース グループの削除]** を選択し、リソース グループの名前を入力して削除を確認します。
1. **[削除]** を選択して、このラボで作成されたすべてのリソースを削除します。

---

これでこの演習は完了です。

この演習では、Azure SQL Database と Azure OpenAI を使用して完全な検索拡張生成 (RAG) ソリューションを実装しました。 埋め込みを生成し、速く取得できるようにベクトル インデックスを作成し、セマンティック的に関連性の高いレビューを検索し、結果を言語モデル用のコンテキストとして書式設定し、拡張プロンプトを典拠に関する指示付きで作成し、Azure OpenAI エンドポイントを T-SQL から呼び出し、RAG パイプライン全体をパッケージ化してエラー処理付きの再利用可能なストアド プロシージャを作成しました。
