---
lab:
  topic: Azure events and messaging
  title: Azure Event Grid を使用してイベントをカスタム エンドポイントにルーティングする
  description: Azure Event Grid を使用してイベントをカスタム エンドポイントにルーティングする方法について説明します。
---

# Azure Event Grid を使用してイベントをカスタム エンドポイントにルーティングする

この演習では、Azure Event Grid トピックと Web アプリ エンドポイントを作成し、Event Grid トピックにカスタム イベントを送信する .NET コンソール アプリケーションを構築します。 イベント サブスクリプションを構成し、Event Grid で認証し、Web アプリでイベントを表示してイベントがエンドポイントに正常にルーティングされていることを確認する方法について説明します。

この演習で実行されるタスク:

* Azure Event Grid リソースを作成する
* Event Grid リソース プロバイダーを有効にする
* Event Grid でトピックを作成する
* メッセージ エンドポイントの作成
* トピックを購読する
* .NET コンソール アプリでイベントを送信する
* リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

## Azure Event Grid リソースを作成する

演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. Cloud Shell ツール バーの **[設定]** メニューで、**[クラシック バージョンに移動]** を選択します (これはコード エディターを使用するのに必要です)。

1. この演習に必要なリソースのリソース グループを作成します。 使用するリソース グループが既にある場合は、次の手順に進みます。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus** を最寄りのリージョンに置き換えることができます。

    ```bash
    az group create --name myResourceGroup --location eastus
    ```

1. 一意の名前が必須であり、同じパラメーターを使用するコマンドが多くあります。 変数をいくつか作成すると、リソースを作成するコマンドに必要な変更が減ります。 次のコマンドを実行して、必要な変数を作成します。 **myResourceGroup** を、この演習で使用する名前に置き換えます。 前の手順で場所を変更した場合は、**location** 変数にも同じ変更を加えます。

    ```bash
    let rNum=$RANDOM
    resourceGroup=myResourceGroup
    location=eastus
    topicName="mytopic-evgtopic-${rNum}"
    siteName="evgsite-${rNum}"
    siteURL="https://${siteName}.azurewebsites.net"
    ```

### Event Grid リソース プロバイダーを有効にする

Azure リソース プロバイダーは、Azure 内の特定の種類のリソースを定義および管理するサービスです。 リソースをデプロイまたは管理するときに、Azure のバックグラウンドで使用されます。 **az provider register** コマンドを使用して Event Grid リソース プロバイダーを登録します。 

```bash
az provider register --namespace Microsoft.EventGrid
```

登録の完了には数分かかる場合があります。 次のコマンドで状態を確認できます。

```bash
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

> **注:**  この手順は、以前に Event Grid 使用したことがないサブスクリプションでのみ必要です。

### Event Grid でトピックを作成する

**az eventgrid topic create** コマンドを使用してトピックを作成します。 名前は、DNS エントリの一部であるため、一意である必要があります。  

```bash
az eventgrid topic create --name $topicName \
    --location $location \
    --resource-group $resourceGroup
```

### メッセージ エンドポイントの作成

カスタム トピックをサブスクライブする前に、イベント メッセージ用のエンドポイントを作成する必要があります。 通常、エンドポイントは、イベント データに基づくアクションを実行します。 次のスクリプトでは、イベント メッセージを表示する事前構築済みの Web アプリを使います。 デプロイされたソリューションには、App Service プラン、App Service Web アプリ、および GitHub からのソース コードが含まれています。

1. 次のコマンドを実行してメッセージ エンドポイントを作成します。 **echo** コマンドを実行してエンドポイントのサイト URL を表示します。

    ```bash
    az deployment group create \
        --resource-group $resourceGroup \
        --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
        --parameters siteName=$siteName hostingPlanName=viewerhost
    
    echo "Your web app URL: ${siteURL}"
    ```

    > **注:**  このコマンドが完了するまでに数分かかる場合があります。

1. ブラウザーで新しいタブを開き、前のスクリプトの最後に生成された URL に移動して、Web アプリが実行されていることを確認します。 現在表示されているメッセージがないサイトが表示されます。

    > **ヒント:** ブラウザーを実行したまま、更新の表示に使用します。

### トピックを購読する

どのイベントを追跡し、どこにイベントを送信するかは、Event Grid トピックにサブスクライブすることによって Event Grid に伝えます。 

1. **az eventgrid event-subscription create** コマンドを使用してトピックを購読します。 次のスクリプトでは、アカウントからサブスクリプション ID を取得し、それをイベント サブスクリプションの作成に使用します。

    ```bash
    endpoint="${siteURL}/api/updates"
    topicId=$(az eventgrid topic show --resource-group $resourceGroup \
        --name $topicName --query "id" --output tsv)
    
    az eventgrid event-subscription create \
        --source-resource-id $topicId \
        --name TopicSubscription \
        --endpoint $endpoint
    ```

1. Web アプリをもう一度表示し、その Web アプリにサブスクリプションの検証イベントが送信されたことに注目します。 目のアイコンを選択してイベント データを展開します。 Event Grid は検証イベントを送信するので、エンドポイントはイベント データを受信することを確認できます。 Web アプリには、サブスクリプションを検証するコードが含まれています。

## .NET コンソール アプリケーションでイベントを送信する

必要なリソースが Azure にデプロイされたら、次の手順はコンソール アプリケーションの設定です。 以下の手順は Cloud Shell で実行されます。

>**ヒント:** 上部の境界線をドラッグして Cloud Shell のサイズを変更すると、より多くの情報とコードが表示されます。 最小化ボタンと最大化ボタンを使用して、Cloud Shell とメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行してプロジェクトを格納するディレクトリを作成し、プロジェクト ディレクトリに変更します。

    ```bash
    mkdir eventgrid
    cd eventgrid
    ```

1. .NET コンソール アプリケーションを作成します。

    ```bash
    dotnet new console
    ```

1. 次のコマンドを実行して、**Azure.Messaging.EventGrid** および **dotenv.net** パッケージをプロジェクトに追加します。

    ```bash
    dotnet add package Azure.Messaging.EventGrid
    dotnet add package dotenv.net
    ```

### コンソール アプリケーションを構成する

このセクションでは、トピックのエンドポイントとアクセス キーを取得し、**.env** ファイルに追加して、それらのシークレットを保持できるようにします。

1. 次のコマンドを実行して、先ほど作成したトピックの URL とアクセス キーを取得します。 これらの値は必ずメモしておいてください。

    ```bash
    az eventgrid topic show --name $topicName -g $resourceGroup --query "endpoint" --output tsv
    az eventgrid topic key list --name $topicName -g $resourceGroup --query "key1" --output tsv
    ```

1. 次のコマンドを実行して、シークレットを保持する **.env** ファイルを作成し、コード エディターで開きます。

    ```bash
    touch .env
    code .env
    ```

1. 次のコードを **.env** ファイルに追加します。 **YOUR_TOPIC_ENDPOINT** と **YOUR_TOPIC_ACCESS_KEY** を、先ほどメモした値に置き換えます。

    ```
    TOPIC_ENDPOINT="YOUR_TOPIC_ENDPOINT"
    TOPIC_ACCESS_KEY="YOUR_TOPIC_ACCESS_KEY"
    ```

1. **Ctrl + S** キーを押してファイルを保存してから、**Ctrl + Q** キーを押してエディターを終了します。

次に、クラウド シェル内のエディターを使用して、**Program.cs** ファイル内のテンプレート コードを置き換えます。

### プロジェクトのコードを追加する

1. アプリケーションの編集を開始するには、クラウド シェル内で次のコマンドを実行します。

    ```bash
    code Program.cs
    ```

1. 既存のコードを次のコードに置き換えます。 コード内のコメントを必ず確認してください。

    ```csharp
    using dotenv.net; 
    using Azure.Messaging.EventGrid; 
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Start the asynchronous process to send an Event Grid event
    ProcessAsync().GetAwaiter().GetResult();
    
    async Task ProcessAsync()
    {
        // Retrieve Event Grid topic endpoint and access key from environment variables
        var topicEndpoint = envVars["TOPIC_ENDPOINT"];
        var topicKey = envVars["TOPIC_ACCESS_KEY"];
        
        // Check if the required environment variables are set
        if (string.IsNullOrEmpty(topicEndpoint) || string.IsNullOrEmpty(topicKey))
        {
            Console.WriteLine("Please set TOPIC_ENDPOINT and TOPIC_ACCESS_KEY in your .env file.");
            return;
        }
    
        // Create an EventGridPublisherClient to send events to the specified topic
        EventGridPublisherClient client = new EventGridPublisherClient
            (new Uri(topicEndpoint),
            new Azure.AzureKeyCredential(topicKey));
    
        // Create a new EventGridEvent with sample data
        var eventGridEvent = new EventGridEvent(
            subject: "ExampleSubject",
            eventType: "ExampleEventType",
            dataVersion: "1.0",
            data: new { Message = "Hello, Event Grid!" }
        );
    
        // Send the event to Azure Event Grid
        await client.SendEventAsync(eventGridEvent);
        Console.WriteLine("Event sent successfully.");
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

1. Cloud Shell で次のコマンドを実行して、コンソール アプリケーションを起動します。 メッセージが送信されると、"**イベントが正常に送信されました。**"  というメッセージが表示されます。

    ```bash
    dotnet run
    ```

1. Web アプリを表示して、送信したイベント確認します。 目のアイコンを選択してイベント データを展開します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。