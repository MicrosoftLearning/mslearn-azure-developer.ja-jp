---
lab:
  topic: Azure container services
  title: Azure CLI コマンドを使用して Azure Container Instances にコンテナーをデプロイする
  description: Azure CLI コマンドを使用してコンテナーを Azure Container Instances にデプロイする方法について説明します。
---

# Azure CLI コマンドを使用して Azure Container Instances にコンテナーをデプロイする

この演習では、Azure CLI を使用して、Azure Container Instances (ACI) にコンテナーをデプロイして実行します。 コンテナー グループを作成し、コンテナー設定を指定し、コンテナ化されたアプリケーションがクラウドで実行されていることを確認する方法について説明します。

この演習で実行されるタスク:

* Azure で Azure コンテナー インスタンス リソースを作成する
* コンテナーを作成してデプロイする
* コンテナーが実行中であることを確認する

この演習の所要時間は約 **15** 分です。

## リソース グループを作成する

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。

1. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal 内に新しいクラウド シェルを作成します。***Bash*** 環境を選択すること。 Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。 ファイルを保持するストレージ アカウントを選択するように求められた場合は、**[ストレージ アカウントは必要ありません]**、お使いのサブスクリプションを選択し、**[適用]** を選択します。

    > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成している場合は、それを ***Bash*** に切り替えること。

1. この演習に必要なリソースのリソース グループを作成します。 **myResourceGroup** をリソース グループに使用する名前に置き換えます。 必要に応じて、**eastus** を最寄りのリージョンに置き換えることができます。 使用するリソース グループが既にある場合は、次の手順に進みます。

    ```
    az group create --location eastus --name myResourceGroup
    ```

## コンテナーを作成してデプロイする

名前、Docker イメージ、および Azure リソース グループを **az container create** コマンドに指定することで、コンテナーを作成します。 DNS 名ラベルを指定して、コンテナーをインターネットに公開します。

1. 次のコマンドを実行して、コンテナーをインターネットに発行するために使用する DNS 名を作成します。 DNS 名は一意である必要があります。Cloud Shell から次のコマンドを実行して、一意の名前を保持する変数を作成します。

    ```bash
    DNS_NAME_LABEL=aci-example-$RANDOM
    ```

1. 次のコマンドを実行して、コンテナー インスタンスを作成します。 **myResourceGroup** と **myLocation** を先ほど使用した値に置き換えます。 この操作の完了には数分かかります。

    ```bash
    az container create --resource-group myResourceGroup \
        --name mycontainer \
        --image mcr.microsoft.com/azuredocs/aci-helloworld \
        --ports 80 \
        --dns-name-label $DNS_NAME_LABEL --location myLocation \
        --os-type Linux \
        --cpu 1 \
        --memory 1.5 
    ```

    前のコマンドでは、**$DNS_NAME_LABEL** に DNS 名を指定しています。 イメージ名 **mcr.microsoft.com/azuredocs/aci-helloworld** は、基本的な Node.js Web アプリケーションを実行する Docker イメージを参照しています。

**az container create** コマンドが完了したら、次のセクションに進みます。

## コンテナーが実行中であることを確認する

コンテナーのビルド状態は、**az container show** コマンドで確認できます。 

1. 次のコマンドを実行して、作成したコンテナーのプロビジョニング状態を確認します。 **myResourceGroup** を先ほど使用した値に置き換えます。

    ```bash
    az container show --resource-group myResourceGroup \
        --name mycontainer \
        --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
        --out table 
    ```

    コンテナーの完全修飾ドメイン名 (FQDN) とそのプロビジョニング状態が表示されます。 次に例を示します。

    ```
    FQDN                                    ProvisioningState
    --------------------------------------  -------------------
    aci-wt.eastus.azurecontainer.io         Succeeded
    ```

    > **注:**  コンテナーが**作成中**状態の場合は、しばらく待ってから、**成功**状態が表示されるまでコマンドを再実行します。

1. ブラウザーからコンテナーの FQDN に移動して、動作していることを確認します。 サイトが安全ではないという警告が表示される場合があります。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。
