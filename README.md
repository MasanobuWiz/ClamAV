# 
特徴
インストールが簡単
無制限の数のS3バケットからイベントを送信します
S3バケットポリシーを使用して感染ファイルの読み取りを防止する
エンドユーザーによるオープンソースのウイルス対策エンジンClamAVの個別インストールにアクセスします
使い方
アーキテクチャ-図

新しいオブジェクトがバケットに追加されるたびに、S3はLambda関数を呼び出してオブジェクトをスキャンします
関数パッケージは、（必要に応じて）現在のアンチウイルス定義をS3バケットからダウンロードします。S3バケットとLambdaの間の転送速度は、通常、別のソースよりも高速で信頼性があります
オブジェクトはウイルスとマルウェアについてスキャンされます。アーカイブファイルが抽出され、内部のファイルもスキャンされます
オブジェクトタグは、スキャンの日付と時刻とともに、スキャンの結果、CLEANまたはINFECTEDを反映するように更新されます。
スキャンの結果を反映するようにオブジェクトメタデータが更新されます（オプション）
メトリックはDataDogに送信されます（オプション）
スキャン結果はSNSトピックに公開されます（オプション）（オプションで、感染した結果のみを公開することを選択します）
感染していることが判明したファイルは自動的に削除されます（オプション）
インストール
ソースからビルド
AWS Lambdaにアップロードするアーカイブをビルドするには、を実行しますmake all。ビルドプロセスは、amazonlinuxDockerイメージを使用して完了 し ます。結果のアーカイブはで構築されbuild/lambda.zipます。このファイルは、以下の両方のLambda関数のためにAWSにアップロードされます。

CloudFormationを介して関連するAWSインフラストラクチャを作成する
cloudformation.yamlディレクトリにあるCloudFormationを使用して、deploy/このプロジェクトの実行に必要なAWSインフラストラクチャをすばやく起動します。CloudFormationは以下を作成します：

アンチウイルス定義を保存するS3バケット。
avUpdateDefinitions3時間ごとにS3バケットのAV定義を更新するというLambda関数が呼び出されます。この関数は、ユーザーの上記のS3バケットにアクセスし、を使用して更新された定義をダウンロードしますfreshclam。
avScannerオブジェクトをスキャンして適切にタグ付けする、新しいS3オブジェクトの作成ごとにトリガーされるLambda関数が呼び出されます。これは十分なメモリで作成されてい1600mbますが、関数のタイムアウトが発生し始めた場合は、このメモリを増やす必要があるかもしれません。以前は、を使用することをお勧めし1024mbましたが、それによってLambdaタイムアウトが発生し始め、このメモリをバンプすることで解決しました。
CloudFormationを実行すると、このスタックに対して2つの入力が要求されます。

BucketType :(privateデフォルト）またはpublic。これは、アンチウイルス定義を保存するS3バケットに適用されます。public他のAWSアカウントがこのバケットにアクセスする必要がある場合にのみ使用することをお勧めします。
SourceBucket：[空でない文字列]。s3://オブジェクトがスキャンされるS3バケットの名前（含まない）。注-これはIAMポリシーを作成するためにのみ使用されます。後で、CloudFormationが出力するIAMポリシーを介してソースバケットを追加/変更できます。
スタックが正常に作成された後、まだ実行する必要のある3つの手動プロセスがあります。

Lambdaコンソールを介してとLambda関数に実行してbuild/lambda.zip作成されたファイルをアップロードします。make allavUpdateDefinitionsavScanner
新しいS3オブジェクトでスキャナー機能をトリガーするには、avScannerLambda関数コンソールに移動し、Configuration-> Trigger-> Add Trigger-> S3の検索に移動し、バケットを選択してを選択しAll object create events、をクリックしますAdd。注-ソースとして複数のバケットを選択した場合、またはCloudFormationパラメーターでソースバケットとは異なるバケットを選択した場合は、これらの新しいバケットを反映するようにIAMロールも編集する必要があります（「ソースバケットの追加または変更」を参照）。 ）。
Lambda関数に移動し、avUpdateDefinitions関数を手動でトリガーして、バケット内の初期Clam定義を取得します（3時間のトリガーが発生するのを待つのではありません）。これを行うには、Testセクションをクリックしてから、オレンジ色のtestボタンをクリックします。関数の実行には数秒かかるはずです。完了するとclam_defs、av-definitionsS3バケットにが表示されます。
ソースバケットの追加または変更
ソースバケットの変更または追加は、AVScannerLambdaRoleIAMロールを編集することによって行われます。より具体的には、そのIAMロールのポリシーのS3AVScanおよび一部。KmsDecrypt

S3イベント
Lambda関数を呼び出すための新しいS3イベントを追加して、追加のバケットのスキャンを設定します。これは、AWSコンソールの任意のバケットのプロパティから実行されます。

s3-イベント

注：オブジェクトメタデータを更新するように構成されている場合、イベントはとに対してのみ構成する必要がありPUTますPOST。メタデータは不変であるため、更新されたメタデータを使用してオブジェクトをそれ自体にコピーする関数が必要です。これにより、正しく構成されていない場合、スキャンの継続的なループが発生する可能性があります。

構成
ランタイム構成は、環境変数を使用して実行されます。以下の表を参照してください。

変数	説明	ディフォルト	必須
AV_DEFINITION_S3_BUCKET	ウイルス対策定義ファイルを含むバケット		はい
AV_DEFINITION_S3_PREFIX	ウイルス対策定義ファイルのプレフィックス	clamav_defs	番号
AV_DEFINITION_PATH	実行時のファイルを含むパス	/ tmp / clamav_defs	番号
AV_SCAN_START_SNS_ARN	スキャン開始に関する通知を公開するSNSトピックARN		番号
AV_SCAN_START_METADATA	スキャンの開始を示すタグ/メタデータ	av-scan-start	番号
AV_SIGNATURE_METADATA	ファイルのAVタイプを表すタグ/メタデータ名	av-signature	番号
AV_STATUS_CLEAN	タグ/メタデータ内のクリーンアイテムに割り当てられた値	綺麗	番号
AV_STATUS_INFECTED	タグ/メタデータ内のクリーンアイテムに割り当てられた値	感染した	番号
AV_STATUS_METADATA	ファイルのAVステータスを表すタグ/メタデータ名	av-status	番号
AV_STATUS_SNS_ARN	スキャン結果を公開するSNSトピックARN（オプション）		番号
AV_STATUS_SNS_PUBLISH_CLEAN	AV_STATUS_CLEANの結果をAV_STATUS_SNS_ARNに公開します	真	番号
AV_STATUS_SNS_PUBLISH_INFECTED	AV_STATUS_INFECTEDの結果をAV_STATUS_SNS_ARNに公開します	真	番号
AV_TIMESTAMP_METADATA	ファイルのスキャン時間を表すタグ/メタデータ名	av-タイムスタンプ	番号
CLAMAVLIB_PATH	ClamAVライブラリファイルへのパス	。/置き場	番号
CLAMSCAN_PATH	ClamAVclamscanバイナリへのパス	./bin/clamscan	番号
FRESHCLAM_PATH	ClamAVfreshclamバイナリへのパス	./bin/freshclam	番号
DATADOG_API_KEY	メトリックをDataDogにプッシュするためのAPIキー（オプション）		番号
AV_PROCESS_ORIGINAL_VERSION_ONLY	S3キーの元のバージョンのみが処理されるように制御します（バケットのバージョン管理が有効になっている場合）	誤り	番号
AV_DELETE_INFECTED_FILES	感染したファイルを自動的に削除するかどうかを制御します	誤り	番号
EVENT_SOURCE	ウイルス対策スキャンイベント「S3」または「SNS」のソース（オプション）	S3	番号
S3_ENDPOINT	S3と対話するときに使用するエンドポイント	なし	番号
SNS_ENDPOINT	SNSとやり取りするときに使用するエンドポイント	なし	番号
LAMBDA_ENDPOINT	Lambdaと対話するときに使用するエンドポイント	なし	番号
S3バケットポリシーの例
「CLEAN」でない場合は、オブジェクトのダウンロードを拒否します
このポリシーでは、次の場合までオブジェクトをダウンロードできません。

Clam-AVを実行するラムダが終了しました（オブジェクトにタグが付いているため）
ファイルがクリーンではありません
cloudtrailでarn：aws：stsを確認してください。イベントを見つけて開き、stsをコピーしてください。以下の形式にする必要があります。

{
     "Effect"：" Deny "、
     "NotPrincipal"：{
         "AWS"：[
             " arn ：aws：iam :: << aws-account-number >>：role / <<bucket-antivirus-role >> "、
             " arn：aws：sts :: << aws-account-number >>：assumed-role / <<bucket-antivirus-role >> / <<bucket-antivirus-role >> "、
             " arn：aws：iam： ：<< aws-account-number >>：root "
        ]
    }、
    "アクション"：" s3：GetObject "、
     "リソース"：" arn：aws：s3 ::: <<bucket-name >> / * "、
     "Condition"：{
         "StringNotEquals"：{
             "s3：ExistingObjectTag / av -ステータス"：" CLEAN "
        }
    }
}
「INFECTED」オブジェクトのダウンロードと再タグ付けを拒否します
{
   "バージョン"：" 2012-10-17 "、
   "ステートメント"：[
    {{
      "Effect"：" Deny "、
       "Action"：[ " s3：GetObject "、" s3：PutObjectTagging " ]、
       "Principal"：" * "、
       "Resource"：[ " arn：aws：s3 ::: <<バケット名>>/* " ]、
       "条件 "：{
         " StringEquals "：{
           " s3：ExistingObjectTag / av-status "：" INFECTED "
        }
      }
    }
  ]
}
バケットを手動でスキャンする
ラムダ関数を設定する前に、以前にスキャンされていないか、作成されたバケット内のすべてのオブジェクトをスキャンすることをお勧めします。これを行うには、scan_bucket.pyユーティリティを使用できます。

pip install boto3
scan_bucket.py --lambda-function-name = < lambda_function_name > --s3-bucket-name = < s3-bucket-to-scan >
このツールは、バケット内で以前にスキャンされていないすべてのオブジェクトをスキャンし、ラムダ関数を非同期で呼び出します。そのため、スキャン結果または失敗を確認するには、cloudwatchログにアクセスする必要があります。さらに、スクリプトはラムダで使用するのと同じ環境変数を使用するため、同様に構成できます。

テスト
このリポジトリには2種類のテストがあります。1つ目はpre-commitテストで、2つ目はpythonテストです。これらのテストはすべてCircleCIによって実行されます。

事前コミットテスト
事前コミットテストは、このリポジトリに送信されたコードがリポジトリの基準を満たしていることを確認します。これらのテストを開始するには、を実行しますmake pre_commit_install。これにより、pre-commitツールがインストールされ、このリポジトリにインストールされます。次に、コードをコミットする前に、githubpre-commitフックがこれらのテストを実行します。

テストを手動で実行するには、make pre_commit_testsまたはを実行しますpre-commit run -a。

Pythonテスト
このリポジトリのPythonテストは、ユーティリティunittestを介して実行されます。noseそれらを実行するには、開発者リソースをインストールしてからテストを実行する必要があります。

pip install -r Requirements.txt
pip install -rrequirements-dev.txtテスト
を行う
ローカルラムダ
ラムダをローカルで実行して、AWSにデプロイせずにラムダが実行していることをテストできます。これは、ラムダと同様に機能するDockerコンテナーを使用することで実現されます。を実行する前に、ファイルにいくつかのローカル変数を設定し、 .envrc.localそれらを適切に変更する必要がありますdirenv allow。お持ちでない場合はdirenv 、でインストールできますbrew install direnv。

Scan lambdaの場合、S3にアップロードされたテストファイルと変数が必要になり、ファイルTEST_BUCKETにTEST_KEY 設定されます.envrc.local。次に、実行できます。

direnv allow
アーカイブスキャンを行う
ウイルスとして認識されるファイルが必要な場合は、EICAR Webサイトからテストファイルをダウンロードして、バケットにアップロードできます。

Updateラムダの場合、次を実行できます。

direnv allow
アーカイブを更新する
ライセンス
Upside Travel, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
ClamAVはGPLバージョン2ライセンスの下でリリースされており、ClamAV のすべてのソースはGithubからダウンロードできます。
