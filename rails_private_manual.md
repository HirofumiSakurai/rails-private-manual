#背景
Rails はチュートリアルもガイドも実例とストーリーに基づいて記されている。
それはアジャイルな開発の場ではごく普通のことだとは理解できる。
しかし、Railsはプログラムであり、利用する立場できは仕様がマニュアル化されて
いることが再利用性と効率的利用のためには近道だと思う。

公式のそういったドキュメントがないならば、サードパーティーの作った文書に頼る
よりも、自らそれを作成して管理することのほうが信頼性も、学習といった観点でも
良いと思う。

したがって、この文書はあくまで個人的なメモであり、公開を目指すべきものではない。
この文書では、それぞれのRailsの文の書式を整理するだけなので、その意味とか使い方は
Railsガイドの該当する部分を参照してほしい。


#マイグレーション

##マイグレーションの構文

*パターン1 : 普通のパターン。逆転が自動でできる場合。
    class #{マイグレーション名} <AcriveRecord::Migration
      def change
        #{マイグレーション定義} ...
        ...
      end
    end

*パターン2
    class #{マイグレーション名} <AcriveRecord::Migration
      def change
        reversible do |dir|
          #{マイグレーション定義} do |t|
            dir.up {t.change ...}
            dir.down {t.change ...}
          end
        end
      end
    end

*パターン3
    class #{マイグレーション名} <AcriveRecord::Migration
      def up
        #{マイグレーション定義}
      end
      def down
        #{マイグレーション定義}
      end
    end

* マイグレーション定義
  *create_[table|join_table] (, テーブルオプション)do |t| #{カラム指定}
    *テーブルオプションでMySQLのテーブルを指定する例: options: "ENGINE=BLACKHOLE"
    *カラム指定: t.#{型名} #{カラム名} (,カラムオプション)
      *カラムオプション例: null: true
  *change_table do |t| ...
  *drop_[talbe|join_table]
  *rename_table
  *add_[column|index|reference|timestamps|foreign_key]
  *change_[column|column_null|column_default]
  *remove_[column|index|reference|timestamps|foreign_key]
  *execute <<-SQL ..... SQL
  *revert
  *suppress_message, say, say_with_time

* changeメソッドでは以下のマイグレーションで自動的なロールバックがサポートされている。
  *create_[table|join_table]
  *drop_[talbe|join_table]
  *rename_table
  *add_[column|index|reference|timestamps|foreign_key]
  *remove_[column|index|reference|timestamps]
  *change_table : ただし、change, change_default, removeが呼び出されていないこと。

* カラム指定には以下のような記述ができる。
  * t.#{型名} :#{カラム名}
  * t.index #{:キーカラム、または複合キー}
  * t.remove :#{カラム名}
  * t.rename　:#{旧カラム名}, :#{新カラム名}
  * t.timestamps

* 逆転不可能なマイグレーションは downメソッド内にActiveRecord::IrreverseibleMigrationを記述しておく。
* 以前のマイグレーションのロールバックを別のマイグレーションで書きたいときは、revert #{マイグレーション名}を
  changeメソッド内で宣言する。revertにブロックをとることで、過去のマイグレーションの一部のみを逆転することが
  よりスマートに記述できる。

##マイグレーションの生成

*rails generate migration <マイグレーション名> <オプション>
*rails generate model <モデル名> <オプション>

###マイグレーション名
マイグレーション名は Camelパターンで構成された以下の形式で書くと、単なる名前以上の
意味を含むことができる。

* Create#{テーブル名} : テーブルを作成(create_table)
* CreateJoinTable#{テーブル名} : Joinテーブルを作成(create_join_table)
* Add#{カラム名}To#{テーブル名} : テーブルにカラムを追加(add_column)
* Add#{カラム名}RefTo#{テーブル名} : テーブルにReferenceを追加(add_reference)
* Remove#{カラム名}From#{テーブル名} : テーブルからカラムを削除(remove_column)

同時に複数のカラムを操作する場合はカラム名の部分をDetailsとする。
それぞれのマイグレーション名によってどのようなマイグレーションが生成されるかは、railsguides参照。

##オプション
オプション部分には以下のいずれかををいくつかかくことができる。
*カラム名:型名
*カラム名:型名:index
*カラム名:reference
*カラム名:belongs_to
 型名にはカラム修飾子（または型修飾子ともいう？）をつけることができる。
 カラム修飾子は 3.5 カラム修飾子を参照

## rake db:#{コマンド名} とマイグレーションの実行

### migrate
未実行のchangeあるいはupメソッドを順番に実行する。
db:schema:dump を呼び出すのでdb/schema.rbも更新される。
バージョン番号を指定することにより、指定したバージョンまでup/changeを呼ぶか、
あるいはロールバックする。

### rollback
直前のマイグレーションを逆転させる。
STEPを指定されたら複数個のロールバックを一度に実行する。
単にmigrateでバージョン番号を指定する代わりに現在の状態から相対的に戻る指定を可能にしたもの。

### setup
データベースの作成、スキーマの読み込み、シードデータを使用したデータベースの初期化を行う。

### reset
データベースをドロップして再設定する。db:drop db:setup と同じ。

### migrate:up, migrate:down
特定のマイグレーションをup, down する。実行済のときは繰り返さない。

## スキーマダンプ
db/schema.rb に書くテーブルのカラムとその型がまとめられている。
これを直接いじってはいけない。
アプリケーションのデプロイをするときには、その環境でmigrateするよりスキーマをもっていった
ほうが早くて安全。
anote_models gem を使うと、モデルファイルの冒頭にスキーマの要約コメントが自動的に更新される。
スキーマダンプの詳細は railsガイド参照。

## 参照整合性
モデルのvalidatesの:foreign_key, uniqueness: true だけでなくActiveRecordの外部キー制約を使って参照整合性を増大できる。

## シードデータ
初期データ用のコードを db/seeds.rb ファイルに書いて rake db:seed する。

# ActiveRecord でのバリデーション
*valid?, invalid? はバリデーションの結果をtrue/falseで返す。
* create, create!, save, save!, update, update! はバリデーションをトリガする。
  *それぞれのメソッドにvalidate: false属性をつけるとバリデーションをスキップする。
* 一方、[in|de]crement!, [in|de]crement_counter, toggle!, touch,
  update_[all|attribute|column|columns|counters] はバリデーションをスキップする。
* error.messages インスタントメソッドはエラーの内容を示すメッセージを返す。
* errors[:attribue]に各属性のバリデーションのエラーの配列が保存される。

## バリデーションヘルパー
* acceptance - 同意のチェックボックスなどの属性
* validates_associated - 他のモデルも同時にバリデーション。無限ループに注意
* confirmation - パスワードの確認用などの属性
* inclusion - 与えられた集合に値が含まれていることを検証
* exclusion - 与えられた集合に値が含まれていないことを検証
* format - 与えられた正規表現にマッチするか検証
* length - 属性の長さを検証。 以下のオプションが付けられる。
    * :minimum - 属性はこの値より小さな値を取れません。
    * :maximum - 属性はこの値より大きな値を取れません。
    * :in または :within - 属性の長さは、与えられた区間以内でなければなりません。このオプションの値は範囲でなければなりません。
    * :is - 属性の長さは与えられた値と等しくなければなりません。
    * :wrong_length, :too_long, :too_short - エラーメッセージを指定。%{count} が使える。
    * :tokenizer - 文字数以外の（単語数など）方法でカウントするとき使う。
* numericality - 属性がオプションに示された数値であることを検証。以下のオプションがある。
  * :only_integer, :greater_than, :greater_than_or_equal_to, :equal_to, :less_than,
    :less_than_or_equal_to, :odd, :even
* presence - 属性が空でないことを検証
* absence -  属性が空であることを検証
* uniqueness - 属性が一意であることを検証。SQLクエリを使う。scope:, message:, case_senesitive: オプションが付けられる。
* validates_with - バリデーション専用の別のクラスにレコードを渡す。
* validates_each - ブロックをとって検証する。

## 共通のバリデーションオプション
* :allow_nil - 対象の値がnilの場合にバリデーションをスキップ
* :allow_blank - 対象の値がblankの場合にバリデーションをスキップ
* :message - バリデーション失敗時にerrorsコレクションに追加されるカスタムエラーメッセージを指定
* :on - バリデーション実行のタイミングを指定
* :strict - 例外発生
* if:, unless: - 条件付き検証

他に、カスタムバリデーションを書いたり、バリデーションエラーに特別に対応したりすることができる。

# ActiveRecord のコールバック
* [before|after]_validation, [before|after|around]_[save|create|update|destroy]
* after_initialize, after_find, after_touch
* after_[commit|rollback]

## コールバックをトリガするメソッド
* create, create!, [in|de]crement!, destroy, destroy!, destroy_all, save, save!, save(validate:false),
  toggle!, update_attribute, update, update!, valid?

## コールバックをスキップするメソッド
* [in|de]crement, [in|de]crement_counter, delete, delete_all, toggle, update_[column|columns|all|counters]

条件付きコールバックでコールバックを柔軟に利用できる。
また、コールバッククラスでコールバックを複数のモデルで共有できる。

# ActiveRecord の関連付け

## belongs_to
「1対1」のつながり。
belongs_to :#{モデル名} の場合、モデル名に該当するモデルが存在することと、
#{モデル名}_id というinteger の属性が存在することが必要。

## has_one
他方のモデルのインスタンスを「まるごと含んでいる」または「所有している」。

## has_many
h他のモデルとの間の「1対多」(0個以上)のつながり。
has_many関連付けが使用されている場合、「反対側」のモデルではbelongs_toが使用されることが多くあります。
相手のモデル名は「複数形」にする必要があります。

## has_many :through
他方のモデルと「多対多」のつながりを設定する場合によく使われます。
2つのモデルの間に「第3のモデル」(結合モデル)が介在する点が特徴です。

## has_one :through
他のモデルとの間に1対1のつながり。2つのモデルの間に「第3のモデル」(結合モデル)が介在する点が特徴です。

## has_and_belongs_to_many
他方のモデルと「多対多」のつながり。
through:を指定した場合と異なり、第3のモデル(結合モデル)が介在しません(結合用のテーブルは必要)。

## ポリモーフィック関連付け
ある1つのモデルが他の複数のモデルに属していることを、1つの関連付けだけで表現することができます。

## 自己結合
自分自身に関連付けられる必要のあるモデルに使う。
