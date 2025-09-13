---
lab:
  topic: Azure events and messaging
  title: Azure Event Hubs からイベントを送信および取得する
  description: .NET Azure.Messaging.EventHubs SDK を使用して Azure Event Hubs からイベントを送信および取得する方法について説明します。
---

# Azure Event Hubs からイベントを送信および取得する

この演習では、Azure Event Hubs リソースを作成し、**Azure.Messaging.EventHubs** SDK を使用してイベントを送受信する .NET コンソール アプリを構築します。 クラウド リソースをプロビジョニングし、Event Hubs を操作し、完了したら環境をクリーンアップする方法について説明します。

この演習で実行されるタスク:

* リソース グループを作成する
* Azure Event Hubs リソースを作成する
* イベントを送信および取得する .NET コンソール アプリを作成する
* リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

## Azure Event Hubs リソースを作成する

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
    namespaceName=eventhubsns$RANDOM
    ```

### Azure Event Hubs 名前空間とイベント ハブを作成する

Azure Event Hubs 名前空間は、Azure 内のイベント ハブ リソースの論理コンテナーです。 1 つ以上のイベント ハブを作成できる固有のスコープを持つコンテナーとして機能し、大量のイベント データの取り込み、処理、保存に使用されます。 以下の手順は Cloud Shell で実行されます。 

1. 次のコマンドを実行して、Event Hubs 名前空間を作成します。

    ```
    az eventhubs namespace create --name $namespaceName --resource-group $resourceGroup -l $location
    ```

1. 次のコマンドを実行して、Event Hubs 名前空間に **myEventHub** というイベント ハブを作成します。 

    ```
    az eventhubs eventhub create --name myEventHub --resource-group $resourceGroup \
      --namespace-name $namespaceName
    ```

### Microsoft Entra ユーザー名にロールを割り当てる

アプリがメッセージを送受信できるようにするには、Microsoft Entra ユーザーを Event Hubs 名前空間レベルで **Azure Event Hubs データ所有者**ロールに割り当てます。 これにより、Azure RBAC を使用してキューとトピックを管理およびアクセスするアクセス許可がユーザー アカウントに付与されます。 Cloud Shell で次の手順を実行します。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。 これは、ロールが割り当てられる対象を表します。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 次のコマンドを実行して、Event Hubs 名前空間のリソース ID を取得します。 リソース ID によって、特定の名前空間に対するロールの割り当てのスコープが決まります。

    ```
    resourceID=$(az eventhubs namespace show --resource-group $resourceGroup \
        --name $namespaceName --query id --output tsv)
    ```
1. 次のコマンドを実行して、**Azure Event Hubs データ所有者**ロールを作成して割り当てます。これにより、イベントの送信と取得のアクセス許可が付与されます。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Event Hubs Data Owner" \
        --scope $resourceID
    ```

## .NET コンソール アプリケーションでイベントを送信および取得する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```
    mkdir eventhubs
    cd eventhubs
    ```

1. .NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、**Azure.Messaging.EventHubs** および **Azure.Identity** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Azure.Messaging.EventHubs
    dotnet add package Azure.Identity
    ```

次に、クラウド シェル内のエディターを使用して、**Program.cs** ファイル内のテンプレート コードを置き換えます。

### プロジェクトのスタート コードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```
    code Program.cs
    ```

1. 既存の内容を次のコードに置き換えます。 必ずコード内のコメントを確認し、**YOUR_EVENT_HUB_NAMESPACE** をイベント ハブの名前空間に置き換えます。

    ```csharp
    using Azure.Messaging.EventHubs;
    using Azure.Messaging.EventHubs.Producer;
    using Azure.Messaging.EventHubs.Consumer;
    using Azure.Identity;
    using System.Text;
    
    // TO-DO: Replace YOUR_EVENT_HUB_NAMESPACE with your actual Event Hub namespace
    string namespaceURL = "YOUR_EVENT_HUB_NAMESPACE.servicebus.windows.net";
    string eventHubName = "myEventHub"; 
    
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Number of events to be sent to the event hub
    int numOfEvents = 3;
    
    // CREATE A PRODUCER CLIENT AND SEND EVENTS
    
    
    
    // CREATE A CONSUMER CLIENT AND RECEIVE EVENTS
    
    
    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

### コードを追加してアプリケーションを完成させます

このセクションでは、イベントを送受信するプロデューサー クライアントとコンシューマー クライアントを作成するコードを追加します。

1. **// CREATE A PRODUCER CLIENT AND SEND EVENTS** コメントを見つけて、コメントの直後に次のコードを追加します。 コード内のコメントを必ず確認してください。

    ```csharp
    // Create a producer client to send events to the event hub
    EventHubProducerClient producerClient = new EventHubProducerClient(
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    // Create a batch of events 
    using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
    
    
    // Adding a random number to the event body and sending the events. 
    var random = new Random();
    for (int i = 1; i <= numOfEvents; i++)
    {
        int randomNumber = random.Next(1, 101); // 1 to 100 inclusive
        string eventBody = $"Event {randomNumber}";
        if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes(eventBody))))
        {
            // if it is too large for the batch
            throw new Exception($"Event {i} is too large for the batch and cannot be sent.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of events to the event hub
        await producerClient.SendAsync(eventBatch);
    
        Console.WriteLine($"A batch of {numOfEvents} events has been published.");
        Console.WriteLine("Press Enter to retrieve and print the events...");
        Console.ReadLine();
    }
    finally
    {
        await producerClient.DisposeAsync();
    }
    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

1. **// CREATE A CONSUMER CLIENT AND RETRIEVE EVENTS** コメントを見つけて、コメントの直後に次のコードを追加します。 コード内のコメントを必ず確認してください。

    ```csharp
    // Create an EventHubConsumerClient
    await using var consumerClient = new EventHubConsumerClient(
        EventHubConsumerClient.DefaultConsumerGroupName,
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    Console.Clear();
    Console.WriteLine("Retrieving all events from the hub...");
    
    // Get total number of events in the hub by summing (last - first + 1) for all partitions
    // This count is used to determine when to stop reading events
    long totalEventCount = 0;
    string[] partitionIds = await consumerClient.GetPartitionIdsAsync();
    foreach (var partitionId in partitionIds)
    {
        PartitionProperties properties = await consumerClient.GetPartitionPropertiesAsync(partitionId);
        if (!properties.IsEmpty && properties.LastEnqueuedSequenceNumber >= properties.BeginningSequenceNumber)
        {
            totalEventCount += (properties.LastEnqueuedSequenceNumber - properties.BeginningSequenceNumber + 1);
        }
    }
    
    // Start retrieving events from the event hub and print to the console
    int retrievedCount = 0;
    await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync(startReadingAtEarliestEvent: true))
    {
        if (partitionEvent.Data != null)
        {
            string body = Encoding.UTF8.GetString(partitionEvent.Data.Body.ToArray());
            Console.WriteLine($"Retrieved event: {body}");
            retrievedCount++;
            if (retrievedCount >= totalEventCount)
            {
                Console.WriteLine("Done retrieving events. Press Enter to exit...");
                Console.ReadLine();
                return;
            }
        }
    }
    ```

1. **Ctrl + S** キーを押してファイルを保存してから、**Ctrl + Q** キーを押してエディターを終了します。

## Azure にサインインしてアプリを実行する

1. Cloud Shell コマンド ライン ペインで、次のコマンドを入力して Azure にサインインします。

    ```
    az login
    ```

    **<font color="red">Cloud Shell セッションが既に認証されている場合でも、Azure にサインインする必要があります。</font>**

    > **注**: ほとんどのシナリオでは、*az ログイン*を使用するだけで十分です。 ただし、複数のテナントにサブスクリプションがある場合は、*[--tenant]* パラメーターを使用してテナントを指定する必要があります。 詳細については、「[Azure CLI を使用して対話形式で Azure にサインインする](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)」を参照してください。

1. 次のコマンドを実行してアプリケーションを起動します。

    ```
    dotnet run
    ```

    数秒後、次の例のような出力が表示されます。
    
    ```
    A batch of 3 events has been published.
    Press Enter to retrieve and print the events...
    
    Retrieving all events from the hub...
    Retrieved event: Event 4
    Retrieved event: Event 96
    Retrieved event: Event 74
    Done retrieving events. Press Enter to exit...
    ```

アプリケーションからハブには常に 3 つのイベントが送信されますが、ハブ内のすべてのイベントが取得されます。 アプリケーションを複数回実行すると、取得されるイベント数が増えます。 イベント作成に使用される乱数は、さまざまなイベントを特定するのに役立ちます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。 
