---
lab:
  title: ラボ 9 - Azure SQL Database での埋め込みの生成と更新
  module: Design and implement models and embeddings with SQL
  description: この演習では、外部モデル参照を作成し、Azure SQL Database に格納されているテキストから埋め込みを生成し、基本的なベクトル検索を実行する方法を学習します。
  duration: 30
  level: 300
  islab: true
  status: released
  targetDate: '2099-01-01'
---

# Azure SQL Database での埋め込みの生成と更新

**推定時間**:30 分

この演習では、外部モデル参照を作成し、Azure SQL Database に格納されているテキストから `AI_GENERATE_EMBEDDINGS` 関数を使って埋め込みを生成し、結果を確認します。 また、ソース データが変化したときに埋め込みをメンテナンスする必要があることについても見ていきます。 最後に、基本的なベクトル検索を実行して、埋め込みがセマンティック的意味を捉えていることを確認します。

あなたは Adventure Works のデータベース開発者であるとします。 あなたのチームは、AI 搭載の検索機能を製品カタログに追加しようとしています。 最初のステップは、顧客レビューを表すベクトル埋め込みを生成して保存することです。これで、後でセマンティック的類似性に基づいて比較できるようになります。 あなたは、レビュー データが変化したときに埋め込みの同期を維持する方法についても理解している必要があります。

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

## ProductReview テーブルを作成する

AdventureWorksLT サンプル データベースには製品情報が格納されていますが、顧客レビューはありません。 このステップでは、あるスクリプトをダウンロードして実行することによって **ProductReview** テーブルを作成し、多数の製品カテゴリにわたる 140 件のリアルなレビューをここに格納します。 この演習では、これらのレビューのテキスト データから埋め込みを生成します。

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

このセクションでは、ProductReview テーブルにベクトル列を追加し、レビュー テキストを表す埋め込みを生成します。 埋め込みモデルによって各レビューのテキストが、そのセマンティック的意味を捉えた 1536 次元ベクトルに変換されます。

1. レビュー埋め込みを格納するために、`dbo.ProductReview` テーブルにベクトル列を追加します。

    ```sql
    -- Add a vector column to store review text embeddings
    ALTER TABLE dbo.ProductReview
    ADD ReviewVector VECTOR(1536);
    GO
    ```

    > &#128221; `VECTOR(1536)` データ型には 1536 次元のベクトルが格納されますが、これは *text-embedding-3-small* モデルの出力に合わせたものです。 各要素は単精度 (4 バイト) 浮動小数点数として格納されるため、1536 次元のベクトルには行あたり約 6 KB が使用されます。

1. 埋め込み生成のテストを、最初はレビュー 1 件のみで行います。 このテストで、外部モデルと資格情報が機能していることを確認します。

    ```sql
    -- Test embedding generation on a single review
    SELECT TOP 1
        r.ReviewID,
        r.ReviewTitle,
        AI_GENERATE_EMBEDDINGS(
            r.ReviewTitle + ': ' + r.ReviewText
            USE MODEL my_embedding_model
        ) AS TestEmbedding
    FROM dbo.ProductReview r;
    GO
    ```

    > &#128221; 多数の浮動小数点数から成る配列が表示されるはずです。 これで、あなたの資格情報、外部モデル、Azure OpenAI デプロイがすべて正しく構成されていることが確定します。 エラーが返された場合は、マネージド ID のロールの割り当てが有効になっていることを確認してください (最大 5 分かかる可能性があります)。

1. ここで、すべてのレビューについて埋め込みをバッチで生成します。 このスクリプトは製品名、レビュー タイトル、レビュー テキストを結合してリッチな埋め込みを作成するものであり、1 バッチあたり 30 件のレビューを処理します。また、レート制限に対応するための再試行ロジックもあります。

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

---

## 生成された埋め込みを確認して検査する

埋め込みを生成した後に、すべてのレビューにベクトルがあることを確認し、ベクトルのプロパティを検査します。

1. 埋め込みを持つレビューの数を調べます。

    ```sql
    -- Verify embedding counts
    SELECT 
        COUNT(*) AS TotalReviews,
        COUNT(ReviewVector) AS ReviewsWithEmbeddings,
        COUNT(*) - COUNT(ReviewVector) AS ReviewsMissingEmbeddings
    FROM dbo.ProductReview;
    GO
    ```

    > &#128221; 140 件のレビューすべてに埋め込みがあるはずです。 欠落がある場合は、バッチ埋め込みスクリプトをもう一度実行してください。このスクリプトでは `ReviewVector IS NULL` の行のみが処理されます。

1. `VECTORPROPERTY` を使用してベクトル次元とデータ型を検査します。

    ```sql
    -- Check vector metadata
    SELECT TOP 1
        r.ReviewID,
        r.ReviewTitle,
        VECTORPROPERTY(r.ReviewVector, 'Dimensions') AS VectorDimensions,
        VECTORPROPERTY(r.ReviewVector, 'BaseType') AS VectorBaseType,
        DATALENGTH(r.ReviewVector) AS VectorSizeBytes
    FROM dbo.ProductReview r
    WHERE r.ReviewVector IS NOT NULL;
    GO
    ```

    > &#128221; `VECTORPROPERTY` は 1 つのベクトルに関するメタデータを返します。 1536 次元、`float` ベース型、およびベクトルあたり約 6,148 バイトと表示されるはずです。 この関数は、ベクトルの構造が期待どおりかどうかを検証するのに便利であり、特に次元不一致エラーのトラブルシューティングに役立ちます。

1. 実際のベクトル値のサンプルを、レビュー 1 件について表示します。

    ```sql
    -- View a sample embedding
    SELECT TOP 1
        r.ReviewID,
        r.ReviewTitle,
        LEFT(CAST(r.ReviewVector AS NVARCHAR(MAX)), 200) AS EmbeddingPreview
    FROM dbo.ProductReview r
    WHERE r.ReviewVector IS NOT NULL;
    GO
    ```

    > &#128221; 埋め込みは、浮動小数点数から成る JSON 配列です。 フル ベクトルには 1536 個の要素があるため、このクエリでは最初の 200 文字のみを表示します。 各数値は、埋め込みモデルがトレーニング中に学習した意味の次元の 1 つを表しています。

---

## 基本的なベクトル検索で埋め込みを検証する

埋め込みがセマンティック的意味を捉えていることを確認するために、`VECTOR_DISTANCE` を使用して基本的なベクトル類似性検索を実行します。 この関数は 2 つのベクトル間のコサイン距離を計算するものであり、値が小さいほど類似性が高いことを示します。

1. ある自然言語での質問に類似したレビューを検索します。

    ```sql
    -- Find reviews semantically similar to a question
    DECLARE @searchText NVARCHAR(1000) = 'Which tires last the longest and resist punctures?';
    DECLARE @searchVector VECTOR(1536);

    -- Generate an embedding for the search text
    SELECT @searchVector = AI_GENERATE_EMBEDDINGS(@searchText USE MODEL my_embedding_model);

    -- Find the 5 closest reviews by cosine distance
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

    > &#128221; 検索テキストとは異なる言い回しがレビュー テキストで使われていることもありますが、結果にはタイヤ関連のレビューのうち、パンクのしにくさや耐久性に関するものが返されるはずです。 このことから、埋め込みがセマンティック的な意味を捉えていることが確認されます。 `VECTOR_DISTANCE` にコサイン メトリックを指定すると、0 (同一) から 2 (反対) までの範囲内の値が返されます。 値が小さいほど、質問に対するそのレビューの関連性が高いことになります。

1. セマンティック検索と単純なキーワード検索を比較します。

    ```sql
    -- Keyword search for comparison
    SELECT TOP 5
        p.Name AS ProductName,
        r.ReviewTitle,
        r.ReviewText,
        r.Rating
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    WHERE r.ReviewText LIKE '%puncture%';
    GO
    ```

    > &#128221; `LIKE` 検索では、"puncture" という単語がこのとおりの形で含まれているレビューだけが見つかります。 ベクトル検索では、タイヤの耐久性や寿命に関するレビューが見つかり、これらは言葉遣いは異なっていても同じ概念について述べています。 この違いは、なぜ埋め込みが検索に対して有益であるかを表しています。

---

## 埋め込みのメンテナンスを探究する

埋め込みは、それが生成された時点のソース テキストのスナップショットです。 ソース テキストが変化すると埋め込みは陳腐化し、その時点の内容を反映したものではなくなります。 このセクションでは、この問題を観察して埋め込みの再生成を実践します。

1. レビューを 1 件選び、ある既知のクエリからの現在の埋め込み距離を調べます。

    ```sql
    -- Check distance of a specific review before update
    DECLARE @searchVector VECTOR(1536);
    SELECT @searchVector = AI_GENERATE_EMBEDDINGS('warm winter cycling gloves' USE MODEL my_embedding_model);

    SELECT
        r.ReviewID,
        r.ReviewTitle,
        r.ReviewText,
        VECTOR_DISTANCE('cosine', @searchVector, r.ReviewVector) AS DistanceBefore
    FROM dbo.ProductReview r
    WHERE r.ReviewID = 124;
    GO
    ```

    > &#128221; レビュー 124 は暖かい冬用手袋 (warm winter gloves) に関するものです。 距離の値に注目してください。 現時点では埋め込みにこの内容が反映されているため、距離は小さいはずです。

1. ここで、レビューのテキストを更新して別のものにします。 埋め込みは依然として元のテキストを反映したものであるため、この時点では陳腐化しています。

    ```sql
    -- Change the review text without updating the embedding
    UPDATE dbo.ProductReview
    SET ReviewText = 'These tires are the most puncture-resistant tires I have ever used. Over 3000 miles with zero flats on rough gravel roads.',
        ReviewTitle = N'Indestructible tires'
    WHERE ReviewID = 124;
    GO
    ```

1. もう一度距離を調べます。 埋め込みに反映されているのは手袋についての古いレビューのままですが、テキストはこの時点でタイヤに関するものになっています。

    ```sql
    -- Check distance after text change (embedding is stale)
    DECLARE @searchVector VECTOR(1536);
    SELECT @searchVector = AI_GENERATE_EMBEDDINGS('warm winter cycling gloves' USE MODEL my_embedding_model);

    SELECT
        r.ReviewID,
        r.ReviewTitle,
        r.ReviewText,
        VECTOR_DISTANCE('cosine', @searchVector, r.ReviewVector) AS DistanceAfterTextChange
    FROM dbo.ProductReview r
    WHERE r.ReviewID = 124;
    GO
    ```

    > &#128221; 距離は小さいままですが、その理由は埋め込みがまだ更新されていないからです。 ベクトルは依然として "暖かい冬用手袋" を表していますが、テキストはパンクしにくいタイヤに関するものに変化しています。 ベクトル検索で手袋を見つけようとすると、誤ってこのタイヤに関するレビューが返されます。 これが、埋め込みのメンテナンスが重要である理由です。

1. 更新されたレビューに合わせて埋め込みを再生成します。

    ```sql
    -- Regenerate the embedding to match the updated text
    UPDATE r
    SET r.ReviewVector = AI_GENERATE_EMBEDDINGS(
        p.Name + ' - ' + r.ReviewTitle + ': ' + r.ReviewText
        USE MODEL my_embedding_model)
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    WHERE r.ReviewID = 124;
    GO
    ```

1. 更新された内容が距離に反映されていることを確認します。

    ```sql
    -- Check distance after embedding regeneration
    DECLARE @searchVector VECTOR(1536);
    SELECT @searchVector = AI_GENERATE_EMBEDDINGS('warm winter cycling gloves' USE MODEL my_embedding_model);

    SELECT
        r.ReviewID,
        r.ReviewTitle,
        r.ReviewText,
        VECTOR_DISTANCE('cosine', @searchVector, r.ReviewVector) AS DistanceAfterRegeneration
    FROM dbo.ProductReview r
    WHERE r.ReviewID = 124;
    GO
    ```

    > &#128221; この時点で距離はかなり大きくなるはずです。埋め込みは手袋ではなくタイヤに関する内容を反映しているからです。 ベクトルとテキストは同期している状態に戻りました。実務では一般的に、この再生成は自動化され、その方法としてはトリガー、Change Tracking、変更データ キャプチャ、または外部プロセス (たとえば Azure Functions) が使用されます。 どのアプローチが最適かは、データが変化する頻度と、その変化をどれだけ早く埋め込みに反映する必要があるかによります。

1. 元のレビューを復元します。これは、後のラボに備えてデータの一貫性を保つためです。

    ```sql
    -- Restore the original review text and regenerate the embedding
    UPDATE dbo.ProductReview
    SET ReviewTitle = N'Warm hands well below freezing',
        ReviewText = N'Rode through an entire winter with the Full-Finger Gloves and never had cold fingers even at minus 5 degrees with wind chill. The fleece lining is cozy without being bulky and I can still operate my brake levers precisely. Essential cold weather gear.'
    WHERE ReviewID = 124;
    GO

    -- Regenerate the embedding for the restored text
    UPDATE r
    SET r.ReviewVector = AI_GENERATE_EMBEDDINGS(
        p.Name + ' - ' + r.ReviewTitle + ': ' + r.ReviewText
        USE MODEL my_embedding_model)
    FROM dbo.ProductReview r
    INNER JOIN SalesLT.Product p ON r.ProductID = p.ProductID
    WHERE r.ReviewID = 124;
    GO
    ```

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

この演習では、Azure SQL Database でのベクトル埋め込みの格納と生成の方法を学びました。 ベクトル列を追加し、埋め込みを個別に生成するとともに、再試行ロジック付きでバッチで生成しました。 ベクトルのメタデータ (次元やストレージ サイズなど) を検査しました。 埋め込みを検証するために類似性検索を行い、結果を従来のキーワード検索と比較しました。 最後に、ソース テキストが変化すると埋め込みが陳腐化することを観察し、一貫性を回復するために再生成する方法を学びました。
