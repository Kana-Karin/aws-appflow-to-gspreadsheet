## 概要
AWS AppFlow を利用して、Google スプレッドシートとAWS S3 bucket内へのデータ転送を自動化します。  
今回は[統計地理情報システムデータダウンロード | 政府統計の総合窓口](https://www.e-stat.go.jp/gis/statmap-search?type=1)にて公開されている、[職業（大分類）別就業者数01 北海道]のデータを利用させていただきました。  
(転送量の事情でデータを札幌市、函館市、私の出身地の苫小牧市に絞っています)

#### AppFlowとは？
サービスとしてのソフトウェア (SaaS) アプリケーションと AWS サービス間でデータを安全に転送させる完全マネージド型の統合サービスです。
- **セキュリティとプライバシー**
Amazon AppFlow を流れるすべてのデータは、保管時と転送中に暗号化されます。AWS キーでデータを暗号化したり、独自のカスタムキーを使用したりできます。
- **スケーラビリティ**
Amazon AppFlow では、リソースを計画またはプロビジョニングしなくても簡単にスケールアップできるため、大量のデータを複数のバッチに分割することなく移動できます。Amazon AppFlow を使用すると、数百万件の Salesforce レコードや Zendesk チケットをすべて単一のフローを実行しながら簡単に転送できます。
- **スピードとオートメーション**
Amazon AppFlow を使用すると、数分でアプリケーションを統合できます。カスタムコネクタをコーディングするのに数日または数週間待つ必要はありません。


## アーキテクチャ図


## 前提条件
- Googleアカウントの作成済み
- AWS アカウントの作成済み、適切なロールを付与済み
- Google Cloudにてプロジェクトの作成済み、適切なロール、スコープを付与済み

## Google Cloud 認証情報の作成
認証情報を作成 → [OAuth クライアント ID] を選択  
<img width="406" alt="スクリーンショット 2023-12-13 15 41 34" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/6c3d1f39-e31f-4610-8ffa-e9f393327a76">

---

アプリケーションの種類、名前を指定し[作成]をクリック
```
アプリケーションの種類:ウェブアプリケーション
名前:aws-appfow
```
<img width="548" alt="スクリーンショット 2023-12-13 15 45 09" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/7c7775a4-cd50-4c0c-9ba5-0540a5c4d33d">


[Google Sheets connector for Amazon AppFlow - Amazon AppFlow](https://docs.aws.amazon.com/appflow/latest/userguide/connectors-google-sheets.html)こちらのドキュメントをもとに、以下のURIを追加します。  
[承認済みの JavaScript 生成元の[URI を追加]  
`https://console.aws.amazon.com`  

[承認済みのリダイレクト URI]の[URI を追加]
`https://ap-northeast-1.console.aws.amazon.com/appflow/oauth`  
※東京以外のリージョンを使用する場合は、[ap-northeast-1]の部分を変更します。

- ウェブ アプリケーションをホストする HTTP オリジン。**この値にワイルドカードやパスを含めることはできません。**80 以外のポートを使用する場合は、ポートを指定する必要あります。例: https://example.com:8080
- ユーザーは Google で認証されると、このパスにリダイレクトされます。 パスには、アクセス用の認証コードが付加されます。またプロトコルを含める必要があります。パスには URL フラグメント、相対パス、ワイルドカードは使用できず、またパブリック IP アドレスは指定できません。


作成すると、以下の画面が表示されます。  
<img width="405" alt="スクリーンショット 2023-12-13 15 49 01" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/496311f1-3bc3-4951-854e-77912c86674c">

こちらでクライアントID、クライアントシークレットIDの値を控えておきます。  
※JSON形式でのダウンロードも可能です

---

次に、[有効なAPIとサービス]　→[API とサービスの有効化]をクリック  
<img width="801" alt="スクリーンショット 2023-12-13 15 53 19" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/e189830c-3856-4ebb-95a2-171c47f5f9d4">


API ライブラリから、Google Workspace　→ [Google Sheets API]を選択  
<img width="441" alt="スクリーンショット 2023-12-13 15 56 00" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/b8ec2a21-c6de-4e22-b50b-02951e39978a">


以下の画面が表示されますので、「有効にする」をクリック  
<img width="414" alt="スクリーンショット 2023-12-13 15 57 54" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/3b22f7ac-6368-4c2a-a080-a3537a28aa02">  

同じく、[Google Drive API]も有効にします。  
<img width="455" alt="スクリーンショット 2023-12-13 15 59 59" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/0398c75d-15d3-43c5-8851-7aa25b03eac4">

---

次に[認証情報] →[サービス アカウントを管理]を選択します
次に、[サービスアカウントを作成]をクリック  
<img width="440" alt="スクリーンショット 2023-12-13 16 01 27" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/009d0d88-7b5e-4091-9bb8-72f4fe83d6c6">  
<img width="432" alt="スクリーンショット 2023-12-13 16 04 28" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/177e2e66-9d71-4393-a9f7-dc8d5ef03f9a">


次に、サービス アカウントの詳細を入力していきます。  
<img width="479" alt="スクリーンショット 2023-12-13 16 07 32" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/bfa5787d-4a89-4acc-96fc-f93563355860">  
```
サービス アカウント名:aws-appflow
サービス アカウント ID:aws-appflow
サービス アカウントの説明:任意
```


作成後、任意の[ロール]を選択します。
※セキュリティ上、ロールの選択には注意してください
<img width="483" alt="スクリーンショット 2023-12-13 16 09 20" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/56f28a61-61dc-427b-9634-216a2017e834">  

[続行]をクリックします。  
サービスアカウントが作成されたことを確認します。
<img width="696" alt="スクリーンショット 2023-12-13 16 13 16" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/d2c5180e-8db2-4ed7-abee-6825b0d40a3b">


## AppFlowの接続とフローを構築する

まず、リージョンを先ほどGoogle Cloudで指定したURIのリージョンに指定されているか確認します。  
<img width="380" alt="スクリーンショット 2023-12-13 16 19 51" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/f9a9cb41-dacc-425a-ad74-6dfd88be1855">



AWSマネコンからAmazon AppFlowサービスをクリックします。
以下の画面が表示されるので、[フローの作成]クリック。  
<img width="422" alt="スクリーンショット 2023-12-13 16 20 39" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/0f027e08-17dc-4d62-9c32-75bcc8d1856d">


フローのセットアップ画面が表示されますので、入力していきます。  
```
フロー名:appflow-gspreadsheet-s3
フローの説明:Google Spreadsheet send to AWS S3 bucket.
```  
※[データ暗号化]につきましてはあくまで検証なので有効にしません。  
※タグにつきましても、アクセスや請求等を制御する場合はタグを追加してください。  
<img width="418" alt="スクリーンショット 2023-12-13 16 29 00" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/1a20bf1d-d5a9-47a9-975f-5e3b6838655a">


次に、[送信元名]に[Google Sheets]を選択します。
[Google Sheets 接続を選択]の[新規接続を作成]を選択します。
<img width="444" alt="スクリーンショット 2023-12-13 16 30 01" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/5a790756-bed9-481a-8e04-808ba0d6a338">

先ほどGoogle認証情報で作成したOAuth クライアントID、シークレットIDを貼り付けます。  

次にアカウントの選択を行い、Google にログインします。  
「このアプリは Google で確認されていません」と表示がされました。  
Google Cloudにてロール付与済みなので[続行]をクリックします  
<img width="477" alt="スクリーンショット 2023-12-13 16 57 06" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/4715a121-5566-47b7-a9c9-7482e24ef637">  



「amazon.com が Google アカウントへの追加アクセスを求めています」の画面が表示されるので、  
[Google スプレッドシートのすべてのスプレッドシートの参照です]を選択をし、  
[続行]をクリックします
<img width="471" alt="スクリーンショット 2023-12-13 16 58 05" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/0ab2506c-2831-4a31-b983-e6ffc2b0e43f">

---

AppFlowとGoogle Spreadsheetの接続が成功すると以下の画面が表示されます。  

以下画面では、送信先を選択します。  
[送信先名]から[Amazon S3]を選択、[バケットの詳細]からデータ転送先に指定するバケットを選択します。  
この際、自動的にフローに指定した名前のフォルダが追加されます。  
<img width="538" alt="スクリーンショット 2023-12-13 17 29 20" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/26f998db-74ea-49f4-9cd9-5720591e0e92">


今回のデータ形式は[JSON形式]を選択します。  
<img width="630" alt="スクリーンショット 2023-12-13 17 34 22" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/bc88c62c-38ce-4329-af2b-fd39aeb967ea">


次にフロートリガーの方法を選択します。  
今回は、フローをトリガーする方法に[スケジュール通りにフローを実行]を選択しています。  
必要に応じて、繰り返し[1時間毎], [開始日], [開始時間]などを指定し、  
転送モードに[完全転送]を選択しています
<img width="626" alt="スクリーンショット 2023-12-13 17 36 16" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/17d6d6b6-660a-455c-b879-de1f3c0f9cdb">


次にマッピングを設定します。マッピング方法は手動・CSVアップロードが選択可能ですが、  
今回は手動で設定します。  
送信元フィールドに[フィールドを直接マッピングする]をクリックして設定します  
<img width="636" alt="スクリーンショット 2023-12-13 17 38 33" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/ea8f3d1b-b0d0-497e-b2a7-b4c7d22e3cea">  
<img width="637" alt="スクリーンショット 2023-12-13 17 38 49" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/921ad5aa-a109-4c99-92e2-c8014cba8cc8">

確認画面が表示されるので、設定を確認後[フローを作成]クリックして作成します。


無事、フローが正常に作成されました。  
<img width="538" alt="スクリーンショット 2023-12-13 17 29 20 2" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/bee7ef84-0791-4834-94ad-0460121a984f">  

ステータスは下書きとなっていますので、**フローをアクティブ化**を押します。  
**もちろんですが、アクティブ化をしないと転送されません。**  
<img width="695" alt="スクリーンショット 2023-12-13 17 41 01" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/03e299c1-87c0-4904-9ec7-6aab81038a49">  

~~これを忘れてしまった人がいます。なぁーにぃ？！
やっちまったなぁ！~~


## Google SpreadsheetとS3のデータ転送のテスト
<img width="410" alt="スクリーンショット 2023-12-13 18 15 38" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/ceb30f57-7890-40fa-b6d2-c65fa078a9f5">

今回の実行IDは、[99b917ac-6eb7-3c4a-8be2-f7c72612f58f]になっていることが確認できます。
フローの「実行履歴」を選択し、トリガに設定したスケジュールにデータ転送が行われたこと、  
実行ステータスが[成功]していることを確認します。  
(データ量の場合により、転送する時間が5分ほどかかります)

AWSマネコンにてS3画面に行き、送信先のS3 bucketを確認します。  
フロー名に指定したフォルダ[appflow-google-s3/]が作成され、  
<img width="379" alt="スクリーンショット 2023-12-13 18 18 14" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/d8e2778f-6059-4f83-9ce5-402bcf3512fd">  
先ほどの実行ID [99b917ac-6eb7-3c4a-8be2-f7c72612f58f]のフォルダ配下にオブジェクトが作成されています。


S3 bucketに転送されたオブジェクトをダウンロードし、オブジェクトの中身を確認します。  
ファイル形式の設定で指定した通り、JSON形式でデータが格納されていることが確認できました。  
<img width="472" alt="スクリーンショット 2023-12-13 18 20 34" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/bd983453-b84f-46d1-8aff-b5ee270a6d9e">

最後までご閲覧していただきありがとうございました。

## つまづいた部分
AppFlowとGoogleを接続する際、読み取りスコープの付与が無かったので以下のエラーが発生。  
```
接続 google の作成中にエラーが発生しました。 Error authenticating to connector: Failed to validate Connection while attempting “ValidateCredentials with CustomConnector" with connector failure The request failed because the service Source Google Sheets returned the following error: Details: The request failed with status code 403 (Forbidden).. (Service: null; Status Code: 400; Error Code: Client; Request ID: null; Proxy: null)
```  
エラー文章から読み取り、Source Google Sheets自体がエラーを返していること、  
Request ID:null;、Proxy: nullになっていることを確認。  
ValidateCredentials(認証情報)の問題だと認識し、  
Google Cloud プロジェクト内の認証情報のスコープ管理から読み取りを許可することで解決。  

## リソースの削除
- AppFlowのフローを非アクティブ化
<img width="871" alt="スクリーンショット 2023-12-13 18 22 37" src="https://github.com/Kana-Karin/aws-appflow-to-gspreadsheet/assets/84316229/528dc663-d095-465b-b97e-653f011a643a">


## 参考にさせていただいたサイト
- [SaaS の統合 - Amazon AppFlow - AWS](https://aws.amazon.com/jp/appflow/?nc1=h_ls)
- [What is Amazon AppFlow? - AWS](https://docs.aws.amazon.com/appflow/latest/userguide/what-is-appflow.html)
- [Tutorial: Transfer data between applications with Amazon AppFlow - Amazon AppFlow](https://docs.aws.amazon.com/appflow/latest/userguide/flow-tutorial.html)
- [Google Sheets connector for Amazon AppFlow - Amazon AppFlow](https://docs.aws.amazon.com/appflow/latest/userguide/connectors-google-sheets.html)


## 最後に
Amazon AppFlowを使用し、コードの記述なしでGoogle Spreadsheetの連携、フローの自動化ができました。
今後はこのフローを利用してQuickSightやGlueサービスを利用し、可視化、データ収集などにチャレンジしていきます。
