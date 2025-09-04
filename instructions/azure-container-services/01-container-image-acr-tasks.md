---
lab:
  topic: Azure container services
  title: Azure Container Registry タスクを使用してコンテナー イメージをビルドして実行する
  description: Azure CLI コマンドを使用して、Azure Container Registry タスクでコンテナー イメージをビルドおよび実行する方法について説明します。
---

# Azure Container Registry タスクを使用してコンテナー イメージをビルドして実行する

この演習では、アプリケーション コードからコンテナー イメージをビルドし、Azure CLI を使用して Azure Container Registry にプッシュします。 アプリをコンテナ化用に準備し、ACR インスタンスを作成し、コンテナー イメージを Azure に格納する方法について説明します。

この演習で実行されるタスク:

* Azure Container Registry  リソースの作成
* Dockerfile からイメージをビルドしてプッシュする
* 結果を確認する
* Azure Container Registry でイメージを実行する

この演習の所要時間は約 **20** 分です。

## Azure Container Registry  リソースの作成

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. この演習に必要なリソースのリソース グループを作成します。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus** を最寄りのリージョンに置き換えることができます。 使用するリソース グループが既にある場合は、次の手順に進みます。

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. 次のコマンドを実行して基本的なコンテナー レジストリを作成します。 レジストリ名は Azure 内で一意にする必要があります。また、5 文字から 50 文字の英数字で構成する必要があります。 **myResourceGroup** を先ほど使用した名前に置き換え、**myContainerRegistry** を一意の値に置き換えます。

    ```bash
    az acr create --resource-group myResourceGroup \
        --name myContainerRegistry --sku Basic
    ```

    > **注:**  このコマンドにより、"*基本*" レジストリが作成されます。これは、Azure Container Registry について学習する開発者向けにコスト最適化されたオプションです。

## Dockerfile からイメージをビルドしてプッシュする

次に、Dockerfile に基づいてイメージをビルドしてプッシュします。

1. 次のコマンドを実行して Dockerfile を作成します。 Dockerfile には、Microsoft Container Registry でホストされている *hello-world* イメージを参照する 1 行が含まれています。

    ```bash
    echo FROM mcr.microsoft.com/hello-world > Dockerfile
    ```

1. 次の **az acr build** コマンドを実行してイメージをビルドします。イメージが正常にビルドされたら、それをレジストリにプッシュします。 **myContainerRegistry** を先ほど作成した名前に置き換えます。

    ```bash
    az acr build --image sample/hello-world:v1  \
        --registry myContainerRegistry \
        --file Dockerfile .
    ```

    次は前のコマンドの出力を短くしたサンプルです。最後の数行に最終結果があります。 *repository* フィールドに、*sample/hello-word* イメージが一覧表示されていることを確認できます。

    ```
    - image:
        registry: myContainerRegistry.azurecr.io
        repository: sample/hello-world
        tag: v1
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      runtime-dependency:
        registry: mcr.microsoft.com
        repository: hello-world
        tag: latest
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      git: {}
    
    
    Run ID: cf1 was successful after 11s
    ```

## 結果を確認する

1. 次のコマンドを実行して、レジストリ内のリポジトリを一覧表示します。 **myContainerRegistry** を先ほど作成した名前に置き換えます。

    ```bash
    az acr repository list --name myContainerRegistry --output table
    ```

    出力:

    ```
    Result
    ----------------
    sample/hello-world
    ```

1. 次のコマンドを実行して、**sample/hello-world** リポジトリのタグを一覧表示します。 **myContainerRegistry** を先ほど使用した名前に置き換えます。

    ```bash
    az acr repository show-tags --name myContainerRegistry \
        --repository sample/hello-world --output table
    ```

    出力:

    ```
    Result
    --------
    v1
    ```

## ACR でイメージを実行する

1. **az acr run** コマンドを使用して、コンテナー レジストリから *sample/hello-world:v1* コンテナー イメージを実行します。 次の例では、**$Registry** を使用して、コマンドを実行するレジストリを指定します。 **myContainerRegistry** を先ほど使用した名前に置き換えます。

    ```bash
    az acr run --registry myContainerRegistry \
        --cmd '$Registry/sample/hello-world:v1' /dev/null
    ```

    この例の **cmd** パラメーターでは、既定の構成でコンテナーが実行されますが、**cmd** は他の **docker run** パラメーターや他の **docker** コマンドもサポートしています。 

    次の出力例は短縮されています。

    ```
    Packing source code into tar to upload...
    Uploading archived source code from '/tmp/run_archive_ebf74da7fcb04683867b129e2ccad5e1.tar.gz'...
    Sending context (1.855 KiB) to registry: mycontainerre...
    Queued a run with ID: cab
    Waiting for an agent...
    2019/03/19 19:01:53 Using acb_vol_60e9a538-b466-475f-9565-80c5b93eaa15 as the home volume
    2019/03/19 19:01:53 Creating Docker network: acb_default_network, driver: 'bridge'
    2019/03/19 19:01:53 Successfully set up Docker network: acb_default_network
    2019/03/19 19:01:53 Setting up Docker configuration...
    2019/03/19 19:01:54 Successfully set up Docker configuration
    2019/03/19 19:01:54 Logging in to registry: mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Successfully logged into mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Executing step ID: acb_step_0. Working directory: '', Network: 'acb_default_network'
    2019/03/19 19:01:55 Launching container with name: acb_step_0
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    2019/03/19 19:01:56 Successfully executed container: acb_step_0
    2019/03/19 19:01:56 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.843801)
    
    Run ID: cab was successful after 6s
    ```

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。
