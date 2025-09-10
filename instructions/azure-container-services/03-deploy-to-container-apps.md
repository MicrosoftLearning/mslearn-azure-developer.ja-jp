---
lab:
  topic: Azure container services
  title: Azure CLI を使用して Azure Container Apps にコンテナーをデプロイする
  description: Azure CLI コマンドを使用してセキュリティで保護された Azure Container Apps 環境を作成し、コンテナーをデプロイする方法について説明します。
---

# Azure CLI を使用して Azure Container Apps にコンテナーをデプロイする

この演習では、Azure CLI を使用して、コンテナ化されたアプリケーションを Azure Container Apps にデプロイします。 コンテナー アプリ環境を作成し、コンテナーをデプロイし、アプリケーションが Azure で実行されていることを確認する方法について説明します。

この演習で実行されるタスク:

* Azure でリソースを作成する
* Azure Container Apps 環境を作成する
* コンテナー アプリを環境にデプロイする

この演習の所要時間は約 **15** 分です。

## リソース グループを作成し、Azure 環境を準備する

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. この演習に必要なリソースのリソース グループを作成します。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus** を最寄りのリージョンに置き換えることができます。 使用するリソース グループが既にある場合は、次の手順に進みます。

    ```azurecli
    az group create --location eastus --name myResourceGroup
    ```

1. 次のコマンドを実行して、CLI 用の Azure Container Apps 拡張機能の最新バージョンがインストールされていることを確認します。

    ```azurecli
    az extension add --name containerapp --upgrade
    ```

### 名前空間を登録する

Azure Container Apps に登録する必要がある名前空間は 2 つあります。以下の手順では、それらが登録されていることを確認します。 サブスクリプションでまだ構成されていない場合は、各登録に数分かかることがあります。 

1. **Microsoft.App** 名前空間を登録します。 

    ```bash
    az provider register --namespace Microsoft.App
    ```

1. これまで使用したことがない場合は、Azure Monitor Log Analytics ワークスペースの **Microsoft.OperationalInsights** プロバイダーを登録します。

    ```bash
    az provider register --namespace Microsoft.OperationalInsights
    ```

## Azure Container Apps 環境を作成する

Azure Container Apps 環境では、コンテナー アプリのグループを囲むセキュリティ保護された境界が作成されます。 同じ環境にデプロイされた Container Apps は、同じ仮想ネットワークにデプロイされ、同じ Log Analytics ワークスペースにログを書き込みます。

1. **az containerapp env create** コマンドを使用して環境を作成します。 **myResourceGroup** と **myLocation** を先ほど使用した値に置き換えます。 この操作の完了には数分かかります。

    ```bash
    az containerapp env create \
        --name my-container-env \
        --resource-group myResourceGroup \
        --location myLocation
    ```

## コンテナー アプリを環境にデプロイする

コンテナー アプリ環境のデプロイが完了したら、コンテナー イメージを環境にデプロイできます。

1. **containerapp create** コマンドを使用してサンプル アプリ コンテナー イメージをデプロイします。 **myResourceGroup** を先ほど使用した値に置き換えます。

    ```bash
    az containerapp create \
        --name my-container-app \
        --resource-group myResourceGroup \
        --environment my-container-env \
        --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
        --target-port 80 \
        --ingress 'external' \
        --query properties.configuration.ingress.fqdn
    ```

    **--ingress** を **external** に設定すると、コンテナー アプリをパブリック要求に使用できるようになります。 このコマンドから、アプリにアクセスするリンクが返されます。

    ```
    Container app created. Access your app at <url>
    ```

デプロイを確認するには、**az containerapp create** コマンドから返された URL を選択して、コンテナー アプリが実行中であることを確認します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。
