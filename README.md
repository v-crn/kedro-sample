# Kedro Tutorial

[Kedro starters — Kedro 0.16.6 documentation](https://kedro.readthedocs.io/en/stable/02_get_started/06_starters.html)

## Install

```
pip install kedro
```

## プロジェクト作成

[Creating a new project — Kedro 0.16.2 documentation](https://kedro.readthedocs.io/en/0.16.2/02_getting_started/03_new_project.html)

`kedro new` で作業ディレクトリ上に kedro のプロジェクトフォルダが用意される。

```
kedro new
```

次のようなオプションを付けることもできる。

- config ファイルから作成する: `--config=my_kedro_pyspark_project.yml`
- pyspark を使う: `--starter=pyspark`

この例ではデフォルトの new_proj という名前でプロジェクトを作成する。作成したらその階層に移動しておこう。`kedro run` などのコマンドはこの階層で実行する必要がある。

```
cd new_proj
```

## パッケージのインストール

kedro-sample/new_proj/src/requirements.txt に必要なパッケージを記述した上で次のコマンドを実行すると requirements.in というファイルが生成される。

```
kedro build-reqs
```

続いて requirements.in をもとに実際にパッケージをインストールする。

```
kedro install
```

ちなみに次のコマンドで `build-reqs` と `install` を一括で行うこともできる。

```
kedro install --build-reqs
```

- 使用するパッケージを変更する場合は再度 `build-reqs` から実行する

## データの準備

前処理前のデータは data/01_raw/ に置く。
パイプラインで扱うデータはすべてデータカタログ conf/base/catalog.yml に次のような形式で定義する。

```yml
example_iris_data:
  type: pandas.CSVDataSet
  filepath: data/01_raw/iris.csv
```

## Jupyter 起動

```py
kedro jupyter lab
# もしくは
kedro jupyter notebook
```

## データカタログ

マジックコマンド `%reload_kedro` でデータアクセス用の catalog というグローバル変数が使えるようになる

```py
%reload_kedro

# 読み込み
df = catalog.load("example_iris_data")
# 書き込み
catalog.save('example_iris_data', df)
```

## Configuration

[Configuration — Kedro 0.16.6 documentation](https://kedro.readthedocs.io/en/stable/04_kedro_project_setup/02_configuration.html)

### 設定環境 (configuration environments)

デフォルトでは `conf/base` と `conf/local` の 2 つのフォルダが configration 用の変数を定義する場所として用意されている。`local` は `base` より優先され、他に任意の名前で `conf` 以下に環境設定用のフォルダを用意することもできる。

たとえば `conf/test/` を指定してパイプラインを実行したい場合は次のようにする。

```
kedro run --env=test
```

設定情報はデータカタログ catalog.yml と同様に `yml` 形式で記述する。

parameters.yml

```yml
example_test_data_ratio: 0.2
example_num_train_iter: 10000
example_learning_rate: 0.01
```

## パイプライン

./src/new_proj/pipelines/ 以下には deta_engineering と data_science という 2 つのディレクトリが存在する。

### deta_engineering

ここでは主に前処理を担うパイプラインを定義する。

例： ./pipelines/data_engineering/pipeline.py

```py
from kedro.pipeline import Pipeline, node
from .nodes import split_data


def create_pipeline(**kwargs):
    return Pipeline(
        [
            node(
                func=split_data,
                inputs=["example_iris_data", "params:example_test_data_ratio"],
                outputs=dict(
                    train_x="example_train_x",
                    train_y="example_train_y",
                    test_x="example_test_x",
                    test_y="example_test_y",
                ),
            )
        ]
    )
```

- node はパイプラインの処理単位
- `node()` の引数 `func` には関数を、`inputs` には catalog に定義したデータ名(`str`/`List[str]`/`None`)を、`outpus` には `func` に指定した関数の返り値(`str`/`Dict[str, str]`/`None`)を渡す
- `inputs` 内の `params` は `conf` ディレクトリ内の parameters.yml を参照する
- 上の例では同じ階層の nodes.py にパイプラインで使う関数を用意して呼び出している

./src/new_proj/pipelines/data_engineering/nodes.py

```py
def split_data(data: pd.DataFrame, example_test_data_ratio: float) -> Dict[str, Any]:
    ...(中略)...

    # Split the data to features and labels
    train_data_x = training_data.loc[:, "sepal_length":"petal_width"]
    train_data_y = training_data[classes]
    test_data_x = test_data.loc[:, "sepal_length":"petal_width"]
    test_data_y = test_data[classes]

    # When returning many variables, it is a good practice to give them names:
    return dict(
        train_x=train_data_x,
        train_y=train_data_y,
        test_x=test_data_x,
        test_y=test_data_y,
    )
```

### data_science

主にモデルの学習、予測、結果の出力を担うパイプラインを定義する。

例： ./pipelines/data_science/pipeline.py

```py
from kedro.pipeline import Pipeline, node
from .nodes import predict, report_accuracy, train_model


def create_pipeline(**kwargs):
    return Pipeline(
        [
            node(
                train_model,
                ["example_train_x", "example_train_y", "parameters"],
                "example_model",
            ),
            node(
                predict,
                dict(model="example_model", test_x="example_test_x"),
                "example_predictions",
            ),
            node(report_accuracy, ["example_predictions", "example_test_y"], None),
        ]
    )
```

### テスト

コマンドラインで`kedro test`を実行するだけで手軽にパイプラインのエラーチェックができる。

```console
kedro test
```

### パイプライン実行

[Create a pipeline — Kedro 0.16.6 documentation](https://kedro.readthedocs.io/en/latest/03_tutorial/04_create_pipelines.html#partial-pipeline-runs)

テストで問題がなければ実行しよう。

```console
kedro run
```

#### 一部のパイプラインのみ実行したい場合

```console
kedro run --pipeline=data_engineering
```

#### 並列処理で実行

```console
kedro run --parallel
```

## パイプラインの可視化

[Visualise pipelines — Kedro 0.16.6 documentation](https://kedro.readthedocs.io/en/stable/03_tutorial/06_visualise_pipeline.html)

### kedro-viz のインストール

```
pip install kedro-viz
```

### 可視化

`kedro viz` を実行し、`http://127.0.0.1:4141` にアクセスするとパイプラインを可視化した画面が表示される。

```
kedro viz
```

### 可視化したパイプライン構成を json ファイルとして保存

```console
kedro viz --save-file my_shareable_pipeline.json
```

### 保存されたパイプライン構成の json ファイルを kedro viz で表示

```
kedro viz --load-file my_shareable_pipeline.json
```
