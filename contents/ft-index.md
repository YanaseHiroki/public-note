# MySQL FULLTEXTインデックスの簡単な説明

## まえがき
FULLTEXTインデックスはMySQLの機能の1つですが、調べ始めたとき私にとって簡単な解説記事が見つからなかったため本記事を書くことにしました。<br>
公式ドキュメントを初見で理解できる方は以下リンクからどうぞ。

[MySQL :: MySQL 8.0 リファレンスマニュアル :: 15.6.2.4 InnoDB FULLTEXT インデックス](https://dev.mysql.com/doc/refman/8.0/ja/innodb-fulltext-index.html)

### 免責
概要を理解していただくことを重視するため、「基本的に」や「～など」といった語をつけた方が正しい内容でもあえて省略して、一例だけ説明します。

公式ドキュメントを理解するための踏み台になるように、まずはざっくり理解してもらうことが狙いです。

なんとなく理解できた方は公式ドキュメントに目を通すことを強く推奨いたします(←自戒も込めて記載)。

## 本編
結論からいうと、FULLTEXTインデックスは文字列検索の高速化に役立ちますが導入の際は設定変更が必要かもしれません。 

> FULLTEXT インデックスは、テキストベースのカラム (CHAR、VARCHAR または TEXT カラム) に作成され、それらのカラムに含まれるデータに対するクエリーおよび DML 操作を高速化し、ストップワードとして定義されている単語を省略します。 

引用元：[MySQL :: MySQL 8.0 リファレンスマニュアル :: 15.6.2.4 InnoDB FULLTEXT インデックス](https://dev.mysql.com/doc/refman/8.0/ja/innodb-fulltext-index.html)

足掛かりとして公式ドキュメント冒頭の説明文を解説します：

> FULLTEXT インデックスは、テキストベースのカラム (CHAR、VARCHAR または TEXT カラム) に作成され、

→ カラム(テーブルの列)の型は用途に応じてINT型, DATETIME型などありますが、FULLTEXT インデックスは文字列に対応した型のカラムの検索を高速化するためのものです。

<hr>

> それらのカラムに含まれるデータに対するクエリーおよび DML 操作を高速化し、

→ FULLTEXT インデックスを作成しておくと、カラムの値(文字列)を検索対象としてレコードを抽出する処理を何倍か高速化できます。抽出するときの処理の下準備が済んだ状態を常に維持/最新化しておくイメージです。

<hr>

> ストップワードとして定義されている単語を省略します。 

→ ストップワードとは、英語のインデックスを効率よく作成するためのブラックリストのようなものです。デフォルトでは英単語としてはそこまでユニークな意味を持たない a, the, in, it などの36単語がストップワードとして登録されています。

```
-- 参考：デフォルトのストップワード
mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_FT_DEFAULT_STOPWORD;
+-------+
| value |
+-------+
| a     |
| about |
| an    |

......

| und   |
| the   |
| www   |
+-------+
36 rows in set (0.00 sec)
```
引用元：[MySQL :: MySQL 8.0 リファレンスマニュアル :: 12.10.4 全文ストップワード](https://dev.mysql.com/doc/refman/8.0/ja/fulltext-stopwords.html)

<hr>
### 注意1「ストップワードは不要かもしれません」
たとえばデフォルトのストップワードを用いて"1年A組17番"といった文字列のFULLTEXT インデックスを生成すると、"A"を含むインデックスは生成されずそれ以外の一部を切り取った"組17", "17番", "組17番"しか生成されません。

すると"1年A組"のように'A'を含む値を抽出しようとしてもヒットしない、ということが起きてしまいます。

このように英文以外の値が想定される場合は、ストップワードを用いた機能は不要なため、何も指定しないように設定変更が必要です。

<hr>

### FULLTEXT インデックスの実体と処理
FULLTEXT インデックスの生成は、"1年A組17番"といった文字列の一部を切り取って"組17", "17番", "組17番"のようなパーツに分けて、それぞれがどの位置に存在するかを表形式でまとめておく処理です。

FULLTEXT インデックス自体を設定したときに作成され、対象の値が更新される度に更新されます。

```
-- インデックステーブルの一例
+------------+--------------+-------------+-----------+--------+----------+
| WORD       | FIRST_DOC_ID | LAST_DOC_ID | DOC_COUNT | DOC_ID | POSITION |
+------------+--------------+-------------+-----------+--------+----------+
| 1001       |            5 |           5 |         1 |      5 |        0 |
| after      |            3 |           3 |         1 |      3 |       22 |
| comparison |            6 |           6 |         1 |      6 |       44 |
| configured |            7 |           7 |         1 |      7 |       20 |
| database   |            2 |           6 |         2 |      2 |       31 |
+------------+--------------+-------------+-----------+--------+----------+
```
引用元：[MySQL :: MySQL 8.0 リファレンスマニュアル :: 15.15.4 InnoDB INFORMATION_SCHEMA FULLTEXT インデックステーブル](https://dev.mysql.com/doc/refman/8.0/ja/innodb-information-schema-fulltext_index-tables.html)

<hr>

### 注意2「2文字または1文字でも検索できるようにするためには」
MySQLでInnoDBというストレージエンジンを採用している場合、デフォルト設定では3文字以上のFULLTEXT インデックスしか生成されません。

ストップワードと似た考え方で、英文のFULLTEXT インデックスを生成する場合は2文字以下の文字列で検索できなくていいという意図が感じられます。

2文字または1文字でも検索できるようにするためには、ngram パーサーを有効にした上でngram_token_size オプションを2または1にしてください。
<hr>
→ もう少しわかりやすく書くと……

 デフォルトのパーサー(文字列の区切りを見つけて分割する機能)は英文を想定していて半角スペース、カンマ、ピリオドで区切った単語をインデックスとして生成します。
 <br><br>
しかし英文以外の文字列を想定する場合はこのような区切り方ではなく、以下のように特定の文字数でインデックスを作成するための設定変更が必要かもしれません。
<br><br>
1. まずデフォルトのパーサーではなくngram パーサーを有効にします。
2. そしてngram_token_sizeという設定値を2または1にしてください。<br><br>

こうすることで2文字または1文字のインデックスが生成され、2文字または1文字の検索語で検索が可能になります。

参考：[MySQL :: MySQL 8.0 リファレンスマニュアル :: 12.10.6 MySQL の全文検索の微調整](https://dev.mysql.com/doc/refman/8.0/ja/fulltext-fine-tuning.html)
<hr>

## まとめ
FULLTEXTインデックスは文字列検索の高速化に役立ちますが導入の際は設定変更が必要かもしれません。
<br><br><br><br><br>
<p class="timestamp">初稿：2025/02/09</p>