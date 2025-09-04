---
lab:
  topic: Azure Storage
  title: .NET クライアント ライブラリを使用して BLOB ストレージ リソースを作成する
  description: Azure Storage .NET クライアント ライブラリを使用してコンテナーを作成し、BLOB をアップロードして一覧表示し、コンテナーを削除する方法について説明します。
---

# .NET クライアント ライブラリを使用して BLOB ストレージ リソースを作成する

この演習では、Azure Storage アカウントを作成し、Azure Storage Blob クライアント ライブラリを使用して .NET コンソール アプリケーションを構築し、コンテナーの作成、BLOB ストレージへのファイルのアップロード、BLOB の一覧表示、ファイルのダウンロードを行います。 Azure の認証を受け、プログラムで BLOB ストレージ操作を実行し、Azure portal で結果を確認する方法について説明します。

この演習で実行されるタスク:

* Azure リソースを準備する
* データの作成とダウンロードを行うコンソール アプリを作成する
* アプリを実行して結果を確認する
* リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

## Azure ストレージ アカウントの作成

演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. Cloud Shell ツール バーの **[設定]** メニューで、**[クラシック バージョンに移動]** を選択します (これはコード エディターを使用するのに必要です)。

1. この演習に必要なリソースのリソース グループを作成します。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus2** を最寄りのリージョンに置き換えることができます。 使用するリソース グループが既にある場合は、次の手順に進みます。

    ```
    az group create --location eastus2 --name myResourceGroup
    ```

1. 一意の名前が必須であり、同じパラメーターを使用するコマンドが多くあります。 変数をいくつか作成すると、リソースを作成するコマンドに必要な変更が減ります。 次のコマンドを実行して、必要な変数を作成します。 **myResourceGroup** を、この演習で使用する名前に置き換えます。

    ```
    resourceGroup=myResourceGroup
    location=eastus
    accountName=storageacct$RANDOM
    ```

1. 次のコマンドを実行して Azure Storage アカウントを作成します。各アカウント名は一意にする必要があります。 最初のコマンドで、ストレージ アカウント用の一意の名前を持つ変数を作成します。 **echo** コマンドの出力にあるアカウント名をメモします。 

    ```
    az storage account create --name $accountName \
        --resource-group $resourceGroup \
        --location $location \
        --sku Standard_LRS 
    
    echo $accountName
    ```

### Microsoft Entra ユーザー名にロールを割り当てる

アプリでリソースと項目を作成できるようにするには、Microsoft Entra ユーザーを**ストレージ BLOB データ所有者**ロールに割り当てます。 Cloud Shell で次の手順を実行します。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。 これは、ロールが割り当てられる対象を表します。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 次のコマンドを実行して、ストレージ アカウントのリソース ID を取得します。 リソース ID によって、特定の名前空間に対するロールの割り当てのスコープが決まります。

    ```
    resourceID=$(az storage account show --name $accountName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. 次のコマンドを実行して、**ストレージ BLOB データ所有者**ロールを作成して割り当てます。 このロールにより、コンテナーと項目を管理するアクセス許可が付与されます。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Blob Data Owner" \
        --scope $resourceID
    ```

## コンテナーと項目を作成する .NET コンソール アプリを作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```
    mkdir azstor
    cd azstor
    ```

1. .NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、アプリケーションに必要なパッケージを追加します。

    ```
    dotnet add package Azure.Storage.Blobs
    dotnet add package Azure.Identity
    ```

1. 次のコマンドを実行して、プロジェクトに **data** フォルダーを作成します。 

    ```
    mkdir data
    ```

次に、プロジェクトのコードを追加します。

### プロジェクトのスタート コードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```
    code Program.cs
    ```

1. 既存の内容を次のコードに置き換えます。 コード内のコメントを必ず確認してください。

    ```csharp
    using Azure.Storage.Blobs;
    using Azure.Storage.Blobs.Models;
    using Azure.Identity;
    
    Console.WriteLine("Azure Blob Storage exercise\n");
    
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Run the examples asynchronously, wait for the results before proceeding
    await ProcessAsync();
    
    Console.WriteLine("\nPress enter to exit the sample application.");
    Console.ReadLine();
    
    async Task ProcessAsync()
    {
        // CREATE A BLOB STORAGE CLIENT
        
    
    
        // CREATE A CONTAINER
        
    
    
        // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
        
    
    
        // UPLOAD THE FILE TO BLOB STORAGE
        
    
    
        // LIST BLOBS IN THE CONTAINER
        
    
    
        // DOWNLOAD THE BLOB TO A LOCAL FILE
        
    
    }
    ```

1. **Ctrl + S** キーを押して変更を保存し、次の手順に進みます。


## コードを追加してプロジェクトを完成させます

演習の残りの部分では、指定された領域にコードを追加して、完全なアプリケーションを作成します。 

1. **// CREATE A BLOB STORAGE CLIENT** コメントを見つけて、そのコメントの直後に次のコードを追加します。 **BlobServiceClient** は、ストレージ アカウント内のコンテナーと BLOB を管理する主要なエントリ ポイントとして機能します。 クライアントは認証に *DefaultAzureCredential* を使用します。 **YOUR_ACCOUNT_NAME** を、先ほどメモした名前に置き換えます。

    ```csharp
    // Create a credential using DefaultAzureCredential with configured options
    string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name
    
    // Use the DefaultAzureCredential with the options configured at the top of the program
    DefaultAzureCredential credential = new DefaultAzureCredential(options);
    
    // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
    string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
    BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
    ```

1. **Ctrl + S** キーを押して変更を保存し、次の手順に進みます。

1. **// CREATE A CONTAINER** コメントを見つけて、そのコメントの直後に次のコードを追加します。 コンテナーを作成するには、**BlobServiceClient** クラスのインスタンスを作成し、**CreateBlobContainerAsync** メソッドを呼び出してストレージ アカウントにコンテナーを作成します。 コンテナー名の後にはそれを一意にするために GUID 値が追加されます。 コンテナーが既に存在する場合、**CreateBlobContainerAsync** メソッドは失敗します。

    ```csharp
    // Create a unique name for the container
    string containerName = "wtblob" + Guid.NewGuid().ToString();
    
    // Create the container and return a container client object
    Console.WriteLine("Creating container: " + containerName);
    BlobContainerClient containerClient = 
        await blobServiceClient.CreateBlobContainerAsync(containerName);
    
    // Check if the container was created successfully
    if (containerClient != null)
    {
        Console.WriteLine("Container created successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("Failed to create the container, exiting program.");
        return;
    }
    ```

1. **Ctrl + S** キーを押して変更を保存し、次の手順に進みます。

1. **// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE** コメントを見つけて、そのコメントの直後に次のコードを追加します。 これにより、データ ディレクトリにファイルが作成され、それがコンテナーにアップロードされます。

    ```csharp
    // Create a local file in the ./data/ directory for uploading and downloading
    Console.WriteLine("Creating a local file for upload to Blob storage...");
    string localPath = "./data/";
    string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
    string localFilePath = Path.Combine(localPath, fileName);
    
    // Write text to the file
    await File.WriteAllTextAsync(localFilePath, "Hello, World!");
    Console.WriteLine("Local file created, press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. **Ctrl + S** キーを押して変更を保存し、次の手順に進みます。

1. **// UPLOAD THE FILE TO BLOB STORAGE** コメントを見つけて、そのコメントの直後に次のコードを追加します。 このコードを使用して、前のセクションで作成したコンテナーの **GetBlobClient** メソッドを呼び出して、**BlobClient** オブジェクトへの参照を取得します。 次に、**UploadAsync** メソッドを使用して、生成されたローカル ファイルをアップロードします。 このメソッドは、BLOB がまだ存在しない場合は作成し、既に存在する場合は上書きします。

    ```csharp
    // Get a reference to the blob and upload the file
    BlobClient blobClient = containerClient.GetBlobClient(fileName);
    
    Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);
    
    // Open the file and upload its data
    using (FileStream uploadFileStream = File.OpenRead(localFilePath))
    {
        await blobClient.UploadAsync(uploadFileStream);
        uploadFileStream.Close();
    }
    
    // Verify if the file was uploaded successfully
    bool blobExists = await blobClient.ExistsAsync();
    if (blobExists)
    {
        Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("File upload failed, exiting program..");
        return;
    }
    ```

1. **Ctrl + S** キーを押して変更を保存し、次の手順に進みます。

1. **// LIST BLOBS IN THE CONTAINER** コメントを見つけて、そのコメントの直後に次のコードを追加します。 **GetBlobsAsync** メソッドを使用してコンテナー内の BLOB を一覧表示します。 この場合、コンテナーに追加された BLOB は 1 つだけであるため、リスト操作で返されるのはその BLOB 1 つだけです。 

    ```csharp
    Console.WriteLine("Listing blobs in container...");
    await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
    {
        Console.WriteLine("\t" + blobItem.Name);
    }
    
    Console.WriteLine("Press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. **Ctrl + S** キーを押して変更を保存し、次の手順に進みます。

1. **// DOWNLOAD THE BLOB TO A LOCAL FILE** コメントを見つけて、そのコメントの直後に次のコードを追加します。 このコードでは、**DownloadAsync** メソッドを使用して、先ほど作成した BLOB をローカル ファイル システムにダウンロードします。 このコード例では、ローカル ファイル システムで両方のファイルを表示できるように、BLOB 名に "DOWNLOADED" というサフィックスを追加します。 

    ```csharp
    // Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
    // overwrite the original file
    
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
    
    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);
    
    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();
    
    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }
    
    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
    ```

1. **Ctrl + S** キーを押してファイルを保存してから、**Ctrl + Q** キーを押してエディターを終了します。

## Azure にサインインしてアプリを実行する

1. Cloud Shell コマンド ライン ペインで、次のコマンドを入力して Azure にサインインします。

    ```
    az login
    ```

    **<font color="red">Cloud Shell セッションが既に認証されている場合でも、Azure にサインインする必要があります。</font>**

    > **注**: ほとんどのシナリオでは、*az ログイン*を使用するだけで十分です。 ただし、複数のテナントにサブスクリプションがある場合は、*[--tenant]* パラメーターを使用してテナントを指定する必要があります。 詳細については、「[Azure CLI を使用して対話形式で Azure にサインインする](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)」を参照してください。

1. 次のコマンドを実行して、コンソール アプリを起動します。 アプリは実行中に何度も一時停止し、任意のキー入力を待機し、処理を続行します。 これにより、Azure portal のメッセージを確認できます。

    ```
    dotnet run
    ```

1. Azure portal で、作成した Azure Storage アカウントに移動します。 

1. 左側のナビゲーションの **[> データ ストレージ]** を展開し、**[コンテナー]** を選択します。

1. アプリケーションによって作成されたコンテナーを選択します。アップロードされた BLOB を確認できます。

1. 以下の 2 つのコマンドを実行して、**data** ディレクトリに移動し、アップロードおよびダウンロードされたファイルを一覧表示します。

    ```
    cd data
    ls
    ```

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合、その範囲外にある既存のリソースは 

