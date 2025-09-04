---
lab:
  topic: Azure events and messaging
  title: Azure Service Bus からメッセージを送受信する
  description: .NET Azure.Messaging.ServiceBus SDK を使用して Azure Service Bus からメッセージを送信する方法について説明します。
---

# Azure Service Bus からメッセージを送受信する

この演習では、Azure Service Bus リソースを作成して構成し、**Azure.Messaging.ServiceBus SDK** を使用してメッセージを送受信する .NET アプリを構築します。 Service Bus 名前空間とキューをプロビジョニングし、アクセス許可を割り当て、プログラムでメッセージを操作する方法について説明します。 

この演習で実行されるタスク:

* Azure Service Bus リソースを作成する
* Microsoft Entra ユーザー名にロールを割り当てる
* メッセージを送受信する .NET コンソール アプリを作成する
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
    namespaceName=svcbusns$RANDOM
    ```

1. この演習の後半で、この名前空間に割り当てた名前が必要になります。 次のコマンドを実行し、出力をメモします。

    ```
    echo $namespaceName
    ```

### Azure Service Bus の名前空間とキューを作成する

1. Service Bus メッセージング名前空間を作成します。 次のコマンドによって、前の手順で作成した変数を使用して名前空間が作成されます。 この操作は、完了するまでに数分かかります。

    ```bash
    az servicebus namespace create \
        --resource-group $resourceGroup \
        --name $namespaceName \
        --location $location
    ```

1. 名前空間が作成されたので、メッセージを保持するキューを作成する必要があります。 次のコマンドを実行して、**myqueue** というキューを作成します。

    ```bash
    az servicebus queue create --resource-group $resourceGroup \
        --namespace-name $namespaceName \
        --name myqueue
    ```

### Microsoft Entra ユーザー名にロールを割り当てる

アプリがメッセージを送受信できるようにするには、Service Bus 名前空間レベルで Microsoft Entra ユーザーを **Azure Service Bus データ所有者**ロールに割り当てます。 これにより、Azure RBAC を使用してキューとトピックを管理およびアクセスするアクセス許可がユーザー アカウントに付与されます。 Cloud Shell で次の手順を実行します。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。 これは、ロールが割り当てられる対象を表します。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 次のコマンドを実行して、Service Bus 名前空間のリソース ID を取得します。 リソース ID によって、特定の名前空間に対するロールの割り当てのスコープが決まります。

    ```
    resourceID=$(az servicebus namespace show --name $namespaceName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. 次のコマンドを実行して、**Azure Service Bus データ所有者**ロールを作成して割り当てます。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Service Bus Data Owner" \
        --scope $resourceID
    ```

## メッセージを送受信する .NET コンソール アプリを作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```
    mkdir svcbus
    cd svcbus
    ```

1. .NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、**Azure.Messaging.ServiceBus** および **Azure.Identity** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Azure.Messaging.ServiceBus
    dotnet add package Azure.Identity
    ```

### プロジェクトのスタート コードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```
    code Program.cs
    ```

1. 既存の内容を次のコードに置き換えます。 必ずコード内のコメントを確認し、**<YOUR-NAMESPACE>** を先ほどメモした Service Bus 名前空間に置き換えます。

    ```csharp
    using Azure.Messaging.ServiceBus;
    using Azure.Identity;
    using System.Timers;
    
    
    // TODO: Replace <YOUR-NAMESPACE> with your Service Bus namespace
    string svcbusNameSpace = "<YOUR-NAMESPACE>.servicebus.windows.net";
    string queueName = "myQueue";
    
    
    // ADD CODE TO CREATE A SERVICE BUS CLIENT
    
    
    
    // ADD CODE TO SEND MESSAGES TO THE QUEUE
    
    
    
    // ADD CODE TO PROCESS MESSAGES FROM THE QUEUE
    
    
    
    // Dispose client after use
    await client.DisposeAsync();
    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

### キューにメッセージを送信するコードを追加します

次に、Service Bus クライアントを作成し、一連のメッセージをキューに送信するコードを追加します。

1. **// ADD CODE TO CREATE A SERVICE BUS CLIENT** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a Service Bus client using the namespace and DefaultAzureCredential
    // The DefaultAzureCredential will use the Azure CLI credentials, so ensure you are logged in
    ServiceBusClient client = new(svcbusNameSpace, new DefaultAzureCredential(options));
    ```

1. **// ADD CODE TO SEND MESSAGES TO THE QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Create a sender for the specified queue
    ServiceBusSender sender = client.CreateSender(queueName);
    
    // create a batch 
    using ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync();
    
    // number of messages to be sent to the queue
    const int numOfMessages = 3;
    
    for (int i = 1; i <= numOfMessages; i++)
    {
        // try adding a message to the batch
        if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
        {
            // if it is too large for the batch
            throw new Exception($"The message {i} is too large to fit in the batch.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of messages to the Service Bus queue
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
    }
    finally
    {
        // Calling DisposeAsync on client types is required to ensure that network
        // resources and other unmanaged objects are properly cleaned up.
        await sender.DisposeAsync();
    }
    
    Console.WriteLine("Press any key to continue");
    Console.ReadKey();
    ```

1. **Ctrl + S** キーを押してファイルを保存し、演習を続行します。

### キュー内のメッセージを処理するコードを追加します

1. **// ADD CODE TO PROCESS MESSAGES FROM THE QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Create a processor that we can use to process the messages in the queue
    ServiceBusProcessor processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions());
    
    // Idle timeout in milliseconds, the idle timer will stop the processor if there are no more 
    // messages in the queue to process
    const int idleTimeoutMs = 3000;
    System.Timers.Timer idleTimer = new(idleTimeoutMs);
    idleTimer.Elapsed += async (s, e) =>
    {
        Console.WriteLine($"No messages received for {idleTimeoutMs / 1000} seconds. Stopping processor...");
        await processor.StopProcessingAsync();
    };
    
    try
    {
        // add handler to process messages
        processor.ProcessMessageAsync += MessageHandler;
    
        // add handler to process any errors
        processor.ProcessErrorAsync += ErrorHandler;
    
        // start processing 
        idleTimer.Start();
        await processor.StartProcessingAsync();
    
        Console.WriteLine($"Processor started. Will stop after {idleTimeoutMs / 1000} seconds of inactivity.");
        // Wait for the processor to stop
        while (processor.IsProcessing)
        {
            await Task.Delay(500);
        }
        idleTimer.Stop();
        Console.WriteLine("Stopped receiving messages");
    }
    finally
    {
        // Dispose processor after use
        await processor.DisposeAsync();
    }
    
    // handle received messages
    async Task MessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        Console.WriteLine($"Received: {body}");
    
        // Reset the idle timer on each message
        idleTimer.Stop();
        idleTimer.Start();
    
        // complete the message. message is deleted from the queue. 
        await args.CompleteMessageAsync(args.Message);
    }
    
    // handle any errors when receiving messages
    Task ErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine(args.Exception.ToString());
        return Task.CompletedTask;
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

1. 次のコマンドを実行して、コンソール アプリを起動します。 アプリはさまざまなステージで一時停止し、キー入力によって処理を続行します。 これにより、Azure portal のメッセージを確認できます。

    ```
    dotnet run
    ```

    

1. Azure portal で、作成した Service Bus 名前空間に移動します。 

1. **[概要]** ウィンドウの下部にある **[myqueue]** を選択します。

1. 左側のナビゲーション ウィンドウの **[Service Bus Explorer]** を選択します。

1. **[最初からクイック表示]** を選択します。数秒後に 3 つのメッセージが表示されます。

1. Cloud Shell で任意のキーを押して続行すると、アプリケーションによって 3 つのメッセージが処理されます。 
 
1. アプリケーションによるメッセージの処理が完了したら、ポータルに戻ります。 もう一度 **[最初からクイック表示]** を選択し、キューにメッセージがないことを確認します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合、その範囲外にある既存のリソースは 

