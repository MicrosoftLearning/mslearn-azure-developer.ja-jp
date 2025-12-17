---
lab:
  topic: Azure authentication and authorization
  title: MSAL.NET を使用して対話型認証を実装する
  description: MSAL.NET SDK を使用して対話型認証を実装し、トークンを取得する方法について説明します。
---

# MSAL.NET を使用して対話型認証を実装する

この演習では、Microsoft Entra ID にアプリケーションを登録し、MSAL.NET を使用して対話型認証を実行し、Microsoft Graph のアクセス トークンを取得する .NET コンソール アプリケーションを作成します。 認証スコープを構成し、ユーザーの同意を処理する方法について説明し、後続の実行のためにトークンがキャッシュされるしくみを確認します。

この演習で実行されるタスク:

* Microsoft ID プラットフォームにアプリケーションを登録する
* 認証を構成するために **PublicClientApplicationBuilder** クラスを実装する .NET コンソール アプリを作成します。
* **user.read** Microsoft Graph アクセス許可を使用して、対話的にトークンを取得します。

この演習の所要時間は約 **15** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

* Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。

* [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。

* [.NET 8](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) 以降。

* Visual Studio Code 用の [C# 開発キット](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)

## 新しいアプリケーションの登録

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ポータルで **[アプリの登録]** を検索して選択します。

1. **[+ 新しい登録]** を選択し、**[アプリケーションの登録]** ページが表示されたら、アプリケーションの登録情報を入力します。

    | フィールド | 値 |
    |--|--|
    | **名前** | 「`myMsalApplication`」と入力します  |
    | **サポートされているアカウントの種類** | **[この組織ディレクトリのみに含まれるアカウント]** を選択します。 |
    | **リダイレクト URI (省略可能)** | **[パブリック クライアント/ネイティブ (モバイルとデスクトップ)]** を選択し、右側のボックスに「`http://localhost`」と入力します。 |

1. **登録** を選択します。 Microsoft Entra ID によってアプリケーションに一意のアプリケーション (クライアント) ID が割り当てられ、アプリケーションの「**概要**」ページが表示されます。

1. **[概要]** ページの **[基本情報]** セクションで、**[アプリケーション (クライアント) ID]** と **[ディレクトリ (テナント) ID]** をメモします。 アプリケーションにはこの情報が必要です。

    ![コピーするフィールドの位置を示すスクリーンショット。](./media/01-app-directory-id-location.png)

## トークンを取得する .NET コンソール アプリを作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 次の手順はローカル環境で実行されます。

1. プロジェクト用に **authapp** または任意の名前のフォルダーを作成します。

1. **Visual Studio Code** を起動し、**[ファイル] > [フォルダーを開く]** を選択して、プロジェクト フォルダーを選択します。

1. **[表示] > [ターミナル]** を選択し、ターミナルを開きます。

1. VS Code ターミナルで次のコマンドを実行し、.NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、**Microsoft.Identity.Client** および **dotenv.net** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Microsoft.Identity.Client
    dotnet add package dotenv.net
    ```

### コンソール アプリケーションを構成する

このセクションでは、先ほどメモしたシークレットを保持する **.env** ファイルを作成し、編集します。

1. **[ファイル] > [新しいファイル]** を選択し、プロジェクト フォルダーに *.env* というファイルを作成します。

1. **.env** ファイルを開き、次のコードを追加します。 **YOUR_CLIENT_ID** と **YOUR_TENANT_ID** を、先ほどメモした値に置き換えます。

    ```
    CLIENT_ID="YOUR_CLIENT_ID"
    TENANT_ID="YOUR_TENANT_ID"
    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

### プロジェクトのスタート コードを追加する

1. *Program.cs* ファイルを開き、既存の内容を次のコードに置き換えます。 コード内のコメントを必ず確認してください。

    ```csharp
    using Microsoft.Identity.Client;
    using dotenv.net;

    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();

    // Retrieve Azure AD Application ID and tenant ID from environment variables
    string _clientId = envVars["CLIENT_ID"];
    string _tenantId = envVars["TENANT_ID"];

    // ADD CODE TO DEFINE SCOPES AND CREATE CLIENT



    // ADD CODE TO ACQUIRE AN ACCESS TOKEN


    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

### コードを追加してアプリケーションを完成させます

1. **// ADD CODE TO DEFINE SCOPES AND CREATE CLIENT** コメントを見つけて、コメントの直後に次のコードを追加します。 コード内のコメントを必ず確認してください。

    ```csharp
    // Define the scopes required for authentication
    string[] _scopes = { "User.Read" };

    // Build the MSAL public client application with authority and redirect URI
    var app = PublicClientApplicationBuilder.Create(_clientId)
        .WithAuthority(AzureCloudInstance.AzurePublic, _tenantId)
        .WithDefaultRedirectUri()
        .Build();
    ```

1. **// ADD CODE TO ACQUIRE AN ACCESS TOKEN** コメントを見つけて、コメントの直後に次のコードを追加します。 コード内のコメントを必ず確認してください。

    ```csharp
    // Attempt to acquire an access token silently or interactively
    AuthenticationResult result;
    try
    {
        // Try to acquire token silently from cache for the first available account
        var accounts = await app.GetAccountsAsync();
        result = await app.AcquireTokenSilent(_scopes, accounts.FirstOrDefault())
                    .ExecuteAsync();
    }
    catch (MsalUiRequiredException)
    {
        // If silent token acquisition fails, prompt the user interactively
        result = await app.AcquireTokenInteractive(_scopes)
                    .ExecuteAsync();
    }

    // Output the acquired access token to the console
    Console.WriteLine($"Access Token:\n{result.AccessToken}");
    ```

1. **Ctrl + S** キーを押してファイルを保存してから、**Ctrl + Q** キーを押してエディターを終了します。

## アプリケーションの実行

アプリが完成したので、実行してみましょう。

1. 次のコマンドを実行してアプリケーションを起動します。

    ```
    dotnet run
    ```

1. アプリによって既定のブラウザーが開かれ、認証するアカウントを選択するよう求められます。 複数のアカウントが一覧表示されている場合は、アプリで使用されるテナントに関連付けられているものを選択します。

1. 登録したアプリの認証を初めて受けた場合は、"**要求されているアクセス許可**" という通知が表示され、アプリによるサインイン、プロファイルの読み取り、自分がアクセスを許可したデータへのアクセス権の維持を承認するよう求められます。 **[Accept](承認)** を選択します。

    !["要求されているアクセス許可" 通知を示すスクリーンショット](./media/01-granting-permission.png)

1. コンソールに次の例のような結果が表示されます。

    ```
    Access Token:
    eyJ0eXAiOiJKV1QiLCJub25jZSI6IlZF.........
    ```

1. アプリケーションをもう一度起動すると、"**要求されているアクセス許可**" 通知は表示されなくなります。 先ほど付与したアクセス許可はキャッシュされました。 **注:**  複数のアカウントをお持ちの場合、一部のアカウント構成では、通知が再び表示される場合があります。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。
