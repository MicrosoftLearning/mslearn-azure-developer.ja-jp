---
lab:
  topic: Azure events and messaging
  title: Azure Queue Storage からメッセージを送受信する
  description: .NET Azure.StorageQueues SDK を使用して、Azure Queue Storage からメッセージを送信する方法について説明します。
---

# Azure Queue Storage からメッセージを送受信する

この演習では、Azure Queue Storage リソースを作成して構成し、**Azure.Storage.Queues SDK** を使用してメッセージを送受信する .NET アプリを構築します。 ストレージ リソースをプロビジョニングし、キュー メッセージを管理し、完了時に環境をクリーンアップする方法について説明します。 

この演習で実行されるタスク:

* Azure Queue Storage リソースを作成する
* Microsoft Entra ユーザー名にロールを割り当てる
* メッセージを送受信する .NET コンソール アプリを作成する
* リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

## Azure Queue Storage リソースを作成する

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
    storAcctName=storactname$RANDOM
    ```

1. この演習の後半で、この名前空間に割り当てたストレージ アカウントが必要になります。 次のコマンドを実行し、出力をメモします。

    ```
    echo $storAcctName
    ```

1. 次のコマンドを実行して、先ほど作成した変数を使用してストレージ アカウントを作成します。 この操作は、完了するまでに数分かかります。

    ```bash
    az storage account create --resource-group $resourceGroup \
        --name $storAcctName --location $location --sku Standard_LRS
    ```

### Microsoft Entra ユーザー名にロールを割り当てる

アプリがメッセージを送受信できるようにするには、Microsoft Entra ユーザーを**ストレージ キュー データ共同作成者**ロールに割り当てます。 これにより、キューを作成し、Azure RBAC を使用してメッセージを送受信するアクセス許可がユーザー アカウントに付与されます。 Cloud Shell で次の手順を実行します。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。 これは、ロールが割り当てられる対象を表します。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 次のコマンドを実行して、ストレージ アカウントのリソース ID を取得します。 リソース ID によって、特定の名前空間に対するロールの割り当てのスコープが決まります。

    ```
    resourceID=$(az storage account show --resource-group $resourceGroup \
        --name $storAcctName --query id --output tsv)
    ```

1. 次のコマンドを実行して、**ストレージ キュー データ共同作成者**ロールを作成して割り当てます。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Queue Data Contributor" \
        --scope $resourceID
    ```

## メッセージを送受信する .NET コンソール アプリを作成する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```
    mkdir queuestor
    cd queuestor
    ```

1. .NET コンソール アプリケーションを作成します。

    ```
    dotnet new console
    ```

1. 次のコマンドを実行して、**Azure.Storage.Queues** および **Azure.Identity** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Azure.Storage.Queues
    dotnet add package Azure.Identity
    ```

### プロジェクトのスタート コードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```
    code Program.cs
    ```

1. 既存の内容を次のコードに置き換えます。 必ずコード内のコメントを確認し、**<YOUR-STORAGE-ACCT-NAME>** を先ほどメモしたストレージ アカウント名に置き換えます。

    ```csharp
    using Azure;
    using Azure.Identity;
    using Azure.Storage.Queues;
    using Azure.Storage.Queues.Models;
    using System;
    using System.Threading.Tasks;
    
    // Create a unique name for the queue
    // TODO: Replace the <YOUR-STORAGE-ACCT-NAME> placeholder 
    string queueName = "myqueue-" + Guid.NewGuid().ToString();
    string storageAccountName = "<YOUR-STORAGE-ACCT-NAME>";
    
    // ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE
    
    
    
    // ADD CODE TO SEND AND LIST MESSAGES
    
    
    
    // ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES
    
    
    
    // ADD CODE TO DELETE MESSAGES AND THE QUEUE
    
    
    ```

1. **Ctrl + S** キーを押して変更内容を保存します。

### キュー クライアントを作成し、キューを作成するコードを追加します

次に、キュー ストレージ クライアントを作成し、キューを作成するコードを追加します。

1. **// ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Instantiate a QueueClient to create and interact with the queue
    QueueClient queueClient = new QueueClient(
        new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
        new DefaultAzureCredential(options));
    
    Console.WriteLine($"Creating queue: {queueName}");
    
    // Create the queue
    await queueClient.CreateAsync();
    
    Console.WriteLine("Queue created, press Enter to add messages to the queue...");
    Console.ReadLine();
    ```

1. **Ctrl + S** キーを押してファイルを保存し、演習を続行します。

### キュー内のメッセージを送信し、一覧表示するコードを追加します

1. **// ADD CODE TO SEND AND LIST MESSAGES** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Send several messages to the queue with the SendMessageAsync method.
    await queueClient.SendMessageAsync("Message 1");
    await queueClient.SendMessageAsync("Message 2");
    
    // Send a message and save the receipt for later use
    SendReceipt receipt = await queueClient.SendMessageAsync("Message 3");
    
    Console.WriteLine("Messages added to the queue. Press Enter to peek at the messages...");
    Console.ReadLine();
    
    // Peeking messages lets you view the messages without removing them from the queue.
    
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to update a message in the queue...");
    Console.ReadLine();
    ```

1. **Ctrl + S** キーを押してファイルを保存し、演習を続行します。

### メッセージを更新し、結果を一覧表示するコードを追加します

1. **// ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Update a message with the UpdateMessageAsync method and the saved receipt
    await queueClient.UpdateMessageAsync(receipt.MessageId, receipt.PopReceipt, "Message 3 has been updated");
    
    Console.WriteLine("Message three updated. Press Enter to peek at the messages again...");
    Console.ReadLine();
    
    
    // Peek messages from the queue to compare updated content
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to delete messages from the queue...");
    Console.ReadLine();
    ```

1. **Ctrl + S** キーを押してファイルを保存し、演習を続行します。

### メッセージとキューを削除するコードを追加します

1. **// ADD CODE TO DELETE MESSAGES AND THE QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。 必ずコードとコメントを確認してください。

    ```csharp
    // Delete messages from the queue with the DeleteMessagesAsync method.
    foreach (var message in (await queueClient.ReceiveMessagesAsync(maxMessages: 10)).Value)
    {
        // "Process" the message
        Console.WriteLine($"Deleting message: {message.MessageText}");
    
        // Let the service know we're finished with the message and it can be safely deleted.
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
    Console.WriteLine("Messages deleted from the queue.");
    Console.WriteLine("\nPress Enter key to delete the queue...");
    Console.ReadLine();
    
    // Delete the queue with the DeleteAsync method.
    Console.WriteLine($"Deleting queue: {queueClient.Name}");
    await queueClient.DeleteAsync();
    
    Console.WriteLine("Done");
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

1. 左側のナビゲーションの **[> データ ストレージ]** を展開し、**[キュー]** を選択します。

1. アプリケーションで作成するキューを選択します。送信されたメッセージを表示し、アプリケーションの実行状況を監視することができます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

