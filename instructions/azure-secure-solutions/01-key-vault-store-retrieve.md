---
lab:
  topic: Secure solutions in Azure
  title: Azure Key Vault からのシークレットの作成と取得に関するページ
  description: Azure CLI を使用して、またはプログラムで、キー コンテナーを作成し、シークレットを作成および取得する方法について説明します。
---

# Azure Key Vault からのシークレットの作成と取得に関するページ

この演習では、Azure Key Vault を作成し、Azure CLI を使用してシークレットを格納し、キー コンテナーからシークレットを作成および取得できる .NET コンソール アプリケーションを構築します。 認証を構成し、プログラムでシークレットを管理し、完了時にリソースをクリーンアップする方法について説明します。  

この演習で実行されるタスク:

* Azure Key Vault リソースを作成する
* Azure CLI を使用してキー コンテナーにシークレットを保存する
* シークレットを作成および取得する .NET コンソール アプリを作成する
* リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

## Azure Key Vault リソースを作成し、シークレットを追加する

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
    keyVaultName=mykeyvaultname$RANDOM
    ```

1. 次のコマンドを実行してキー コンテナーの名前を取得し、その名前をメモします。 これは、この演習の後半で必要になります。

    ```
    echo $keyVaultName
    ```

1. 次のコマンドを実行して、Azure Key Vault リソースを作成します。 この実行には数分かかることがあります。

    ```
    az keyvault create --name $keyVaultName \
        --resource-group $resourceGroup --location $location
    ```

### Microsoft Entra ユーザー名にロールを割り当てる

シークレットを作成して取得するには、Microsoft Entra ユーザーを**キー コンテナー シークレット責任者**ロールに割り当てます。 こうすることで、シークレットを設定、削除、一覧表示するアクセス許可がユーザー アカウントに付与されます。 一般的なシナリオでは、**キー コンテナー シークレット責任者**を 1 つのグループに割り当て、**キー コンテナー シークレット ユーザー** (シークレットを取得および一覧表示できる) を別のグループに割り当てることで、作成と読み取りのアクションを分けることができます。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。 これは、ロールが割り当てられる対象を表します。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 次のコマンドを実行して、キー コンテナーのリソース ID を取得します。 リソース ID によって、特定のキー コンテナーに対するロールの割り当てのスコープが決まります。

    ```
    resourceID=$(az keyvault show --resource-group $resourceGroup \
        --name $keyVaultName --query id --output tsv)
    ```

1. 次のコマンドを実行して、**キー コンテナー シークレット責任者**ロールを作成して割り当てます。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Key Vault Secrets Officer" \
        --scope $resourceID
    ```

次に、作成したキー コンテナーにシークレットを追加します。

### Azure CLI を使用してシークレットを追加および取得する

1. 次のコマンドを実行して、シークレットを作成します。 

    ```
    az keyvault secret set --vault-name $keyVaultName \
        --name "MySecret" --value "My secret value"
    ```

1. 次のコマンドを実行してシークレットを取得し、設定されていることを確認します。

    ```
    az keyvault secret show --name "MySecret" --vault-name $keyVaultName
    ```

    このコマンドからは JSON が返されます。 最後の行にパスワードがプレーン テキストで格納されます。 

    ```json
    "value": "My secret value"
    ```

## シークレットを保存および取得する .NET コンソール アプリを作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```
    mkdir keyvault
    cd keyvault
    ```

1. .NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、**Azure.Identity** および **Azure.Security.KeyVault.Secrets** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Azure.Identity
    dotnet add package Azure.Security.KeyVault.Secrets
    ```

### プロジェクトのスタート コードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```
    code Program.cs
    ```

1. 既存の内容を次のコードに置き換えます。 必ず **YOUR-KEYVAULT-NAME** を実際のキー コンテナー名に置き換えてください。

    ```csharp
    using Azure.Identity;
    using Azure.Security.KeyVault.Secrets;
    
    // Replace YOUR-KEYVAULT-NAME with your actual Key Vault name
    string KeyVaultUrl = "https://YOUR-KEYVAULT-NAME.vault.azure.net/";
    
    
    // ADD CODE TO CREATE A CLIENT
    
    
    
    // ADD CODE TO CREATE A MENU SYSTEM
    
    
    
    // ADD CODE TO CREATE A SECRET
    
    
    
    // ADD CODE TO LIST SECRETS
    
    
    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

### コードを追加してアプリケーションを完成させます

次に、コードを追加してアプリケーションを完成させます。

1. **// ADD CODE TO CREATE A CLIENT** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Configure authentication options for connecting to Azure Key Vault
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create the Key Vault client using the URL and authentication credentials
    var client = new SecretClient(new Uri(KeyVaultUrl), new DefaultAzureCredential(options));
    ```

1. **// ADD CODE TO CREATE A MENU SYSTEM** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Main application loop - continues until user types 'quit'
    while (true)
    {
        // Display menu options to the user
        Console.Clear();
        Console.WriteLine("\nPlease select an option:");
        Console.WriteLine("1. Create a new secret");
        Console.WriteLine("2. List all secrets");
        Console.WriteLine("Type 'quit' to exit");
        Console.Write("Enter your choice: ");
    
        // Read user input and convert to lowercase for easier comparison
        string? input = Console.ReadLine()?.Trim().ToLower();
        
        // Check if user wants to exit the application
        if (input == "quit")
        {
            Console.WriteLine("Goodbye!");
            break;
        }
    
        // Process the user's menu selection
        switch (input)
        {
            case "1":
                // Call the method to create a new secret
                await CreateSecretAsync(client);
                break;
            case "2":
                // Call the method to list all existing secrets
                await ListSecretsAsync(client);
                break;
            default:
                // Handle invalid input
                Console.WriteLine("Invalid option. Please enter 1, 2, or 'quit'.");
                break;
        }
    }
    ```

1. **// ADD CODE TO CREATE A SECRET** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    async Task CreateSecretAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("\nCreating a new secret...");
            
            // Get the secret name from user input
            Console.Write("Enter secret name: ");
            string? secretName = Console.ReadLine()?.Trim();
    
            // Validate that the secret name is not empty
            if (string.IsNullOrEmpty(secretName))
            {
                Console.WriteLine("Secret name cannot be empty.");
                return;
            }
            
            // Get the secret value from user input
            Console.Write("Enter secret value: ");
            string? secretValue = Console.ReadLine()?.Trim();
    
            // Validate that the secret value is not empty
            if (string.IsNullOrEmpty(secretValue))
            {
                Console.WriteLine("Secret value cannot be empty.");
                return;
            }
    
            // Create a new KeyVaultSecret object with the provided name and value
            var secret = new KeyVaultSecret(secretName, secretValue);
            
            // Store the secret in Azure Key Vault
            await client.SetSecretAsync(secret);
    
            Console.WriteLine($"Secret '{secretName}' created successfully!");
            Console.WriteLine("Press Enter to continue...");
            Console.ReadLine();
        }
        catch (Exception ex)
        {
            // Handle any errors that occur during secret creation
            Console.WriteLine($"Error creating secret: {ex.Message}");
        }
    }
    ```

1. **// ADD CODE TO LIST SECRETS** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    async Task ListSecretsAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("Listing all secrets in the Key Vault...");
            Console.WriteLine("----------------------------------------");
    
            // Get an async enumerable of all secret properties in the Key Vault
            var secretProperties = client.GetPropertiesOfSecretsAsync();
            bool hasSecrets = false;
    
            // Iterate through each secret property to retrieve full secret details
            await foreach (var secretProperty in secretProperties)
            {
                hasSecrets = true;
                try
                {
                    // Retrieve the actual secret value and metadata using the secret name
                    var secret = await client.GetSecretAsync(secretProperty.Name);
                    
                    // Display the secret information to the console
                    Console.WriteLine($"Name: {secret.Value.Name}");
                    Console.WriteLine($"Value: {secret.Value.Value}");
                    Console.WriteLine($"Created: {secret.Value.Properties.CreatedOn}");
                    Console.WriteLine("----------------------------------------");
                }
                catch (Exception ex)
                {
                    // Handle errors for individual secrets (e.g., access denied, secret not found)
                    Console.WriteLine($"Error retrieving secret '{secretProperty.Name}': {ex.Message}");
                    Console.WriteLine("----------------------------------------");
                }
            }
    
            // Inform user if no secrets were found in the Key Vault
            if (!hasSecrets)
            {
                Console.WriteLine("No secrets found in the Key Vault.");
            }
        }
        catch (Exception ex)
        {
            // Handle general errors that occur during the listing operation
            Console.WriteLine($"Error listing secrets: {ex.Message}");
        
        }
        Console.WriteLine("Press Enter to continue...");
        Console.ReadLine();
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

1. 次のコマンドを実行して、コンソール アプリを起動します。 アプリにアプリケーションのメニュー システムが表示されます。 

    ```
    dotnet run
    ```

1. この演習の冒頭でシークレットを作成しました。それを取得し、表示するには、「**2**」と入力します。

1. シークレットの名前と値を入力し、新しいシークレットを作成するには、「**1**」と入力します。

1. シークレットをもう一度一覧表示して、新しく追加されたシークレットを確認します。

アプリケーションを完了するときは、「**quit**」と入力します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合、その範囲外にある既存のリソースは 
