---
lab:
  topic: Azure Cosmos DB
  title: .NET を使用して Azure Cosmos DB for NoSQL にリソースを作成する
  description: Microsoft .NET SDK v3 を使用して Azure Cosmos DB 内でデータベース リソースとコンテナー リソースを作成する方法について説明します。
---

# .NET を使用して Azure Cosmos DB for NoSQL にリソースを作成する

この演習では、Azure Cosmos DB アカウントを作成し、Microsoft Azure Cosmos DB SDK を使用してデータベース、コンテナー、サンプル項目を作成する .NET コンソール アプリケーションを構築します。 認証を構成し、プログラムでデータベース操作を実行し、Azure portal で結果を確認する方法について説明します。

この演習で実行されるタスク:

* Azure Cosmos DB アカウントを作成する
* データベース、コンテナー、項目を作成するコンソール アプリを作成する
* コンソール アプリを実行して結果を確認する

この演習の所要時間は約 **30** 分です。

## Azure Cosmos DB アカウントを作成する

演習のこのセクションでは、リソース グループと Azure Cosmos DB アカウントを作成します。 また、エンドポイントとアカウント用アクセス キーも記録します。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. Cloud Shell ツール バーの **[設定]** メニューで、**[クラシック バージョンに移動]** を選択します (これはコード エディターを使用するのに必要です)。

1. この演習に必要なリソースのリソース グループを作成します。 使用するリソース グループが既にある場合は、次の手順に進みます。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus** を最寄りのリージョンに置き換えることができます。

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. 一意の名前が必須であり、同じパラメーターを使用するコマンドが多くあります。 変数をいくつか作成すると、リソースを作成するコマンドに必要な変更が減ります。 次のコマンドを実行して、必要な変数を作成します。 **myResourceGroup** を、この演習で使用する名前に置き換えます。

    ```
    resourceGroup=myResourceGroup
    accountName=cosmosexercise$RANDOM
    ```

1. 次のコマンドを実行して Azure Cosmos DB アカウントを作成します。各アカウント名は一意にする必要があります。 

    ```
    az cosmosdb create --name $accountName \
        --resource-group $resourceGroup
    ```

1.  次のコマンドを実行して、Azure Cosmos DB アカウントの **documentEndpoint** を取得します。 コマンド結果に表示されるエンドポイントを記録しておきます。これは演習の後半で必要になります。

    ```
    az cosmosdb show --name $accountName \
        --resource-group $resourceGroup \
        --query "documentEndpoint" --output tsv
    ```

1. 次のコマンドを使用して、アカウントの主キーを取得します。 コマンド結果の主キーをメモしておきます。これは、演習の後半で必要になります。

    ```
    az cosmosdb keys list --name $accountName \
        --resource-group $resourceGroup \
        --query "primaryMasterKey" --output tsv
    ```

## .NET コンソール アプリケーションを使用してデータ リソースと項目を作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. プロジェクト用のフォルダーを作成し、フォルダーに移動します。

    ```bash
    mkdir cosmosdb
    cd cosmosdb
    ```

1. .NET コンソール アプリを作成します。

    ```bash
    dotnet new console
    ```

### コンソール アプリケーションを構成する

1. 次のコマンドを実行して、**Microsoft.Azure.Cosmos**、**Newtonsoft.Json**、**dotenv.net** パッケージをプロジェクトに追加します。

    ```bash
    dotnet add package Microsoft.Azure.Cosmos --version 3.*
    dotnet add package Newtonsoft.Json --version 13.*
    dotnet add package dotenv.net
    ```

1. 次のコマンドを実行して、シークレットを保持する **.env** ファイルを作成し、コード エディターで開きます。

    ```bash
    touch .env
    code .env
    ```

1. 次のコードを **.env** ファイルに追加します。 **YOUR_DOCUMENT_ENDPOINT** と **YOUR_ACCOUNT_KEY** を、先ほどメモした値に置き換えます。

    ```
    DOCUMENT_ENDPOINT="YOUR_DOCUMENT_ENDPOINT"
    ACCOUNT_KEY="YOUR_ACCOUNT_KEY"
    ```

1. **Ctrl + S** キーを押してファイルを保存してから、**Ctrl + Q** キーを押してエディターを終了します。

次に、クラウド シェル内のエディターを使用して、**Program.cs** ファイル内のテンプレート コードを置き換えます。

### プロジェクトの開始コードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```bash
    code Program.cs
    ```

1. 既存のコードを次のコード スニペットに置き換えます。 

    このコードは、アプリ全体の構造を示しています。 コード内のコメントを確認して、しくみを理解してください。 アプリケーションを完成させるには、演習の後半で、指定された領域にコードを追加します。 

    ```csharp
    using Microsoft.Azure.Cosmos;
    using dotenv.net;
    
    string databaseName = "myDatabase"; // Name of the database to create or use
    string containerName = "myContainer"; // Name of the container to create or use
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    string cosmosDbAccountUrl = envVars["DOCUMENT_ENDPOINT"];
    string accountKey = envVars["ACCOUNT_KEY"];
    
    if (string.IsNullOrEmpty(cosmosDbAccountUrl) || string.IsNullOrEmpty(accountKey))
    {
        Console.WriteLine("Please set the DOCUMENT_ENDPOINT and ACCOUNT_KEY environment variables.");
        return;
    }
    
    // CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY
    
    
    try
    {
        // CREATE A DATABASE IF IT DOESN'T ALREADY EXIST
    
    
        // CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY
    
    
        // DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER
    
    
        // ADD THE ITEM TO THE CONTAINER
    
    
    }
    catch (CosmosException ex)
    {
        // Handle Cosmos DB-specific exceptions
        // Log the status code and error message for debugging
        Console.WriteLine($"Cosmos DB Error: {ex.StatusCode} - {ex.Message}");
    }
    catch (Exception ex)
    {
        // Handle general exceptions
        // Log the error message for debugging
        Console.WriteLine($"Error: {ex.Message}");
    }
    
    // This class represents a product in the Cosmos DB container
    public class Product
    {
        public string? id { get; set; }
        public string? name { get; set; }
        public string? description { get; set; }
    }
    ```

次に、プロジェクトの指定された領域にコードを追加して、クライアント、データベース、コンテナーを作成し、コンテナーにサンプル項目を追加します。

### コードの追加によりクライアントを作成して操作を実行する 

1. **// CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY** コメントの後のスペースに次のコードを追加します。 このコードにより、お使いの Azure Cosmos DB アカウントへの接続に使用するクライアントを定義します。

    ```csharp
    CosmosClient client = new(
        accountEndpoint: cosmosDbAccountUrl,
        authKeyOrResourceToken: accountKey
    );
    ```

    >注: *Azure Identity* ライブラリの **DefaultAzureCredential** を使用することをお勧めします。 サブスクリプションの設定方法によっては、Azure で追加の構成要件が必要になる場合があります。 

1. **// CREATE A DATABASE IF IT DOESN'T ALREADY EXIST** コメントの後のスペースに次のコードを追加します。 

    ```csharp
    Database database = await client.CreateDatabaseIfNotExistsAsync(databaseName);
    Console.WriteLine($"Created or retrieved database: {database.Id}");
    ```

1. **// CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY** コメントの後のスペースに次のコードを追加します。 

    ```csharp
    Container container = await database.CreateContainerIfNotExistsAsync(
        id: containerName,
        partitionKeyPath: "/id"
    );
    Console.WriteLine($"Created or retrieved container: {container.Id}");
    ```

1. **// DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER** コメントの後のスペースに次のコードを追加します。 これにより、コンテナーに追加される項目を定義します。

    ```csharp
    Product newItem = new Product
    {
        id = Guid.NewGuid().ToString(), // Generate a unique ID for the product
        name = "Sample Item",
        description = "This is a sample item in my Azure Cosmos DB exercise."
    };
    ```

1. **// ADD THE ITEM TO THE CONTAINER** コメントの後のスペースに次のコードを追加します。 

    ```csharp
    ItemResponse<Product> createResponse = await container.CreateItemAsync(
        item: newItem,
        partitionKey: new PartitionKey(newItem.id)
    );

    Console.WriteLine($"Created item with ID: {createResponse.Resource.id}");
    Console.WriteLine($"Request charge: {createResponse.RequestCharge} RUs");
    ```

1. コードが完成したので、進行状況を保存します。**Ctrl + S** キーを押してファイルを保存し、**Ctrl + Q** キーを押してエディターを終了します。

1. クラウド シェル内で次のコマンドを実行して、プロジェクト内にエラーがないかテストします。 エラーが表示された場合は、エディターで *Program.cs* ファイルを開き、欠損コードや貼り付けエラーがないか確認します。

    ```
    dotnet build
    ```

プロジェクトが完了したら、アプリケーションを実行し、Azure portal で結果を確認します。

## アプリケーションの実行と結果の確認

1. クラウド シェル内の場合には、`dotnet run` コマンドを実行します。 次のような内容が出力されます。

    ```
    Created or retrieved database: myDatabase
    Created or retrieved container: myContainer
    Created item: c549c3fa-054d-40db-a42b-c05deabbc4a6
    Request charge: 6.29 RUs
    ```

1. Azure portal 内で、前の手順に作成した Azure Cosmos DB アカウントに移動します。 左側のナビゲーションで、**[Data Explorer]** を選択します。 **Data Explorer** で **[myDatabase]** を選択し、次に **[myContainer]** を展開します。 作成した項目を表示するには、**[Items]** を選択します。

    ![データ エクスプローラー内の項目の位置を示すスクリーンショット。](./media/01/cosmos-data-explorer.png)

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。
