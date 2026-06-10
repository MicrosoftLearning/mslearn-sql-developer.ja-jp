---
lab:
  title: ラボ 7 - SQL データベース プロジェクトで CI/CD を実装する
  module: Implement CI/CD by using SQL Database Projects
  description: この演習は、SQL データベース プロジェクトの作成、そのローカルでのビルド、GitHub へのプッシュ、スキーマの自動デプロイのための GitHub Actions パイプラインの構成に役立ちます。
  level: 300
  duration: 45 minutes
  islab: true
  primarytopics:
    - Azure SQL Database
    - CI/CD
    - GitHub Actions
    - SQL Database Projects
---

# SQL データベース プロジェクトで CI/CD を実装する

**予測される所要時間**: 45 分

このラボでは、SQL データベース プロジェクトを作成して、ローカルで構築し、GitHub リポジトリにプッシュし、スキーマの変更を自動的に Azure SQL Database にビルドおよびデプロイする GitHub Actions パイプラインを構成します。

あなたは、Adventure Works のデータベース管理者です。 チームは、運用環境に対してスクリプトを手動で実行してデータベースの変更を適用してきました。 最近のデプロイであるメンバーが順番を間違えてスクリプトを実行したためにテーブルが破損した件で、マネージャーから自動パイプラインの設定を依頼されています。 あなたは、スキーマの信頼できる情報源には SQL データベース プロジェクトを使用し、変更のビルドとデプロイには GitHub Actions を使用します。

> &#128221; これらの演習では、T-SQL コードと YAML コンテンツをコピーして貼り付けるように求められます。 コードが正しくコピーされていることを確認してから、実行またはコミットしてください。

## 前提条件

この演習を開始する前に、次のアカウントとツールが設定されていることを確認します。

- [Azure サブスクリプション](https://azure.microsoft.com/free)。
- [GitHub アカウント](https://github.com/join)。
- [Git](https://git-scm.com/downloads) がマシンにインストールされている。
- マシンに [GitHub CLI (gh)](https://cli.github.com/) がインストールされている。
- [MSSQL 拡張機能](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql)がインストールされている [Visual Studio Code ](https://code.visualstudio.com/download) (MSSQL 拡張機能には SQL データベース プロジェクトのサポートが含まれます)。
- [.NET SDK 8.0](https://dotnet.microsoft.com/download/dotnet/8.0) 以降。

## Azure SQL Database をプロビジョニングする

まず、サンプル データを含む Azure SQL Database を作成します。

> &#128221; 既に AdventureWorksLT Azure SQL Database がプロビジョニング済みの場合は、このセクションをスキップしてください。

1. [Azure SQL ハブ](https://aka.ms/azuresqlhub) に移動して、Azure アカウントでサインインします (要求された場合)。 **[Azure SQL Database]** ペインの **[オプションの表示]** を選択し、**[SQL データベースの作成]** を選択します。

    > &#128161; このページに**無料オファー**のバナーがある場合は、これを利用して Azure SQL Database を無料でお使いいただけます。 この[無料オファー](https://learn.microsoft.com/azure/azure-sql/database/free-offer)では、サーバーレス コンピューティング 100,000 仮想コア秒とストレージ 32 GB (1 か月あたり) が提供されます。 無料オファーを利用する場合は、手順 3 から 6 をスキップします。

1. **[SQL データベースの作成]** ページの **[基本]** タブに、必要な情報を入力します。

    | 設定 | 値 |
    | --- | --- |
    | **サブスクリプション** | Azure サブスクリプションを選択します。 |
    | **リソース グループ** | リソース グループを選択または作成します。 |
    | **データベース名** | *AdventureWorksLT* |
    | **[サーバー]** | **[新規作成]** を選択して一意の名前を持つ新しいサーバーを作成します。 **場所**を選択します。 認証には、次のいずれかのオプションを選択して、**[OK]** を選択します: |

    > &#128221; **認証は省略可能ではありません。** 所属する組織のセキュリティ ポリシーに適合する方法を選択する必要があります。 どのオプションを選ぶかに応じて、後でデータベースに接続する方法が異なります。
    > - **Microsoft Entra 認証のみを使用する** (推奨): これを選択するのは、組織で Entra ベースのアクセスが必須とされている場合です。** ご自分の Azure アカウントを **Microsoft Entra 管理者**として設定してください。データベースへの接続には Microsoft Entra アカウントを使用します (たとえば SSMS では "認証" = **[Microsoft Entra MFA]** を選択します)。**
    > - **SQL と Microsoft Entra 両方の認証/SQL 認証を使用する**: これを選択するのは SQL 管理者としてログインしたい場合、または組織で両方の方法が許可されている場合です。 **[サーバー管理者ログイン]** と **[パスワード]** を入力します。 接続にはこれらの認証情報が必要です。 また、**[Microsoft Entra 管理者]** を設定しておくと、SQL 認証に加えて Entra ログインを有効にすることができます。

    > &#128221; 使用できるテスト サーバーが既にある場合は、新規作成せずにそのサーバーを選択します。

1. **[SQL エラスティック プールを使用しますか?]** を **[いいえ]** に設定したままにします。
1. **[ワークロード環境]** で、**[開発]** を選択します。 これでコンピューティングが **[General Purpose - サーバーレス]** に事前設定され、自動一時停止が有効化されますが、これは有料オプションの中で最もコストを抑えられる構成です。
1. **[コンピューティングとストレージ]** で、 **[データベースの構成]** を選択します。 サービス レベルを **[Hyperscale]** に、コンピューティング レベルを **[サーバーレス]** に変更します。 **[適用]** を選択して確定します。
1. **[バックアップ ストレージの冗長性]** の下で **[ローカル冗長バックアップ ストレージ]** を選択します。
1. **[Next: Networking]\(次へ: ネットワーク\)** を選択します。
1. **[ネットワーク]** タブの **[接続方法]** で、 **[パブリック エンドポイント]** を選択します。

    > &#128221; 新規作成ではなく既存のサーバーを選択した場合は、**[接続方法]** オプションが表示されないことがありますが、これはそのサーバー上で既に設定されているためです。

1. **[ファイアウォール規則]** の下の **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** を **[はい]** に設定し、**[現在のクライアント IP アドレスを追加する]** を **[はい]** に設定します。
1. **[次へ: セキュリティ]**、**[次へ: 追加設定]** の順に選択します。
1. **[追加設定]** タブの **[データソース]** で、**[既存のデータを使用する]** を [サンプル] に設定して AdventureWorksLT サンプル データベースを作成します。** 確認を求められたら **[OK]** を選択します。
1. **[確認および作成]** を選択し、設定を確認してから、**[作成]** を選択します。
1. デプロイが完了するのを待ってから、新しい Azure SQL Database リソースに移動します。

## 開発環境を構成する

Git を設定し、GitHub で認証し、SQL プロジェクト テンプレートをインストールします。

1. Visual Studio Code でターミナル (**[ターミナル]** > **[新しいターミナル]**) を開き、次のコマンドを実行して、必要なツールがインストールされていることを確認します。

    ```bash
    git --version
    gh --version
    dotnet --version
    ```

    <blockquote>
    <div markdown="1">

    &#128221; これらのコマンドのいずれかが認識されない場合は、不足しているツールをインストールしてから次に進んでください。

    - **Git**: [https://git-scm.com/downloads](https://git-scm.com/downloads) からダウンロードしてインストールする。
    - **GitHub CLI**: [https://cli.github.com](https://cli.github.com) からダウンロードしてインストールする。
    - **.NET SDK 8.0+**: [https://dotnet.microsoft.com/download/dotnet/8.0](https://dotnet.microsoft.com/download/dotnet/8.0) からダウンロードしてインストールする。

    </div>
    </blockquote>

    <blockquote>
    <div markdown="1">

    &#128161; インストール後は、ターミナルだけでなく **Visual Studio Code を完全に閉じてから再起動する**と、更新されたシステム PATH が認識されます。 Visual Studio Code のターミナルでコマンドが認識されない場合は、次のコマンドを実行して現在の PowerShell セッションで手動で PATH を更新します。

    `$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")`

    </div>
    </blockquote>

1. Git ID を設定します。

    > &#128221; 前の演習で既に Git の ID を設定済みの場合は、この手順をスキップします。 ターミナルで `git config --global user.name` と `git config --global user.email` を実行すると確認できます。 両方で値が返された場合は、手順 3 に進みます。

    ```bash
    git config --global user.name "Your Name"
    git config --global user.email "your-email@example.com"
    ```

1. GitHub CLI を使って GitHub で認証します。

    ```bash
    gh auth login
    ```

    ダイアログが表示されたら、**[GitHub.com]** を選択し、プロトコルとして **[HTTPS]** を選択し、ブラウザーで認証します。

1. SQL データベース プロジェクト テンプレートをインストールします。

    ```bash
    dotnet new install Microsoft.Build.Sql.Templates
    ```

    > &#128221; このパッケージにより、`sqlproj` テンプレートが .NET CLI に追加されます。 これはマシンごとに 1 回だけインストールする必要があります。

## SQL データベース プロジェクトを作成する

新しい SQL データベースプロジェクトを作成し、2 つのデータベース オブジェクト (テーブルとストアド プロシージャ) を追加します。

1. プロジェクト フォルダーを作成し、SQL プロジェクトを初期化します (必要に応じて、プロジェクトを作成するディレクトリに移動してからこれらのコマンドを実行します):

    ```bash
    mkdir AdventureWorksDB
    cd AdventureWorksDB
    ```

1. Azure SQL Database をターゲット プラットフォームとして SQL データベース プロジェクトを初期化します。

    ```bash
    dotnet new sqlproj -tp SqlAzureV12
    ```

    > &#128221; `-tp SqlAzureV12` フラグはターゲット プラット フォームを Azure SQL Database に設定します。 これを使用しない場合、プロジェクトは既定で最新の SQL Server バージョンに設定され、`.dacpac` は Azure SQL Database に配置されません。 これにより、現在のディレクトリに `AdventureWorksDB.sqlproj` という名前のオブジェクト ファイルが作成されます。 このファイルは Microsoft.Build.Sql SDK を参照し、`dotnet build` で T-SQL を `.dacpac` にコンパイルできるようになります。

1. テーブル定義を保存するための **Tables** フォルダーを作成します。

    ```bash
    mkdir Tables
    ```

1. Visual Studio Code で **AdventureWorksDB** フォルダー (**[ファイル]** > **[フォルダーを開く]**) を開きます。 **[エクスプローラー]** ペインで、**Tables** フォルダーを右クリックし、**[新しいファイル]** を選択します。 ファイルに `InventoryLog.sql` と名前を付け、次の内容を追加します。

    ```sql
    CREATE TABLE [dbo].[InventoryLog] (
        [LogID] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        [ProductID] INT NOT NULL,
        [ChangeDate] DATETIME2 NOT NULL DEFAULT GETDATE(),
        [QuantityChange] INT NOT NULL,
        [ChangeType] NVARCHAR(20) NOT NULL
    );
    ```

1. プロシージャ定義を保存するための **StoredProcedures** フォルダーを作成します。

    ```bash
    mkdir StoredProcedures
    ```

1. **[エクスプローラー]** ウィンドウで、**StoredProcedures** フォルダーを右クリックし、**[新しいファイル]** を選択します。 ファイルに `uspLogInventoryChange.sql` と名前を付け、次の内容を追加します。

    ```sql
    CREATE PROCEDURE [dbo].[uspLogInventoryChange]
        @ProductID INT,
        @QuantityChange INT,
        @ChangeType NVARCHAR(20)
    AS
    BEGIN
        INSERT INTO [dbo].[InventoryLog] (ProductID, QuantityChange, ChangeType)
        VALUES (@ProductID, @QuantityChange, @ChangeType);
    END;
    ```

    > &#128221; SDK スタイルの SQL プロジェクトでは既定のグロビングが使用されます。つまり、プロジェクト ディレクトリまたはサブディレクトリ内の `.sql` ファイルは自動的にビルドに含まれます。 `.sqlproj` 内の各ファイルをリストアップする必要はありません。

## ローカル環境でプロジェクトをビルドする

プロジェクトをビルドして、T-SQL のコンパイルと `.dacpac` が生成されることを確認します。

1. `AdventureWorksDB` ディレクトリから、次を実行します。

    ```bash
    dotnet build
    ```

1. ビルド出力に末尾が `AdventureWorksDB -> ...bin/Debug/AdventureWorksDB.dacpac` の行が含まれていることを確認します。

    > &#128221; `.dacpac` はデプロイ可能なアーティファクトです。 プロジェクト内のすべてのオブジェクトのスキーマ定義が含まれています。 SqlPackage (または `azure/sql-action`) は、この `.dacpac` をターゲット データベースと比較し、データベースを同期させるために必要な ALTER/CREATE ステートメントを生成します。

1. ビルドが失敗した場合は、ターミナル出力のエラーを確認します。 一般的な問題には、セミコロンの欠落、かっこの不一致、列参照の誤りなどがあります。

## Git を初期化し、GitHub リポジトリを作成する

バージョン管理を設定し、プロジェクトを GitHub にプッシュします。 ターミナルが引き続き **AdventureWorksDB** フォルダー (`AdventureWorksDB.sqlproj` を含むのと同じディレクトリ) にあることを確認します。

1. プロジェクト フォルダー内の Git リポジトリを初期化します。

    ```bash
    git init -b main
    ```

1. Visual Studio Code の **[エクスプローラー]** ペインで、ファイル リストの上部にある **[新しいファイル]** "アイコン" を選択します。** ファイルに `.gitignore` という名前を付け、次の内容を追加してビルド出力を除外し、ファイルを保存します。

    ```text
    bin/
    obj/
    ```

1. ターミナルで GitHub リポジトリを作成し、ローカル プロジェクトにリンクします:

    ```bash
    gh repo create AdventureWorksDB --private --source=. --remote=origin
    ```

    > &#128221; このコマンドは、GitHub アカウントに `AdventureWorksDB` という名前のプライベート リポジトリ (`https://github.com/YourGitHubUser/AdventureWorksDB`など) を作成し、これを `origin` という名前のリモートとして追加します。 ファイルは、まだプッシュされません。

## デプロイ用に GitHub シークレットを構成する

Azure SQL Database にデプロイするには、パイプラインに資格情報が必要です。 手順はサーバーの認証モードによって異なります。

### オプション A: SQL 認証 (既定)

サーバーで SQL 認証がサポートされている場合は、接続文字列シークレットのみが必要です。

1. Azure portal で、**SQL データベース** リソースに移動します。
1. 左側のメニューで、**[設定]** の **[接続文字列]** を選択します。
1. **[ADO.NET (SQL 認証)]** タブで、接続文字列をコピーします。 それは次のようになります。

    ```text
    Server=tcp:yourserver.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Persist Security Info=False;User ID=youradmin;Password={your_password};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
    ```

1. コピーした文字列の `{your_password}` を、サーバーの作成時に設定した実際のパスワードに置き換えます。
1. ターミナルで、接続文字列を GitHub のシークレットとして追加します:

    ```bash
    gh secret set SQL_CONNECTION_STRING
    ```

    ダイアログが表示されたら、完全な接続文字列 (実際のパスワードを含む) を貼り付け、**Enter** キーを押します。

    > &#128221; GitHub ではシークレット値が暗号化されます。 ログやリポジトリを表示するユーザーには表示されません。 ワークフローは `{% raw %}${{ secrets.SQL_CONNECTION_STRING }}{% endraw %}` を通じてこれにアクセスします。

1. 「**GitHub Actions ワークフロー の作成**」セクションに進み、ワークフロー YAML の**オプション A** を使用します。

### オプション B: Microsoft Entra のみの認証 (Entra のみの代替手段)

サーバーで Microsoft Entra のみの認証を使用している場合は、Microsoft Entra ID でアプリを登録し、GitHub Actions のフェデレーション資格情報を構成し、複数のシークレットを格納する必要があります。

1. Azure portal で、**[Microsoft Entra ID]** > **[管理]** > **[アプリの登録]** > **[新規登録]** の順に移動します。
1. **[名前]** に `sqldeploysp` を設定し、他の設定はすべて既定値のままにして **[登録]** を選択します。
1. アプリの登録の **[概要]** ページで、**アプリケーション (クライアント) ID** と**ディレクトリ (テナント) ID** をメモします。 これらの値は後で必要になります。
1. **[管理]** > **[証明書とシークレット]** > **[フェデレーション資格情報]** > **[資格情報の追加]** の順に選択します。
1. **[Azure リソースをデプロイする GitHub Actions]** を選択し、次のように入力します。

    | 設定 | 値 |
    | --- | --- |
    | **組織** | ご自分の GitHub ユーザー名** |
    | **リポジトリ** | *AdventureWorksDB* |
    | **エンティティの種類** | *ブランチ* |
    | **GitHub ブランチ名** | *main* |
    | **名前** | *github-actions-deploy* |

1. **[追加]** を選択します。

1. SQL サーバーを含む**リソース グループ**に移動し、**[アクセス制御 (IAM)]** > **[追加]** > **[ロールの割り当ての]** の順に選択します。
1. **[ロール]** タブで `SQL Server Contributor` を検索して選択し、**[次へ]** を選択します。

    > &#128221; **SQL Server Contributor** ロールは最小権限の原則に従っています。 これは、リソース グループ内のすべてのリソースに広範なアクセス権を付与せずに、SQL サーバーとファイアウォール規則を管理するアクセス許可を付与します。 `azure/sql-action` はデプロイ時に一時的なファイアウォール規則を作成および削除するために、この役割を必要とします。 実際のデータベース アクセスは、次の手順で構成する SQL レベルの `db_owner` 許可によって個別に処理されます。

1. **[メンバー]** タブで、**[ユーザー、グループ、またはサービス プリンシパル]** を選択してから、**[+ メンバーの選択]** を選択します。 `sqldeploysp` を検索して選択し、**[レビューと割り当て]** を選択して、もう一度 **[レビューと割り当て]** を選択して完了します。

1. サブスクリプション ID を取得します。 **SQL Server** リソースに移動し、**[概要]** ページから**サブスクリプション ID** をコピーします。

1. サービス プリンシパルに SQL データベースへのアクセス権を付与します。 **[SQL データベース]** リソースにアクセスし、**[クエリ エディター(プレビュー)]** または **[SQL Server Management Studio (SSMS)]** を選択します。 サインインして次を実行します。

    ```sql
    CREATE USER [sqldeploysp] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_owner ADD MEMBER [sqldeploysp];
    ```

1. ターミナルで GitHub のシークレットを追加します。

    ```bash
    gh secret set AZURE_CLIENT_ID
    gh secret set AZURE_TENANT_ID
    gh secret set AZURE_SUBSCRIPTION_ID
    gh secret set SQL_CONNECTION_STRING
    ```

    各シークレットを求められたら、対応する値を貼り付けます。
    - `AZURE_CLIENT_ID`: 手順 3 の **アプリケーション (クライアント) ID**。
    - `AZURE_TENANT_ID`: 手順 3 の**ディレクトリ (テナント) ID**。
    - `AZURE_SUBSCRIPTION_ID`: 手順 11 の**サブスクリプション ID**。
    - `SQL_CONNECTION_STRING`: 次のフォーマットを使用します (`yourserver` をサーバー名に置き換えます):

    ```text
    Server=tcp:yourserver.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;Authentication=Active Directory Default;
    ```

    > &#128221; `Active Directory Default` 認証方法を使用すると、`azure/sql-action` は、`azure/login` の手順で確立されたサービス プリンシパル ID を使用できます。 接続文字列にパスワードは必要ありません。

## GitHub Actions ワークフローを作成する

SQL プロジェクトを構築し、`main` へのプッシュごとに `.dacpac` を Azure SQL Database にデプロイするワークフロー ファイルを作成します。

1. ターミナルでワークフロー ファイルのフォルダー構造を作成します:

    ```bash
    mkdir -p .github/workflows
    ```

    > &#128221; `.github` フォルダーが隠しディレクトリの場合、これを VS Code の [エクスプローラー] ペインで確認するには、**[ファイル]** > **[ユーザー設定]** > **[設定]** を選択し、`files.exclude` を検索し、`**/.github` パターンが表示されていたら、これを削除するか無効にします。

1. **[エクスプローラー]** ペインで **workflows**フォルダー (.github の下) を右クリックし、**[新しいファイル]** を選択します。 そのファイルに `build-deploy.yml` という名前を付けます。

1. 認証方法に基づいてワークフローの内容を追加し、ファイルを保存します。

    ### オプション A: SQL 認証ワークフロー

    ```yaml
    name: Build and Deploy SQL Database Project

    on:
      push:
        branches:
          - main

    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest

        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Setup .NET SDK
            uses: actions/setup-dotnet@v4
            with:
              dotnet-version: '8.x'

          - name: Build SQL project
            run: dotnet build AdventureWorksDB.sqlproj

          - name: Install SqlPackage
            run: dotnet tool install -g microsoft.sqlpackage

          - name: Deploy to Azure SQL Database
            uses: azure/sql-action@v2.3
            with:
              connection-string: {% raw %}${{ secrets.SQL_CONNECTION_STRING }}{% endraw %}
              path: ./bin/Debug/AdventureWorksDB.dacpac
              action: publish
    ```

    ### オプション B: Microsoft Entra 専用認証ワークフロー

    ```yaml
    name: Build and Deploy SQL Database Project

    on:
      push:
        branches:
          - main

    permissions:
      id-token: write
      contents: read

    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest

        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Setup .NET SDK
            uses: actions/setup-dotnet@v4
            with:
              dotnet-version: '8.x'

          - name: Build SQL project
            run: dotnet build AdventureWorksDB.sqlproj

          - name: Install SqlPackage
            run: dotnet tool install -g microsoft.sqlpackage

          - name: Azure Login
            uses: azure/login@v2
            with:
              client-id: {% raw %}${{ secrets.AZURE_CLIENT_ID }}{% endraw %}
              tenant-id: {% raw %}${{ secrets.AZURE_TENANT_ID }}{% endraw %}
              subscription-id: {% raw %}${{ secrets.AZURE_SUBSCRIPTION_ID }}{% endraw %}

          - name: Deploy to Azure SQL Database
            uses: azure/sql-action@v2.3
            with:
              connection-string: {% raw %}${{ secrets.SQL_CONNECTION_STRING }}{% endraw %}
              path: ./bin/Debug/AdventureWorksDB.dacpac
              action: publish
    ```

    > &#128221; 両方のワークフローは SQL プロジェクトを `.dacpac` にビルドし、`azure/sql-action` を使ってデプロイします。 違いは、オプション B が OIDC フェデレーション資格情報を使用して `azure/login` の手順を追加することです。これにより、`azure/sql-action` での Microsoft Entra を使用した認証が可能になります。 `permissions` ブロックは、Azure で認証するトークンをワークフローに付与します。 `publish` アクションは必要に応じて ALTER および CREATE ステートメントを生成するため、デプロイ スクリプトを手作業で書く必要がありません。

## 最初のデプロイをプッシュして検証する

プロジェクトを GitHub にプッシュし、パイプライン実行を監視し、テーブルが Azure SQL Database に作成されたことを確認します。

1. すべてのファイルをステージ、コミット、プッシュします。

    ```bash
    git add -A
    git commit -m "Initial SQL project with InventoryLog table and CI/CD pipeline"
    git push -u origin main
    ```

1. ブラウザーで GitHub リポジトリを開きます。

    ```bash
    gh browse
    ```

1. **[アクション]** タブを選択します。ワークフローが実行中か、最近完了したことがわかります。 これを選択して、ビルドとデプロイの手順を表示します。

1. ワークフローが、緑色のチェックマーク付きで完了していれば、デプロイは成功です。 失敗した場合は、失敗した手順を選択してエラー メッセージを読みます。

    > &#128221; この段階での一般的なエラーには、正しくない接続文字列 (シークレット値を確認します)、保存されていないファイアウォール規則 (サーバーの [ネットワーク] ページに戻ります)、ワークフロー YAML のファイル パスの入力ミスなどがあります。

1. テーブルが Azure SQL Database に作成されたことを確認します。 Azure portal で、SQL データベースに移動し **[クエリ エディター (プレビュー)]** を選択するか、**[SQL Server Management Studio (SSMS)]** を使用します。
1. プロビジョニング中に設定した SQL 管理者資格情報でサインインします。
1. 次のクエリを実行します。

    ```sql
    SELECT TABLE_SCHEMA, TABLE_NAME
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_NAME = 'InventoryLog';
    ```

    > &#128221; `dbo.InventoryLog` を含む列が表示されています。 このテーブルは以前は存在しませんでした。 これは、パイプラインが `.dacpac` をデプロイすることにより作成されました。

1. ストアド プロシージャも作成されていることを確認します。

    ```sql
    SELECT ROUTINE_SCHEMA, ROUTINE_NAME
    FROM INFORMATION_SCHEMA.ROUTINES
    WHERE ROUTINE_NAME = 'uspLogInventoryChange';
    ```

## スキーマの変更をプッシュする

テーブルの変更、変更のプッシュ、パイプラインでのデータベースの更新を確認して、完全なサイクルをテストします。

1. Visual Studio Code で、`Tables/InventoryLog.sql` を開き、閉じかっこの前に `Notes` 列を追加します。

    ```sql
    CREATE TABLE [dbo].[InventoryLog] (
        [LogID] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        [ProductID] INT NOT NULL,
        [ChangeDate] DATETIME2 NOT NULL DEFAULT GETDATE(),
        [QuantityChange] INT NOT NULL,
        [ChangeType] NVARCHAR(20) NOT NULL,
        [Notes] NVARCHAR(200) NULL
    );
    ```

    > &#128221; ALTER TABLE スクリプトを記述しているのではなく、CREATE TABLE ステートメントを編集していることに注目してください。 SQL プロジェクトは宣言型です。 テーブルの外観を定義します。 パイプラインによりこの更新された `.dacpac` がデプロイされると、`azure/sql-action` が既存のテーブルと比較され、自動的に `ALTER TABLE ADD` ステートメントが生成されます。

1. ローカルでビルドして、変更がコンパイルされるようにします。

    ```bash
    dotnet build
    ```

1. 変更をコミットしてプッシュします。

    ```bash
    git add -A
    git commit -m "Add Notes column to InventoryLog table"
    git push
    ```

1. ブラウザーで、GitHub リポジトリの **[Actions]** タブを選択し、新しいワークフローの実行を監視します。

1. ワークフローが完了したら、Azure portal の **クエリ エディター** に戻るか、**SQL Server Management Studio (SSMS)** を使用して次を実行します。

    ```sql
    SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
    FROM INFORMATION_SCHEMA.COLUMNS
    WHERE TABLE_NAME = 'InventoryLog'
    ORDER BY ORDINAL_POSITION;
    ```

    > &#128221; `Notes` 列にデータ型 `nvarchar`、`IS_NULLABLE` に `YES` が設定されて一覧表示されます。 パイプラインは、`.dacpac` とライブ データベースの違いを検出し、ALTER TABLE ステートメントを生成して適用しました。これらはすべて SQL ファイルの 1 行の変更のみによるものです。

## クリーンアップ

コストが発生しないようにするため、この演習で作成したリソースを削除します。

1. GitHub リポジトリを削除します。

    ```bash
    gh repo delete AdventureWorksDB --yes
    ```

    > &#128221; **403** エラー "リポジトリに対する管理者権限が必要です" が表示される場合は、GitHub CLI トークンに `delete_repo` スコープが含まれていません。** `gh auth refresh -h github.com -s delete_repo` を実行してこれを追加してから、delete コマンドをもう一度実行してみてください。

1. Azure portal で、**[SQL Server]** リソースに移動し、**[ネットワーク]** を選択します。 **[ファイアウォール規則]** で `AllowGitHubRunners` 規則を削除し、**[保存]** を選択します。

> &#128221; このコースのすべての演習が完了したら、Azure SQL Database またはリソース グループ全体を削除すると、すべての料金を停止できます。 コース内の他の演習で同じデータベースを使用する場合は、それを保持し、ファイアウォール規則のみを削除します。

データベースが不要になった場合は、次の手順で削除できます。

1. Azure portal で、**SQL データベース** リソースに移動します。
1. 上部のメニューから **[削除]** を選択し、データベース名を入力して確認し、**[削除]** を選択します。


これでこの演習は完了です。

## 要点

この演習では、テーブルとストアド プロシージャを使用して SQL データベース プロジェクトを作成し、`.dacpac` をローカルにビルドし、プロジェクトを GitHub にプッシュしました。 プロジェクトをビルドし、`azure/sql-action` を使用して Azure SQL Database にデプロイする GitHub Actions ワークフローを構成しました。 その後、スキーマの変更をプッシュし (テーブルに列を追加)、パイプラインで差異が検出され、ALTER TABLE が自動的に適用されたことを確認しました。 この演習では、宣言型スキーマの編集、ソース管理へのプッシュ、パイプラインによるビルドとデプロイの処理など、データベース プロジェクトの中核的な CI/CD のワークフローを示しました。
