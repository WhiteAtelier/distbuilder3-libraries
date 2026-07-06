# distbuilder3 libraries

distbuilder3 がハンドリングするライブラリビルドについて、その手順を記したレシピファイルがまとまっています。

## 基本的な概念

### 型

distbuilder3 では、次の4つの型を扱います。

| 型名      | 説明         |
| --------- | ------------ |
| `string`  | 文字列型     |
| `boolean` | 真偽値型     |
| `integer` | 整数型       |
| `double`  | 浮動小数点型 |

それぞれの型は相互変換が可能です。

| from    |     | to string        | to boolean | to integer    | to double     |
| ------- | --- | ---------------- | ---------- | ------------- | ------------- |
| string  | ->  | PT               | \*RULE1    | `std::stoll`  | `std::stod`   |
| boolean | ->  | `ON` / `OFF`     | PT         | `1` / `0`     | `1.0` / `0.0` |
| integer | ->  | `std::to_string` | `!=0`      | PT            | `static_cast` |
| double  | ->  | `std::to_string` | `!=0.0`    | `static_cast` | PT            |

PT は Pass Through、変換なしを表します。

\*RULE1 について、 `false` と評価される値以外は、すべて `true` とします。
`false` と評価される値は、**CASE INSENSITIVE** で、 `""` (空文字)、 `"false"`、`"off"`、`"0"`、`"no"` です。これ以外の文字列はすべて `true` 評価されます。

### 展開可能な文字列

`"${VARIABLE_NAME}"` と記述することで、変数を展開することができます。

変数は、動的に定義されるものもありますが、常に利用可能な変数は、次の通りです。

| Variable                 | Type    | Description                                                    |
| ------------------------ | ------- | -------------------------------------------------------------- |
| `${version}`             | string  | Entry 内にて、そのエントリーで扱うバージョン文字列             |
| `${previous}`            | string  | Entry 内にて、その直前のエントリーで定義されたバージョン文字列 |
| `${author}`              | string  | リポジトリの author 名                                         |
| `${repo}`                | string  | リポジトリの名前                                               |
| `${repository}`          | string  | リポジトリの名前 (= `${repo}`)                                 |
| `${generator}`           | string  | 使用される CMake のジェネレータ名                              |
| `${arch}`                | string  | 使用される CMake のアーキテクチャ名                            |
| `${architecture}`        | string  | 使用される CMake のアーキテクチャ名 (= `${arch}`)              |
| `${is_windows}`          | boolean | Windows かどうかのフラグ                                       |
| `${is_macos}`            | boolean | macOS かどうかのフラグ                                         |
| `${is_linux}`            | boolean | Linux かどうかのフラグ                                         |
| `${platform}`            | string  | プラットフォーム名 (`windows`/`macos`/`linux`)                 |
| `${options.OPTION_NAME}` | Any     | Entry 内で定義されたオプションの値                             |

また、ビルドステップ内において、追加で利用可能な変数は、次の通りです。

| Variable              | Type   | Description                                        |
| --------------------- | ------ | -------------------------------------------------- |
| `${install_root_dir}` | string | インストール先のルートディレクトリ (config 由来)   |
| `${build_root_dir}`   | string | ビルド用のルートディレクトリ (config 由来)         |
| `${cmake_executable}` | string | CMake の実行ファイルパス (config 由来)             |
| `${working_dir}`      | string | 現在のビルド作業ディレクトリ                       |
| `${install_dir}`      | string | 現在のインストールディレクトリ                     |
| `${cxx_standard}`     | string | ビルドで使う C++ 標準のバージョン (deps.toml 由来) |

#### Post Processor

この展開可能な変数については、その変数を展開するときに、いくつかのポストプロセスを実施することができます。ポストプロセスは `${VARIABLE_NAME:PP1,PP2,...}` のように、変数名とコロンで区切って、カンマ繋ぎで指定します。

変数の値を、指定されたポストプロセスの Input Type に変換したあと、実施します。ポストプロセスが連結されている場合は、出力された型を、次のポストプロセスの Input Type に変換して実施します。

| Post Processor | Input Type | Output Type | Description                              |
| -------------- | ---------- | ----------- | ---------------------------------------- |
| `upper`        | string     | string      | 文字列を大文字に変換します               |
| `lower`        | string     | string      | 文字列を小文字に変換します               |
| `str`          | Any        | string      | 前述したルールに従って文字列に変換します |
| `not`          | boolean    | boolean     | 真偽値を反転します                       |

## レシピファイルの書き方

### ファイル名

レシピファイルは、必ず `{author}.{repo}.json[c]` という名前で作成してください。
`json` ファイルでもよいですが、コメントを残せるため `jsonc` ファイルを推奨します。
distbuilder3 は、このファイル名を基に検索を行います。

distbuilder3 のワークフローでは、ライブラリ名は `{author}.{repo}` という形式で記述します。
以降のドキュメントでは `<RepoID>` と表記します。

### 基本形

リポジトリへのリンクと、バージョン定義のリストを記述します。
ここでのバージョン定義順は非常に重要です。
distbuilder3 は、バージョン文字列を semver 形式として**解釈しません**。
`entries` 配列の定義順は、必ず**古い順**に定義する必要があります。
後述する Version Range は、この定義順を基に解釈されます。

```jsonc
{
    // リポジトリへのリンク
    "repository": "https://path/to/repository",

    "entries": [
        // List
        // -- [<Entry (Base)>, (<Entry (Base)>|<Entry (Inherit)>)*]
        // -- 先頭はか必ず Entry (Base)
        // -- 二つ目以降は Entry (Inherit) も利用可能です
    ],
}
```

### Entry (Base)

ひとつのバージョンに関する情報を記述します。

先頭のバージョンは必ず Base となります。

```jsonc
// 以下 Entry の内容です
{
    // バージョン番号
    // -- 他から、このバージョンを指定したいときに使用されるキーになります
    // -- ライブラリのタグ名に揃えると良いでしょう
    "version": "1.0.0",

    // オプション
    // -- このライブラリをビルドするときに使用される、オプションのインターフェースです
    // -- この部分は deps.toml にてオーバーライド指定ができます
    // -- このオプションは、 `${options.OPTION_NAME}` として展開が可能になります
    // -- 必要がない場合は省略可能です
    "options": {
        // <Option Name>: <Option>
        "shared": false,
        "use_zlib": false,
        "use_zstd": true,
    },

    // ビルド時の依存
    // -- このライブラリをビルド、利用するときに必要な依存ライブラリを列挙します
    // -- 依存がない場合は省略可能です
    "dependencies": {
        // <RepoID>: <Dependency>
        "facebook.zstd": {
            // <Dependency>
        },
        "gabime.zlib": {
            // <Dependency>
        },
    },

    // ビルドステップ
    // -- このライブラリをビルドするための手順を記述します
    "steps": {
        // "order": step の順序
        // -- この配列に、ステップのキー名を順番に記述することで、ビルドステップの順序を指定します
        // -- この配列に記載されなかったステップは、（定義はあったとしても）実行されません
        // -- "order" は予約キーなので、ステップのキー名としては使用できません
        "order": [
            // List of <Step ID>
        ],

        // Step 定義
        // <Step ID>: <Step>
        // -- step のキー名とともに、ステップを定義します
        "step_name": {
            // Step
        },
    },

    // ライセンスファイル
    // -- install した結果、生成される License ファイルへの、 install_dir からの相対パスを指定します。
    // -- 省略できません。
    // -- ライセンスファイルが存在しない場合は、明示的に "!NoLicense" と指定します。
    "license": "<License File Name>",
}
```

### Entry (Inherit)

2つ目以降のバージョン定義では、それ以前のバージョンを継承して定義することができます。

```jsonc
{
    // このエントリーで定義するバージョン
    // -- 省略できません。
    "version": "1.0.1",

    // 継承元のバージョン
    // -- 継承するバージョン文字列.
    // -- 直前のバージョンを継承する場合は "${previous}" が利用できます
    "inherit": "${previous}",

    // 以降は、Base をオーバーライドする目的で記述します

    // Option
    // -- オーバーライドするものがなければ省略可能です
    "options": {
        // 改めて同じキー名で定義することで、値を上書きします
        "shared": true,

        // null を指定することで、継承元の定義を削除できます
        // -- このバージョンでは "${options.use_zlib}" は利用できません
        "use_zlib": null,

        // 省略した use_zstd は、継承元の定義がそのまま利用されます
    },

    // Dependency
    "dependencies": {
        // null を指定すると、継承元の定義を削除、つまり依存をなくすことができます
        "gabime.zlib": null,

        // 省略した依存は、継承元の定義がそのまま利用されます

        // 改めて同じキー名で定義することで、値を上書きします
        // -- ただしすべてのキーを再定義する必要はありません
        // -- 詳細は Dependency の項で解説します
        "facebook.zstd": {
            // <Dependency>
        },
    },

    // Step
    "steps": {
        // order をオーバーライドしたい場合は、改めて order を定義します
        // -- 手順を省略したり、順番を入れ替えたりできます
        // -- 省略すれば、継承元の定義がそのまま利用されます
        "order": [
            // List of <Step ID>
        ],

        // 省略した step は、継承元の定義がそのまま利用されます

        // オーバーライドしたい step は、そのキー名で再度定義します
        // -- ただしすべてのキーを再定義する必要はありません
        // -- 詳細は Step の項で解説します
        "step_name": {
            // <Step>
        },
    },

    // License
    // -- 設定を上書きしたい場合にのみ、改めて定義します。
    "license": "<New License File Name>",
}
```

### Option

オプション名と、前述した4つの型の値とともに、オプションを定義することができます。
ここで指定している値はデフォルト値となります。
ビルドに関するオプションのインターフェースです。
このオプションの値はそのままビルドステップ中に参照される変数となります。

```jsonc
{
    "shared": false,
    "use_zlib": false,
    "use_zstd": "${is_windows}",
    "kinds": 10,
    "identifier": "foobar",
    "hogehoge": null,
}
```

`use_zstd` のように、展開可能な変数を指定することもできますが、ほかのオプションの値を参照することはできません。（ここでは `${options.xxx}` が利用できません）

`null` を指定した場合は、 `inherit` で継承したバージョンのオプション定義を削除します。

以下のように、deps.toml でオーバーライドすることができます。オーバーライドがないものは、デフォルト値が利用されます。

```toml
[authorName.repoName]
version = "1.0.1"  # 上記の例が 1.0.1 の定義だったとします。
options.shared = true  # 上書き
options.kinds = 20     # 上書き

# identifier は、デフォルト値は文字列ですが、boolean を指定することもできます。
# （この identifier を文字列として使う時には、ルールに従って文字列 "true" となります。）
options.identifier = true

# options.use_zlib は指定がないため、 false に解決されます。
# options.use_zstd は指定がないため、 "${is_windows}" に解決されます。

# options.hogehoge は、このバージョンでは削除されているため指定できません。
```

### Dependency

依存するライブラリを `{author}.{repo}` をキーとして、オブジェクトで定義します。

以下全てにおいて、 `Entry (Inherit)` 内では `null` を指定することで、継承元の定義を削除することができます。

```jsonc
{
    // 依存するバージョン
    // -- 省略可能で、指定する場合はリストか、文字列で指定します。
    // -- 省略した場合は、 `["..."]` と同等です。
    // -- 文字列でバージョンを指定した場合は、 `["指定したバージョン"]` と同等です。
    // -- 依存バージョンを配列で指定し、その範囲を持たせることができます。
    // -- Version Range については、後述の Version Range の項を参照してください。
    "version": [
        // <Version Range>
    ],

    // 依存の条件式
    // -- この依存を有効にする条件式を記述します
    // -- 条件式が true の場合にのみ、この依存が有効になります
    "condition": {
        // <Condition>
    },

    // 依存先に強制するオプションの値
    // -- 上書き指定するべきものだけ記述します
    // -- deps.toml に指定がある場合は、そちらが最優先されます
    // -- 純粋な Option 型とは異なり、 Condition 評価を持つことができます
    // -- 詳細については、後述の Option with Condition の項を参照してください
    "options": {
        // <Option Name>: <Option with Condition>
    },
}
```

### Version Range

対象バージョンを範囲で指定できます。
特殊なバージョン文字列として `"..."` があり、（閉）区間を表します。
末尾にあると、直前のバージョンを含む、以降全てのバージョンを意味します。
先頭にあると、直後のバージョンを含む、以前全てのバージョンを意味します。
バージョンの間にあると、前後のバージョンを含む、その間のバージョンを意味します。
複数の範囲を連結することもできます。

TODO: 開区間も実装するといいかも。 "..^" (= [A, B)) みたいな？何かいい文字の案ないかな

| Version Range                  | Description                                                                |
| ------------------------------ | -------------------------------------------------------------------------- |
| `[]`                           | 空配列は、充足するバージョンがないことを意味します。絶対に解決できません。 |
| `["..."]`                      | すべてのバージョンが利用可能です。                                         |
| `["1.0"]`                      | 1.0 のみが利用可能です。                                                   |
| `["2.0", "3.0"]`               | 2.0 または 3.0 のいずれかが利用可能です。                                  |
| `["...", "2.0"]`               | 2.0 を含む、2.0 以前のすべてのバージョンが利用可能です。                   |
| `["1.0", "..."]`               | 1.0 を含む、1.0 以降のすべてのバージョンが利用可能です。                   |
| `["1.0", "...", "3.0"]`        | 1.0、3.0 を含む、 1.0 以降、 3.0 以前のバージョンが利用可能です。          |
| `["1.0", "2.0", "...", "3.0"]` | 1.0 と、2.0、3.0 を含む、 2.0 以降 3.0 以前のバージョンが利用可能です。    |
| `["...", "0.9", "2.0", "..."]` | 0.9 以前、2.0 以降のすべてのバージョンが利用可能です。                     |

### Condition

条件式を定義します。
Condition ブロックは、`true` か `false` に評価されます。

また、 Entry (Inherit) 内における上書き挙動については、 Condition に関しては**全体を置き換える**ことになります。ほかのオーバーライド可能な要素は、特定のキーについてピンポイントで上書きができますが、 Condition については、継承元の定義を削除して、新しい定義に置き換えることになります。上書きする場合は、 Condition ブロック全体を再定義してください。

#### 基本的な構文

```jsonc
{
    "op": "<Operator>",
}
```

Condition ブロックは、必ず `op` キーを持つオブジェクトです。そしてこの `Operator` の種類によって、残りのキーが規定されます。

#### Operator

| Operator  | Description                                                                                    |
| --------- | ---------------------------------------------------------------------------------------------- |
| `isTrue`  | `value` を boolean として評価し、`true` かどうかを判定します。                                 |
| `isFalse` | `value` を boolean として評価し、`false` かどうかを判定します。                                |
| `and`     | `values` の要素すべてを boolean として評価し、すべてが `true` であるかどうかを判定します。     |
| `or`      | `values` の要素のすべてを boolean として評価し、いずれかが `true` であるかどうかを判定します。 |
| `equal`   | `lhs` と `rhs` の値が等しいかどうかを判定します。                                              |

例えば:

```jsonc
{
    "op": "equal",
    "lhs": "windows",
    "rhs": "${platform}",
}
```

#### 省略型

```jsonc
{
    "condition": "${options.hoge}",
}
```

Condition ブロックに文字列が指定されている場合は、 `isTrue` の省略型として解釈されます。

```jsonc
{
    "condition": {
        "op": "isTrue",
        "value": "${options.hoge}",
    },
}
```

#### 入れ子

Condition ブロックは入れ子にすることができます。先述した通り、 Condition ブロックは boolean 判定されて、親の値に使われます。

```jsonc
{
    "op": "and",
    "values": [
        "${options.hoge}",
        {
            "op": "or",
            "values": ["${options.fuga}", "${options.piyo:not}"],
        },
    ],
}
```

この例では、 `(options.hoge && (options.fuga || !options.piyo))` という条件式を表しています。

### Option with Condition

`<Dependency>` における `options` ブロックでは、その依存先ライブラリが持っているオプションインターフェースに対して、値を指定することができます。

この時、このオプション指定には、前述した Condition ブロックによる条件式を持たせることができます。条件が `false` 評価された場合には、そのオプション指定は無効となり、依存先ライブラリのデフォルト値が利用されます。

```jsonc
{
    "options": {
        "shared": {
            "value": true,
            "condition": "${is_linux}",
        },
    },
}
```

この例では、依存先ライブラリの `shared` オプションを、Linux の場合にのみ `true` に設定することを意味します。
Linux 以外のプラットフォームでは、このオプション指定は無効となり、依存先ライブラリのデフォルト値が使用されます。

### Step

ビルドプロセスにおいて、ひとつの手順を定義します。ステップは、必ず `cmd` キーを持つオブジェクトです。そしてこの `cmd` の種類によって、固有のキーが規定されます。

また、 Step 実行中は、ライブラリ、オプション含めたなかで固有の作業ディレクトリが作成され、カレントディレクトリとなります。変数経由では `${working_dir}` として取得可能です。

継承 Entry においては、この Step 内のキーについても、単独のキーで上書きすることができます。上書きされなかったキーについては、継承元の定義がそのまま利用されます。

```jsonc
{
    "cmd": "<Step Type>",
}
```

#### Commands

以下に、利用可能なコマンドの一覧を示します。

| Command             | Description                                                           |
| ------------------- | --------------------------------------------------------------------- |
| `download`          | 指定された URL から、ファイルをダウンロードします。                   |
| `extract`           | 指定されたアーカイブファイルを展開します。                            |
| `cmake-configure`   | CMake の configure を実行します。                                     |
| `cmake-install-all` | CMake の build / install を実行します。                               |
| `install-file`      | 指定されたファイルをインストールします。                              |
| `github-download`   | GitHub から、指定されたバージョンのソースをダウンロード、展開します。 |

##### download

`curl` コマンドを利用して、指定された URL からファイルをダウンロードします。

| Key               | Type     | Default     | Description                                   |
| ----------------- | -------- | ----------- | --------------------------------------------- |
| url               | string   | (required)  | ダウンロードする URL                          |
| output            | string   | (required)  | ダウンロードしたファイルの出力先              |
| hash              | string   | (required)  | ダウンロードしたファイルの SHA-256 ハッシュ値 |
| access_token_site | string   | `""`        | アクセストークンのサイト文字列                |
| access_token_key  | string   | `"default"` | アクセストークンのキー文字列                  |
| extra_args        | string[] | `[]`        | `curl` コマンドに追加で渡す引数のリスト       |

注意: version3.2.1 現在、 extra_args は非対応です。TODO

アクセストークンについては、 distbuilder3 の Config についてのドキュメントを参照してください。

#### extract

アーカイブの回答は `tar` コマンドを利用して行います。

| Key       | Type    | Default             | Description                                           |
| --------- | ------- | ------------------- | ----------------------------------------------------- |
| input     | string  | (required)          | 展開するアーカイブファイルのパス                      |
| output    | string  | `"."` = working_dir | 展開先のディレクトリパス                              |
| verbosity | boolean | `false`             | 冗長なログ出力を行うかどうか                          |
| error_ok  | boolean | `false`             | `tar` がコード 1 で終了した場合、エラーとして扱わない |

展開する zip に、しばしば simlink が含まれている場合があります。Windows では、権限の問題でシンボリックリンクの展開ができないことがあるため、ファイルをすべて展開後に、失敗が報告されることがあります。その場合は、 `error_ok` を `true` に設定してください。

#### cmake-configure

CMake の configure を実行します。

| Key           | Type   | Default                | Description                                     |
| ------------- | ------ | ---------------------- | ----------------------------------------------- |
| source_dir    | string | `"src"`                | CMake の source ディレクトリ                    |
| build_dir     | string | `"build"`              | CMake のビルドディレクトリ                      |
| debug_postfix | string | `"d"`                  | Multi-Config 時の Debug ビルドの postfix 文字列 |
| configs       | string | `["Debug", "Release"]` | Multi-Config 時のビルド構成のリスト             |
| args          | Args   | (required)             | CMake に渡す引数のリスト                        |

Args については、いくつかの書き方ができます。

```jsonc
{
    "args": {
        "<Arg Set ID>": {
            "args": [
                // Strings...
            ],
            "condition": {
                // <Condition>
            },
        },
    },
}
```

こちらが基本形です。 "args" の下に、 `<Arg Set ID>` をキーにしたオブジェクトを作成します。 `"args"` キーには、文字列に評価される値の配列で、 CMake に渡す引数になります。
`<Arg Set ID>` については、 `Entry (Inherit)` 内で上書きする際に、同じキー名で定義することで、継承元の定義を削除して、新しい定義に置き換えることができます。

また `"condition"` に関しては、省略可能、デフォルト `true` である Condition ブロックです。
ここに Condition ブロックを定義することで、条件式が `true` の場合にのみ、この Arg Set が有効になります。条件式が `false` の場合は、この Arg Set は無効となり、 CMake に渡されません。

```jsonc
{
    "args": {
        "<Arg Set ID>": [
            // Strings...
        ],
    },
}
```

この形が、condition を省略できる場合の省略形です。 `<Arg Set ID>` の下に、文字列の配列を直接定義することができます。 Condition ブロックが省略され、 `true` と評価されるため、この引数は常に CMake に渡されます。

```jsonc
{
    "args": [
        // Strings...
    ],
}
```

この形が、 `<Arg Set ID>` を省略できる場合の省略形です。`<Arg Set ID>` を省略すると、代わりに `"default` という ID が採用されます。 `"args"` の下に、文字列の配列を直接定義することができます。 Condition ブロックが省略され、 `true` と評価されるため、この引数は常に CMake に渡されます。
後続の Entry にて、この args を上書きしたい場合は、同じ記法で上書きするか、 `<Arg Set ID>` に `default` を指定して上書きすることができます。

#### cmake-install-all

CMake の build / install を実行します。

| Key       | Type     | Default                | Description                                                     |
| --------- | -------- | ---------------------- | --------------------------------------------------------------- |
| build_dir | string   | `"build"`              | CMake のビルドディレクトリ                                      |
| configs   | string[] | `["Debug", "Release"]` | Multi-Config 時のビルド構成のリスト。この順番でビルドされます。 |

注意: version3.2.1 現在、 `build_dir` は必須項目です。TODO: デフォルト値 `build` でよいのでは

#### install-file

カレントディレクトリの中から、インストールディレクトリへファイルをコピーします。

| Key             | Type   | Default    | Description                |
| --------------- | ------ | ---------- | -------------------------- |
| source_file     | string | (required) | コピー元のファイルパス     |
| destination_dir | string | (required) | コピー先のディレクトリパス |

コピー先のディレクトリパスは、存在しない場合は自動的に作成されます。
`source_file` はカレントディレクトリが相対パスの起点であり、 `destionation_dir` はインストールディレクトリ (= `${install_dir}`) が相対パスの起点となります。

#### github-download

GitHub のプロジェクトを取得するケースに特化したコマンドです。ダウンロードから展開までを 1Step に定義しています。

| Key              | Type    | Default       | Description                                                 |
| ---------------- | ------- | ------------- | ----------------------------------------------------------- |
| output_dir       | string  | `"src"`       | 展開先のディレクトリパス                                    |
| ref              | string  | (required)    | 取得するリポジトリのリファレンス (ブランチ、タグ、コミット) |
| hash             | string  | (required)    | 取得したアーカイブの SHA-256 ハッシュ値                     |
| author           | string  | `"${author}"` | GitHub の author 名                                         |
| repo             | string  | `"${repo}"`   | GitHub のリポジトリ名                                       |
| archive_path     | string  | `"src.zip"`   | GitHub から取得するアーカイブの配置パス                     |
| access_token_key | string  | `"default"`   | アクセストークンのキー文字列                                |
| error_ok         | boolean | `false`       | `curl` がコード 1 で終了した場合、エラーとして扱わない      |
