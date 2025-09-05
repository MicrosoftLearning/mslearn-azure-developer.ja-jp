---
lab:
  topic: Secure solutions in Azure
  title: Azure App Configuration から構成設定を取得する
  description: Azure App Configuration リソースを作成し、Azure CLI を使用して構成情報を設定する方法について説明します。 次に、**ConfigurationBuilder** を使用してアプリケーションの設定を取得します。
---

# Azure App Configuration から構成設定を取得する

この演習では、Azure App Configuration リソースを作成し、Azure CLI を使用して構成設定を格納し、**ConfigurationBuilder** を使用して構成値を取得する .NET コンソール アプリケーションを構築します。 階層キーを使用して設定を整理し、クラウドベースの構成データにアクセスするためにアプリケーションを認証する方法について説明します。

この演習で実行されるタスク:

* Azure App Configuration リソースを作成する
* 接続文字列の構成情報を保存する
* 構成情報を取得する .NET コンソール アプリを作成する
* リソースをクリーンアップする

この演習の所要時間は約 **15** 分です。

## Azure App Configuration リソースを作成し、構成情報を追加する

演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. Cloud Shell ツール バーの **[設定]** メニューで、**[クラシック バージョンに移動]** を選択します (これはコード エディターを使用するのに必要です)。

1. この演習に必要なリソースのリソース グループを作成します。 使用するリソース グループが既にある場合は、次の手順に進みます。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus** を最寄りのリージョンに置き換えることができます。

    ```
    az group create --name myResourceGroup --location eastus
    ```

1. 一意の名前が必須であり、同じパラメーターを使用するコマンドが多くあります。 変数をいくつか作成すると、リソースを作成するコマンドに必要な変更が減ります。 次のコマンドを実行して、必要な変数を作成します。 **myResourceGroup** を、この演習で使用する名前に置き換えます。 前の手順で場所を変更した場合は、**location** 変数にも同じ変更を加えます。

    ```
    resourceGroup=myResourceGroup
    location=eastus
    appConfigName=appconfigname$RANDOM
    ```

1. 次のコマンドを実行して、App Configuration リソースの名前を取得します。 名前をメモします。演習の後半で必要になります。

    ```
    echo $appConfigName
    ```

1. 次のコマンドを実行して、**Microsoft.AppConfiguration** プロバイダーがサブスクリプションに登録されていることを確認します。

    ```
    az provider register --namespace Microsoft.AppConfiguration
    ```

1. 登録の完了には数分かかる場合があります。 次のコマンドを実行して、登録の状態を確認します。 "**Registered**" という結果が返されたら、次の手順に進みます。

    ```
    az provider show --namespace Microsoft.AppConfiguration --query "registrationState"
    ```

1. 次のコマンドを実行して、Azure App Configuration リソースを作成します。 この実行には数分かかることがあります。

    ```
    az appconfig create --location $location \
        --name $appConfigName \
        --resource-group $resourceGroup
        --sku Free
    ```

    >**ヒント:****Free** SKU 値の使用によるクォータ制限のために AppConfig リソースの作成に問題が発生した場合は、代わりに **Developer** を使用してください。
    

### Microsoft Entra ユーザー名にロールを割り当てる

構成情報を取得するには、Microsoft Entra ユーザーを **App Configuration データ閲覧者**ロールに割り当てる必要があります。 

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。 これは、ロールが割り当てられる対象を表します。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 次のコマンドを実行して、App Configuration サービスのリソース ID を取得します。 リソース ID によって、ロールの割り当てのスコープが決まります。

    ```
    resourceID=$(az appconfig show --resource-group $resourceGroup \
        --name $appConfigName --query id --output tsv)
    ```

1. 次のコマンドを実行して、**App Configuration データ閲覧者**ロールを作成して割り当てます。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "App Configuration Data Reader" \
        --scope $resourceID
    ```

次に、プレースホルダー接続文字列を App Configuration に追加します。

### Azure CLI を使用して構成情報を追加する

Azure App Configuration では、**Dev:conStr** のようなキーは階層型、つまり名前空間型のキーです。 コロン (:) は、論理階層を形成する区切り記号として機能します。

* **Dev** は名前空間または環境プレフィックスを表します (この構成が開発環境用であることを示します)
* **conStr** は構成名を表します

この階層構造により、環境、機能、またはアプリケーション コンポーネントごとに構成設定を整理できるため、関連する設定の管理と取得が容易になります。

次のコマンドを実行して、プレースホルダー接続文字列を保存します。 

```
az appconfig kv set --name $appConfigName \
    --key Dev:conStr \
    --value connectionString \
    --yes
```

このコマンドからは JSON が返されます。 最後の行には値がプレーンテキストで格納されます。 

```json
"value": "connectionString"
```

## 構成情報を取得する .NET コンソール アプリを作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```
    mkdir appconfig
    cd appconfig
    ```

1. .NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、**Azure.Identity** および **Microsoft.Extensions.Configuration.AzureAppConfiguration** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
    ```

### プロジェクトのコードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```
    code Program.cs
    ```

1. 既存の内容を次のコードに置き換えます。 必ず **YOUR_APP_CONFIGURATION_NAME** を先ほどメモした名前に置き換え、コード内のコメントをよく読みます。

    ```csharp
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.Configuration.AzureAppConfiguration;
    using Azure.Identity;
    
    // Set the Azure App Configuration endpoint, replace YOUR_APP_CONFIGURATION_NAME
    // with the name of your actual App Configuration service
    
    string endpoint = "https://YOUR_APP_CONFIGURATION_NAME.azconfig.io"; 
    
    // Configure which authentication methods to use
    // DefaultAzureCredential tries multiple auth methods automatically
    DefaultAzureCredentialOptions credentialOptions = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a configuration builder to combine multiple config sources
    var builder = new ConfigurationBuilder();
    
    // Add Azure App Configuration as a source
    // This connects to Azure and loads configuration values
    builder.AddAzureAppConfiguration(options =>
    {
        
        options.Connect(new Uri(endpoint), new DefaultAzureCredential(credentialOptions));
    });
    
    // Build the final configuration object
    try
    {
        var config = builder.Build();
        
        // Retrieve a configuration value by key name
        Console.WriteLine(config["Dev:conStr"]);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error connecting to Azure App Configuration: {ex.Message}");
    }
    ```

1. **Ctrl + S** キーを押してファイルを保存してから、**Ctrl + Q** キーを押してエディターを終了します。

## Azure にサインインしてアプリを実行する

1. Cloud Shell で次のコマンドを入力して Azure にサインインします。

    ```
    az login
    ```

    **<font color="red">Cloud Shell セッションが既に認証されている場合でも、Azure にサインインする必要があります。</font>**

    > **注**: ほとんどのシナリオでは、*az ログイン*を使用するだけで十分です。 ただし、複数のテナントにサブスクリプションがある場合は、*[--tenant]* パラメーターを使用してテナントを指定する必要があります。 詳細については、「[Azure CLI を使用して対話形式で Azure にサインインする](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)」を参照してください。

1. 次のコマンドを実行して、コンソール アプリを起動します。 アプリには、演習の前半で **Dev:conStr** 設定に割り当てた **connectionString** 値が表示されます。

    ```
    dotnet run
    ```

    アプリには、演習の前半で **Dev:conStr** 設定に割り当てた **connectionString** 値が表示されます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合、その範囲外にある既存のリソースは 
