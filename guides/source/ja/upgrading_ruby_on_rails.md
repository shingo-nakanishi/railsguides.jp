﻿
Rails アップグレードガイド
===================================

本章では、アプリケーションで使用されているRuby on Railsのバージョンを、新しいバージョンにアップグレードする際の手順について示します。アップグレードの手順は、Railsのバージョンごとに個別に記載されています。

--------------------------------------------------------------------------------

一般的なアドバイス
--------------

言うまでもないことですが、既存のアプリケーションをアップグレードする際には、何のためにアップグレードするのかをはっきりさせておく必要があります。新しいバージョンのうちどの機能が必要になるのか、既存のコードのサポートがどのぐらい困難になるのか、アップグレードに必要な時間とスキルはどれほど必要かなど、いくつもの要素を調整しなければなりません。

### テスティングのカバレッジ

アップグレード後にアプリケーションが正常に動作していることを確認する方法としては、良いテストカバレッジをアップグレード前に準備しておくのが最善です。アプリケーションを一気に検査する自動テストがないと、変更点をすべて手動で確認しなければならず膨大な時間がかかってしまいます。Railsのようなアプリケーションの場合、これはアプリケーションのあらゆる機能を一つ残らず確認しなければならないということです。アップグレードの実施は、テストカバレッジをきちんと準備してから行なうよう、お願いいたします。

### Rubyのバージョン

Railsは、そのバージョンがリリースされた時点で最新のバージョンのRubyに依存しています。

* Rails 3以上では、Ruby 1.8.7以降が必須です。これより古いRubyのサポートは公式に停止しています。できるだけ早くアップグレードをお願いします。
* Rails 3.2.xはRuby 1.8.7の最終ブランチです。
* Rails 4ではRuby 2.0が推奨されます。Ruby 1.9.3以上が必須です。

TIP: Ruby 1.8.7 p248およびp249にはRailsをクラッシュさせるマーシャリングバグがあります。Ruby Enterprise Editionでは1.8.7-2010.02以降このバグは修正されています。Ruby 1.9系を使用する場合、Ruby 1.9.1はあからさまなセグメンテーション違反が発生するため使用できません。1.9.3をご使用ください。

### Rakeタスク

Railsには`rails:update`というrakeタスクがあります。Gemfileに記載されているRailsのバージョンを更新後、このrakeタスクを実行してください。
これにより、新しいバージョンでのファイル作成や既存ファイルの変更を対話形式で行なうことができます。

```bash
$ rake rails:update
   identical  config/boot.rb
       exist  config
    conflict  config/routes.rb
Overwrite /myapp/config/routes.rb? (enter "h" for help) [Ynaqdh]
       force  config/routes.rb
    conflict  config/application.rb
Overwrite /myapp/config/application.rb? (enter "h" for help) [Ynaqdh]
       force  config/application.rb
    conflict  config/environment.rb
...
```

予期しなかった変更が発生した場合は、必ず差分を十分にチェックしてください。

Rails 4.1からRails 4.2へのアップグレード
-------------------------------------

### Web Console gem

最初に、Gemfileの`development`グループに`gem 'web-console', '~> 2.0'`を追加し、`bundle install`を実行してください (このgemはRailsを過去のバージョンからアップグレードした場合には含まれないので、手動で追加する必要があります)。gemのインストール完了後、`<%= console %>`などのコンソールヘルパーへの参照をビューに追加するだけで、どのビューでもコンソールを利用できるようになります。このコンソールは、development環境のビューで表示されるすべてのエラーページにも表示されます。

### Responders gem

`respond_with`およびクラスレベルの`respond_to`メソッドは、`responders` gemに移転しました。これらのメソッドを使用したい場合は、Gemfileに`gem 'responders', '~> 2.0'`と記述するだけで利用できます。今後、`respond_with`呼び出し、およびクラスレベルの`respond_to`呼び出しは、`responders` gemなしでは動作しません。

```ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController
  respond_to :html, :json

  def show
    @user = User.find(params[:id])
    respond_with @user
  end
end
```

インスタンスレベルの`respond_to`は今回のアップグレードの影響を受けませんので、gemを追加する必要はありません。

```ruby
# app/controllers/users_controller.rb

class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    respond_to do |format|
      format.html
      format.json { render json: @user }
    end
  end
end
``` 

詳細については[#16526](https://github.com/rails/rails/pull/16526)を参照してください。

### トランザクションコールバックのエラー処理

現在のActive Recordでは、`after_rollback`や`after_commit`コールバックでの例外を抑制しており、例外時にはログ出力のみが行われます。次のバージョンからは、これらのエラーは抑制されなくなりますのでご注意ください。
今後は他のActive Recordコールバックと同様のエラー処理を行います。

`after_rollback`コールバックや`after_commit`コールバックを定義すると、この変更にともなう非推奨警告が表示されるようになりました。この変更内容を十分理解し、受け入れる準備ができているのであれば、`config/application.rb`に以下の記述を行なうことで非推奨警告が表示されないようにすることができます。

    config.active_record.raise_in_transactional_callbacks = true

詳細については、[#14488](https://github.com/rails/rails/pull/14488)および[#16537](https://github.com/rails/rails/pull/16537)を参照してください。

### テストケースの実行順序

Rails 5.0のテストケースは、デフォルトでランダムに実行されるようになる予定です。この変更に備えて、テスト実行順を明示的に指定する`active_support.test_order`という新しい設定オプションがRails 4.2に導入されました。このオプションを使用すると、たとえばテスト実行順を現行の仕様のままにしておきたい場合は`:sorted`を指定したり、ランダム実行を今のうちに導入したい場合は`:random`を指定したりすることができます。

このオプションに値が指定されていないと、非推奨警告が表示されます。非推奨警告が表示されないようにするには、test環境に以下の記述を追加します。

```ruby
# config/environments/test.rb
Rails.application.configure do
  config.active_support.test_order = :sorted # `:random`にしてもよい
end
```

### シリアル化属性

`serialize :metadata, JSON`などのカスタムコーダーを使用している場合に、シリアル化属性 (serialized attribute) に`nil`を割り当てると、コーダー内で`nil`値を渡すのではなく、データベースに`NULL`として保存されるようになりました (`JSON`コーダーを使用している場合の`"null"`など)。

### Productionログのレベル

Rails 5のproduction環境では、デフォルトのログレベルが`:info`から`:debug`に変更される予定です。現在のログレベルを変更したくない場合は`production.rb`に以下の行を追加してください。

```ruby
# `:info`を指定すると現在のデフォルト設定が使用され、
# `:debug`を指定すると今後のデフォルト設定が使用される
config.log_level = :info
```

### Railsテンプレートの`after_bundle`

Railsテンプレートを使用し、かつすべてのファイルを (Gitなどで) バージョン管理している場合、生成されたbinstubをバージョン管理システムに追加できません。これは、binstubの生成がBundlerの実行前に行われるためです。

```ruby
# template.rb
generate(:scaffold, "person name:string")
route "root to: 'people#index'"
$ rake db:migrate

git :init
git add: "."
git commit: %Q{ -m 'Initial commit' }
```

この問題を回避するために、`git`呼び出しを`after_bundle`ブロック内に置くことができるようになりました。こうすることで、binstubの生成が終わってからBundlerが実行されます。

```ruby
# template.rb
generate(:scaffold, "person name:string")
route "root to: 'people#index'"
rake("db:migrate")

after_bundle do
  git :init
  git add: "."
  git commit: %Q{ -m 'Initial commit' }
end
```

### RailsのHTMLサニタイザ

アプリケーションでHTMLの断片をサニタイズする方法に新しい選択肢が1つ増えました。従来の伝統的なHTMLスキャンによるサニタイズは公式に非推奨化されました。現在推奨される方法は[`Rails HTMLサニタイザ`](https://github.com/rails/rails-html-sanitizer)です。

これにより、`sanitize`、`sanitize_css`、`strip_tags`、および`strip_links`メソッドは新しい実装に基いて動作するようになります。

新しいサニタイザは、内部で[Loofah](https://github.com/flavorjones/loofah)を使用しています。そしてLoofahはNokogiriを使用しています。Nokogiriで使用されているXMLパーサーはCとJavaの両方で記述されているので、使用しているRubyのバージョンにかかわらずサニタイズが高速化されるようになりました。

新しいRailsでは`sanitize`メソッドが更新され、`Loofah::Scrubber`を使用して強力なスクラブを行なうことができます。
[スクラブの使用例はここを参照](https://github.com/flavorjones/loofah#loofahscrubber)。

`PermitScrubber`および`TargetScrubber`という2つのスクラバーが新たに追加されました。
詳細については、[gemのReadme](https://github.com/rails/rails-html-sanitizer)を参照してください。

`PermitScrubber`および`TargetScrubber`のドキュメントには、どの要素をどのタイミングで除去すべきかを完全に制御する方法が記載されています。

従来のままのサニタイザの実装が必要な場合は、アプリケーションのGemfileに`rails-deprecated_sanitizer`を追加してください。

```ruby
gem 'rails-deprecated_sanitizer'
```

### RailsのDOMのテスト

`assert_tag`などを含む[`TagAssertions`モジュール](http://api.rubyonrails.org/classes/ActionDispatch/Assertions/TagAssertions.html)は[非推奨](https://github.com/rails/rails/blob/6061472b8c310158a2a2e8e9a6b81a1aef6b60fe/actionpack/lib/action_dispatch/testing/assertions/dom.rb)になりました。今後推奨されるのは、ActionViewから[rails-dom-testing gem](https://github.com/rails/rails-dom-testing)に移行した`SelectorAssertions`モジュールの`assert_select`メソッドです。


### マスク済み真正性トークン

SSL攻撃を緩和するために、`form_authenticity_token`がマスクされるようになりました。これにより、このトークンはリクエストごとに変更されます。トークンの検証はマスク解除 (unmasking)とそれに続く復号化 (decrypting) によって行われます。この変更が行われたことにより、railsアプリケーション以外のフォームから送信される、静的なセッションCSRFトークンに依存するリクエストを検証する際には、このマスク済み真正性トークンのことを常に考慮する必要がありますのでご注意ください。

### Action Mailer

従来は、メイラークラスでメイラーメソッドを呼び出すと、該当するインスタンスメソッドが直接実行されました。Active Jobと`#deliver_later`メソッドの導入に伴い、この動作が変更されました。Rails 4.2では、これらのインスタンスメソッド呼び出しは`deliver_now`または`deliver_later`が呼び出されるまで実行延期されます。以下に例を示します。

```ruby
class Notifier < ActionMailer::Base
  def notify(user, ...)
    puts "Called"
    mail(to: user.email, ...)
  end
end

mail = Notifier.notify(user, ...) # Notifier#notifyはこの時点では呼び出されない
mail = mail.deliver_now           # "Called"を出力する
```

この変更によって実行結果が大きく異なるアプリケーションはそれほどないと思われます。ただし、メイラー以外のメソッドを同期的に実行したい場合、かつ従来の同期的プロキシ動作に依存している場合は、これらのメソッドをメイラークラスにクラスメソッドとして直接定義する必要があります。

```ruby
class Notifier < ActionMailer::Base
  def self.broadcast_notifications(users, ...)
    users.each { |user| Notifier.notify(user, ...) }
  end
end
```

Rails 4.0からRails 4.1へのアップグレード
-------------------------------------

### リモート `<script>` タグにCSRF保護を実施

これを行わないと、「なぜかテストがとおらない...orz」ということになりかねません。

JavaScriptレスポンスを伴うGETリクエストもクロスサイトリクエストフォージェリ (CSRF) 保護の対象となりました。この保護によって、第三者のサイトが重要なデータを奪取する目的で自分のサイトのJavaScript URLを参照して実行しようとすることを防止します。

つまり、以下を使用する機能テストと結合テストは

```ruby
get :index, format: :js
```

CSRF保護をトリガーするようになります。以下のように書き換え、

```ruby
xhr :get, :index, format: :js
``` 

`XmlHttpRequest`を明示的にテストしてください。

本当にJavaScriptをリモートの`<script>`タグから読み込むのであれば、そのアクションではCSRF保護をスキップしてください。

### Spring

アプリケーションのプリローダーとしてSpringを使用する場合は、以下を行う必要があります。

1. `gem 'spring', group: :development` を `Gemfile`に追加する
2. `bundle install`を実行してSpringをインストールする
3. `bundle exec spring binstub --all`を実行してbinstubをSpring化する

NOTE: ユーザーが定義したRakeタスクはデフォルトでdevelopment環境で動作するようになります。これらのRakeタスクを他の環境でも実行したい場合は[Spring README](https://github.com/rails/spring#rake)を参考にしてください。

### `config/secrets.yml`

新しい`secrets.yml`に秘密鍵を保存したい場合は以下の手順を実行します。

1. `secrets.yml`ファイルを`config`フォルダ内に作成し、以下の内容を追加します。

    ```yaml
    development:
      secret_key_base:

    test:
      secret_key_base:

    production:
      secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
    ```

2. `secret_token.rb`イニシャライザに記載されている既存の `secret_key_base`の秘密キーを取り出してSECRET_KEY_BASE環境変数に設定し、Railsアプリケーションをproductionモードで実行するすべてのユーザーが秘密キーの恩恵を受けられるようにします。あるいは、既存の`secret_key_base`を`secret_token.rb`イニシャライザから`secrets.yml`のproductionセクションにコピーし、'<%= ENV["SECRET_KEY_BASE"] %>'を置き換えることもできます。

3. `secret_token.rb`イニシャライザを削除します

4. `rake secret`を実行し、`development`セクション`test`セクションに新しい鍵を生成します。

5. サーバーを再起動します。

### テストヘルパーの変更

テストヘルパーに`ActiveRecord::Migration.check_pending!`の呼び出しがある場合、これを削除することができます。このチェックは`require 'rails/test_help'`の際に自動的に行われるようになりました。この呼び出しを削除しなくても悪影響が生じることはありません。

### Cookiesシリアライザ

Rails 4.1より前に作成されたアプリケーションでは、`Marshal`を使用してcookie値を署名済みまたは暗号化したcookies jarにシリアライズしていました。アプリケーションで新しい`JSON`ベースのフォーマットを使用したい場合、以下のような内容を持つイニシャライザファイルを追加できます。

```ruby
Rails.application.config.action_dispatch.cookies_serializer = :hybrid
```

これにより、`Marshal`でシリアライズされた既存のcookiesを、新しい`JSON`ベースのフォーマットに透過的に移行できます。

`:json`または`:hybrid`シリアライザを使用する場合、一部のRubyオブジェクトがJSONとしてシリアライズされない可能性があることにご注意ください。たとえば、`Date`オブジェクトや`Time`オブジェクトはstringsとしてシリアライズされ、`Hash`のキーはstringに変換されます。

```ruby
class CookiesController < ApplicationController
  def set_cookie
    cookies.encrypted[:expiration_date] = Date.tomorrow # => Thu, 20 Mar 2014
    redirect_to action: 'read_cookie'
  end

  def read_cookie
    cookies.encrypted[:expiration_date] # => "2014-03-20"
  end
end
```

cookieには文字列や数字などの単純なデータだけを保存することをお勧めします。
cookieに複雑なオブジェクトを保存しなければならない場合は、後続のリクエストでcookiesから値を読み出す場合の変換については自分で面倒を見る必要があります。

cookieセッションストアを使用する場合、`session`や`flash`ハッシュについてもこのことは該当します。

### Flash構造の変更

Flashメッセージのキーが[文字列に正規化](https://github.com/rails/rails/commit/a668beffd64106a1e1fedb71cc25eaaa11baf0c1) されました。シンボルまたは文字列のどちらでもアクセスできます。Flashのキーを取り出すと常に文字列になります。

```ruby
flash["string"] = "a string"
flash[:symbol] = "a symbol"

# Rails < 4.1
flash.keys # => ["string", :symbol]

# Rails >= 4.1
flash.keys # => ["string", "symbol"]
```

Flashメッセージのキーは文字列と比較してください。

### JSONの扱いの変更点

Rails 4.1ではJSONの扱いが大きく変更された点が4つあります。

#### MultiJSONの廃止

MultiJSONはその役目を終えて [end-of-life](https://github.com/rails/rails/pull/10576) Railsから削除されました。

アプリケーションがMultiJSONに直接依存している場合、以下のような対応方法があります。

1. 'multi_json'をGemfileに追加する。ただしこのGemは将来使えなくなるかもしれません。

2. `obj.to_json`と`JSON.parse(str)`を使用してMultiJSONから乗り換える。

WARNING: `MultiJson.dump` と `MultiJson.load`をそれぞれ`JSON.dump`と`JSON.load`に単純に置き換えては「いけません」。これらのJSON gem are meant for serializing and deserializing arbitrary Ruby objects and are generally [unsafe]APIは任意のRubyオブジェクトをシリアライズおよびデシリアライズするためのものであり、一般に[安全ではありません](http://www.ruby-doc.org/stdlib-2.0.0/libdoc/json/rdoc/JSON.html#method-i-load)。

#### JSON gemの互換性

これまでのRailsでは、JSON gemとの互換性に何らかの問題が生じていました。Railsアプリケーション内の`JSON.generate`と`JSON.dump`ではときたまエラーが生じることがありました。

Rails 4.1では、Rails自身のエンコーダをJSON gemから切り離すことでこれらの問題が修正されました。JSON gem APIは今後正常に動作しますが、その代わりJSON gem APIからRails特有の機能にアクセスすることはできなくなります。以下に例を示します。

```ruby
class FooBar
  def as_json(options = nil)
    { foo: 'bar' }
  end
end

>> FooBar.new.to_json # => "{\"foo\":\"bar\"}"
>> JSON.generate(FooBar.new, quirks_mode: true) # => "\"#<FooBar:0x007fa80a481610>\""
```

#### 新しいJSONエンコーダ

Rails 4.1のJSONエンコーダは、JSON gemを使用するように書き直されました。この変更によるアプリケーションへの影響はほとんどありません。ただし、エンコーダが書き直された際に以下の機能がエンコーダから削除されました。

1. データ構造の循環検出
2. `encode_json`フックのサポート
3. `BigDecimal`オブジェクトを文字ではなく数字としてエンコードするオプション

アプリケーションがこれらの機能に依存している場合は、[`activesupport-json_encoder`](https://github.com/rails/activesupport-json_encoder) gemをGemfileに追加することで以前の状態に戻すことができます。

#### TimeオブジェクトのJSON形式表現

日時に関連するコンポーネント(`Time`、`DateTime`、`ActiveSupport::TimeWithZone`)を持つオブジェクトに対して`#as_json`を実行すると、デフォルトでミリ秒単位の精度で値が返されるようになりました。ミリ秒より精度の低い従来方式にしておきたい場合は、イニシャライザに以下を設定してください。

```
ActiveSupport::JSON::Encoding.time_precision = 0
```

### インラインコールバックブロックで`return`の使用法

以前のRailsでは、インラインコールバックブロックで以下のように`return`を使用することが許容されていました。

```ruby
class ReadOnlyModel < ActiveRecord::Base
  before_save { return false } # 良くない
end
```

この動作は決して意図されたものではありません。`ActiveSupport::Callbacks`が書き直され、上のような動作はRails 4.1では許容されなくなりました。インラインコールバックブロックで`return`文を書くと、コールバック実行時に`LocalJumpError`が発生するようになりました。

インラインコールバックブロックで`return`を使用している場合、以下のようにリファクタリングすることで、返された値として評価されるようになります。

```ruby
class ReadOnlyModel < ActiveRecord::Base
  before_save { false } # 良い
end
```

`return`を使用したいのであれば、明示的にメソッドを定義することが推奨されます。

```ruby
class ReadOnlyModel < ActiveRecord::Base
  before_save :before_save_callback # 良い

  private
    def before_save_callback
      return false
    end
end
```

この変更は、Railsでコールバックを使用している多くの箇所に適用されます。これにはActive RecordとActive ModelのコールバックやAction Controllerのフィルタ(`before_action` など)も含まれます。

詳細については[このpull request](https://github.com/rails/rails/pull/13271)を参照してください。

### Active Recordフィクスチャで定義されたメソッド

Rails 4.1では、各フィクスチャのERBは独立したコンテキストで評価されます。このため、あるフィクスチャで定義されたヘルパーメソッドは他のフィクスチャでは利用できません。

ヘルパーメソッドを複数のフィクスチャで使用するには、4.1で新しく導入された`ActiveRecord::FixtureSet.context_class` (`test_helper.rb`) に含まれるモジュールで定義する必要があります。

```ruby
module FixtureFileHelpers
  def file_sha(path)
    Digest::SHA2.hexdigest(File.read(Rails.root.join('test/fixtures', path)))
  end
end
ActiveRecord::FixtureSet.context_class.send :include, FixtureFileHelpers
```

### I18nオプションでavailable_localesリストの使用が強制される

Rails 4.1からI18nオプション`enforce_available_locales`がデフォルトで`true`になりました。この設定にすると、I18nに渡されるすべてのロケールは、available_localesリストで宣言されていなければ使用できません。

この機能をオフにしてI18nですべての種類のロケールオプションを使用できるようにするには、以下のように変更します。

```ruby
config.i18n.enforce_available_locales = false
``` 

available_localesの強制はセキュリティのために行われていることにご注意ください。つまり、アプリケーションが把握していないロケールを持つユーザー入力が、ロケール情報として使用されることのないようにするためのものです。従って、やむを得ない理由がない限りこのオプションはfalseにしないでください。

### リレーションに対するミューテーターメソッド呼び出し

`Relation`に`#map!`や`#delete_if`などのミューテーターメソッド (mutator method) が含まれなくなりました。これらのメソッドを使用したい場合は`#to_a`を呼び出して`Array`に変更してからにしてください。

この変更は、`Relation`に対して直接ミューテーターメソッドを呼び出すことによる奇妙なバグや混乱を防止するために行われました。

```ruby
# 以前のミューテーター呼び出し方法
Author.where(name: 'Hank Moody').compact!

# 今後のミューテーター呼び出し方法
authors = Author.where(name: 'Hank Moody').to_a
authors.compact!
``` 

### デフォルトスコープの変更

デフォルトのスコープは、条件を連鎖した場合にオーバーライドされなくなりました。

以前のバージョンでは、モデルで`default_scope`を定義すると、同じフィールドで連鎖した条件によってオーバーライドされました。現在は、他のスコープと同様、マージされるようになりました。

変更前:

```ruby
class User < ActiveRecord::Base
  default_scope { where state: 'pending' }
  scope :active, -> { where state: 'active' }
  scope :inactive, -> { where state: 'inactive' }
end

User.all
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending'

User.active
# SELECT "users".* FROM "users" WHERE "users"."state" = 'active'

User.where(state: 'inactive')
# SELECT "users".* FROM "users" WHERE "users"."state" = 'inactive'
```

変更後:

```ruby
class User < ActiveRecord::Base
  default_scope { where state: 'pending' }
  scope :active, -> { where state: 'active' }
  scope :inactive, -> { where state: 'inactive' }
end

User.all
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending'

User.active
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending' AND "users"."state" = 'active'

User.where(state: 'inactive')
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending' AND "users"."state" = 'inactive'
```

以前と同じ動作に戻したい場合は、`unscoped`、`unscope`、`rewhere`、または`except`を使用して`default_scope`の条件を明示的に除外する必要があります。

```ruby
class User < ActiveRecord::Base
  default_scope { where state: 'pending' }
  scope :active, -> { unscope(where: :state).where(state: 'active') }
  scope :inactive, -> { rewhere state: 'inactive' }
end

User.all
# SELECT "users".* FROM "users" WHERE "users"."state" = 'pending'

User.active
# SELECT "users".* FROM "users" WHERE "users"."state" = 'active'

User.inactive
# SELECT "users".* FROM "users" WHERE "users"."state" = 'inactive'
```

### 文字列からのコンテンツ描出

Rails 4.1の`render`に`:plain`、`:html`、`:body`オプションが導入されました。以下のようにコンテンツタイプを指定できるため、文字列ベースのコンテンツ表示にはこれらのオプションの使用が推奨されます。

* `render :plain`を実行するとcontent typeは`text/plain`に設定される
* `render :html`を実行するとcontent typeは`text/html`に設定される
* `render :body`を実行した場合、content typeヘッダーは「設定されない」

セキュリティ上の観点から、レスポンスのbodyにマークアップを含めない場合には`render :plain`を使用すべきです。これによって多くのブラウザが安全でないコンテンツをエスケープできるからです。

今後のバージョンでは、`render :text`は非推奨にされる予定です。今のうちに、正しい`:plain`、`:html`、`:body`オプションに切り替えてください。
`render :text`を使用すると`text/html`で送信されるため、セキュリティ上のリスクが生じる可能性があります。

### PostgreSQLのデータ型'json'と'hstore'について

Rails 4.1では、PostgreSQLの`json`カラムと`hstore`カラムを、文字列をキーとするRubyの`Hash`に対応付けるようになりました。
なお、以前のバージョンでは`HashWithIndifferentAccess`が使用されていました。この変更は、Rails 4.1以降ではシンボルを使用してこれらのデータ型にアクセスできなくなるということを意味します。`store_accessors`メソッドは`json`カラムや`hstore`カラムに依存しているので、同様にシンボルでのアクセスが行えなくなります。今後は常に文字列をキーにするようにしてください。

### `ActiveSupport::Callbacks`では明示的にブロックを使用すること

Rails 4.1からは`ActiveSupport::Callbacks.set_callback`の呼び出しの際に明示的にブロックを渡すことが期待されます。これは、`ActiveSupport::Callbacks`がRails 4.1リリースにあたって大幅に書き換えられたことによるものです。

```ruby
# Rails 4.0の場合
set_callback :save, :around, ->(r, &block) { stuff; result = block.call; stuff }

# Rails 4.1の場合
set_callback :save, :around, ->(r, block) { stuff; result = block.call; stuff }
```

Rails 3.2からRails 4.0へのアップグレード
-------------------------------------

Railsアプリケーションのバージョンが3.2より前の場合、まず3.2へのアップグレードを完了してからRails 4.0へのアップグレードを開始してください。

以下の変更は、アプリケーションをRails 4.0にアップグレードするためのものです。

### HTTP PATCH

Rails 4では、`config/routes.rb`でRESTfulなリソースが宣言されたときに、更新用の主要なHTTP verbとして`PATCH`が使用されるようになりました。`update`アクションは従来通り使用でき、`PUT`リクエストは今後も`update`アクションにルーティングされます。
標準的なRESTfulのみを使用しているのであれば、これに関する変更は不要です。

```ruby
resources :users
```

```erb
<%= form_for @user do |f| %>
```

```ruby
class UsersController < ApplicationController
  def update
    # 変更不要:PATCHが望ましいがPUTも使用できる
  end
end
```

ただし、`form_for`を使用してリソースを更新しており、`PUT` HTTPメソッドを使用するカスタムルーティングと連動しているのであれば、変更が必要です。

```ruby
resources :users, do
  put :update_name, on: :member
end
```

```erb
<%= form_for [ :update_name, @user ] do |f| %>
```

```ruby
class UsersController < ApplicationController
  def update_name
    # 変更が必要: form_forは、存在しないPATCHルートを探そうとする
  end
end
```

このアクションがパブリックなAPIで使用されておらず、HTTPメソッドを自由に変更できるのであれば、ルーティングを更新して`patch`を`put`の代りに使用できます。

Rails 4で`PUT`リクエストを`/users/:id`に送信すると、従来と同様`update`にルーティングされます。このため、実際のPUTリクエストを受け取るAPIは今後も利用できます。
この場合、`PATCH`リクエストも`/users/:id`経由で`update`アクションにルーティングされます。

```ruby
resources :users do
  patch :update_name, on: :member
end
```

このアクションがパブリックなAPIで使用されており、HTTPメソッドを自由に変更できないのであれば、フォームを更新して`PUT`を代りに使用できます。

```erb
<%= form_for [ :update_name, @user ], method: :put do |f| %>
```

PATCHおよびこの変更が行われた理由についてはRailsブログの [この記事](http://weblog.rubyonrails.org/2012/2/26/edge-rails-patch-is-the-new-primary-http-method-for-updates/) を参照してください。

#### メディアタイプに関するメモ

`PATCH` verbに関する追加情報 [`PATCH`では異なるメディアタイプを使用する必要がある](http://www.rfc-editor.org/errata_search.php?rfc=5789)。[JSON Patch](http://tools.ietf.org/html/rfc6902) などが該当します。RailsはJSON Patchをネイティブではサポートしませんが、サポートは簡単に追加できます。

```
# コントローラに以下を書く
def update
  respond_to do |format|
    format.json do
      # 部分的な変更を行なう
      @article.update params[:article]
    end

    format.json_patch do
      # 何か気の利いた変更を行なう
    end
  end
end

# config/initializers/json_patch.rb に以下を書く
Mime::Type.register 'application/json-patch+json', :json_patch
```

JSON Patchは最近RFC化されたばかりなのでRubyライブラリはそれほどありません。Aaron Pattersonの [hana](https://github.com/tenderlove/hana) gemが代表的ですが、最新の仕様変更をすべてサポートしているわけではありません。

### Gemfile

Rails 4.0では`assets`グループがGemfileから削除されました。アップグレード時にはこの記述をGemfileから削除する必要があります。アプリケーションの`config/application.rb`ファイルも以下のように更新する必要があります。

```ruby
# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(:default, Rails.env)
``` 

### vendor/plugins

Rails 4.0 では `vendor/plugins` 読み込みのサポートは完全に終了しました。使用するプラグインはすべてgemに展開してGemfileに追加しなければなりません。理由があってプラグインをgemにしないのであれば、プラグインを`lib/my_plugin/*`に移動し、適切な初期化の記述を`config/initializers/my_plugin.rb`に書いてください。

### Active Record

* [関連付けに関する若干の不整合](https://github.com/rails/rails/commit/302c912bf6bcd0fa200d964ec2dc4a44abe328a6) のため、Rails 4.0ではActive Recordからidentity mapが削除されました。この機能をアプリケーションで手動で有効にしたい場合は、今や無効になった`config.active_record.identity_map`を削除する必要があるでしょう。

* コレクション関連付けの`delete`メソッドは、`Fixnum`や`String`引数をレコードの他にレコードIDとしても受け付けるようになりました。これにより`destroy`メソッドの動作にかなり近くなりました。以前はこのような引数を使用すると`ActiveRecord::AssociationTypeMismatch`例外が発生しました。Rails 4.0からは、`delete`メソッドを使用すると、与えられたIDにマッチするレコードを自動的に探すようになりました。

* Rails 4.0では、カラムやテーブルの名前を変更すると、関連するインデックスも自動的にリネームされるようになりました。インデックス名を変更するためだけのマイグレーションは今後不要になりました。

* Rails 4.0の`serialized_attributes`メソッドと`attr_readonly`メソッドは、クラスメソッドとしてのみ使用するように変更されました。これらのメソッドをインスタンスメソッドとして使用することは非推奨となったため、行わないでください。たとえば`self.serialized_attributes`は`self.class.serialized_attributes`のようにクラスメソッドとして使用してください。

* デフォルトのコーダーを使用する場合、シリアル化属性に`nil`を渡すと、YAML全体にわたって (`nil`値を渡す代わりに) `NULL`としてデータベースに保存されます (`"--- \n...\n"`)。

* Rails 4.0ではStrong Parametersの導入に伴い、`attr_accessible`と`attr_protected`が廃止されました。これらを引き続き使用したい場合は、[Protected Attributes gem](https://github.com/rails/protected_attributes) を導入することでスムーズにアップグレードすることができます。

* Protected Attributesを使用していないのであれば、`whitelist_attributes`や`mass_assignment_sanitizer`オプションなど、このgemに関連するすべてのオプションを削除できます。

* Rails 4.0のスコープでは、Procやlambdaなどの呼び出し可能なオブジェクトの使用が必須となりました。

```ruby
  scope :active, where(active: true)

  # 上のコードは以下のように変更する必要がある
  scope :active, -> { where active: true }
```

* `ActiveRecord::FixtureSet`の導入に伴い、Rails 4.0では`ActiveRecord::Fixtures`が非推奨となりました。

* `ActiveSupport::TestCase`の導入に伴い、Rails 4.0では`ActiveRecord::TestCase`が非推奨となりました。

* Rails 4.0では、ハッシュを使用する旧来のfinder APIが非推奨となりました。これまでfinderオプションを受け付けていたメソッドは、これらのオプションを今後受け付けなくなりますたとえば、`Book.find(:all, conditions: { name: '1984' })`は非推奨です。今後は`Book.where(name: '1984')`をご使用ください。

* 動的なメソッドは、`find_by_...`と`find_by_...!`を除いて非推奨となりました。
  以下のように変更してください。

      * `find_all_by_...`           に代えて `where(...)` を使用
      * `find_last_by_...`          に代えて `where(...).last` を使用
      * `scoped_by_...`             に代えて `where(...)` を使用
      * `find_or_initialize_by_...` に代えて`find_or_initialize_by(...)`を使用
      * `find_or_create_by_...`   に代えて`find_or_create_by(...)`を使用

* 旧来のfinderが配列を返していたのに対し、`where(...)`はリレーションを返します。`Array`が必要な場合は, `where(...).to_a`を使用してください。

* これらの同等なメソッドが実行するSQLは、従来の実装と同じではありません。

* 旧来のfinderを再度有効にしたい場合は、[activerecord-deprecated_finders gem](https://github.com/rails/activerecord-deprecated_finders) を使用できます。

### Active Resource

Rails 4.0ではActive Resourceがgem化されました。この機能が必要な場合は[Active Resource gem](https://github.com/rails/activeresource) をGemfileに追加できます。

### Active Model

* Rails 4.0では`ActiveModel::Validations::ConfirmationValidator`にエラーがアタッチされる方法が変更されました。確認バリデーションが失敗したときに、`attribute`ではなく`:#{attribute}_confirmation`にアタッチされるようになりました。

* Rails 4.0の`ActiveModel::Serializers::JSON.include_root_in_json`のデフォルト値が`false`に変更されました。これにより、Active Model SerializersとActive Recordオブジェクトのデフォルトの動作が同じになりました。これにより、`config/initializers/wrap_parameters.rb`ファイルの以下のオプションをコメントアウトしたり削除したりできるようになりました。

```ruby
# Disable root element in JSON by default.
# ActiveSupport.on_load(:active_record) do
#   self.include_root_in_json = false
# end
```

### Action Pack

* Rails 4.0から`ActiveSupport::KeyGenerator`が導入され、署名付きcookiesの生成と照合などに使用されるようになりました。Rails 3.xで生成された既存の署名付きcookiesは、既存の`secret_token`はそのままにして`secret_key_base`を新しく追加することで透過的にアップグレードされます。

```ruby
  # config/initializers/secret_token.rb
  Myapp::Application.config.secret_token = 'existing secret token'
  Myapp::Application.config.secret_key_base = 'new secret key base'
```

注意：`secret_key_base`を設定するのは、Rails 4.xへのユーザーベースの移行が100%完了し、Rails 3.xにロールバックする必要が完全になくなってからにしてください。これは、Rails 4.xの新しい`secret_key_base`を使用して署名されたcookiesにはRails 3.xのcookiesとの後方互換性がないためです。他のアップグレードが完全に完了するまでは、既存の`secret_token`をそのままにして`secret_key_base`を設定せず、非推奨警告を無視するという選択肢もあります。

外部アプリケーションやJavaScriptからRailsアプリケーションの署名付きセッションcookies (または一般の署名付きcookies) を読み出せる必要がある場合は、これらの問題を切り離すまでは`secret_key_base`を設定しないでください。

* Rails 4.0では、`secret_key_base`が設定されているとcookieベースのセッションの内容が暗号化されます。Rails 3.xではcookieベースのセッションへの署名は行われますが暗号化は行われません。署名付きcookiesは、そのRailsアプリケーションで生成されたことが確認でき、不正が防止されるという意味では安全です。しかしセッションの内容はエンドユーザーから見えてしまいます。内容を暗号化することで懸念を取り除くことができ、パフォーマンスの低下もそれほどありません。

セッションcookiesを暗号化する方法の詳細については[Pull Request #9978](https://github.com/rails/rails/pull/9978) を参照してください。

* Rails 4.0 では`ActionController::Base.asset_path`オプションが廃止されました。代りにアセットパイプライン機能をご利用ください。

* Rails 4.0では`ActionController::Base.page_cache_extension`オプションが非推奨になりました。代りに`ActionController::Base.default_static_extension`をご利用ください。

* Rails 4.0のAction PackからActionとPageのキャッシュ機能が取り除かれました。コントローラで`caches_action`を使用したい場合は`actionpack-action_caching` gemを、`caches_pages`を使用したい場合は`actionpack-page_caching` gemをそれぞれGemfileに追加する必要があります。

* Rails 4.0からXMLパラメータパーサーが取り除かれました。この機能が必要な場合は`actionpack-xml_parser` gemを追加する必要があります。

* Rails 4.0のデフォルトのmemcachedクライアントが`memcache-client`から`dalli`に変更されました。アップグレードするには、単に`gem 'dalli'`を`Gemfile`に追加します。

* Rails 4.0ではコントローラでの`dom_id`および`dom_class`メソッドの使用が非推奨になりました (ビューでの使用は問題ありません)。この機能が必要なコントローラでは`ActionView::RecordIdentifier`モジュールをインクルードする必要があります。

* Rails 4.0では`link_to`ヘルパーでの`:confirm`オプションが非推奨になりました。代りにデータ属性を使用してください (例： `data: { confirm: 'Are you sure?' }`)。
`link_to_if`や`link_to_unless`などでも同様の対応が必要です。

* Rails 4.0では`assert_generates`、`assert_recognizes`、`assert_routing`の動作が変更されました。これらのアサーションからは`ActionController::RoutingError`の代りに`Assertion`が発生するようになりました。

* Rails 4.0では、名前付きルートの定義が重複している場合に`ArgumentError`が発生するようになりました。このエラーは、明示的に定義された名前付きルートや`resources`メソッドによってトリガされます。名前付きルート`example_path`が衝突している例を2つ示します。

```ruby
  get 'one' => 'test#example', as: :example
  get 'two' => 'test#example', as: :example
```

```ruby
  resources :examples
  get 'clashing/:id' => 'test#example', as: :example
```

最初の例では、複数のルーティングで同じ名前を使用しないようにすれば回避できます。次の例では、`only`または`except`オプションを`resources`メソッドで使用することで、作成されるルーティングを制限することができます。詳細は [Routing Guide](routing.html#ルーティングの作成を制限する) を参照。

* Rails 4.0ではunicode文字のルーティングの描出方法が変更されました。unicode文字を使用するルーティングを直接描出できるようになりました。既にこのようなルーティングを使用している場合は、以下の変更が必要です。

```ruby
get Rack::Utils.escape('こんにちは'), controller: 'welcome', action: 'index'
```

上のコードは以下のように変更する必要があります。

```ruby
get 'こんにちは', controller: 'welcome', action: 'index'
```

* Rails 4.0でルーティングに`match`を使用する場合は、リクエストメソッドの指定が必須となりました。以下に例を示します。

```ruby
  # Rails 3.x
  match '/' => 'root#index'

  # 上のコードは以下のように変更する必要があります。
  match '/' => 'root#index', via: :get

  # または
  get '/' => 'root#index'
```

* Rails 4.0から`ActionDispatch::BestStandardsSupport`ミドルウェアが削除されました。`<!DOCTYPE html>`は既に http://msdn.microsoft.com/en-us/library/jj676915(v=vs.85).aspx の標準モードをトリガするようになり、ChromeFrameヘッダは`config.action_dispatch.default_headers`に移動されました。

アプリケーションコード内にあるこのミドルウェアへの参照はすべて削除する必要がありますので注意が必要です。例：

```ruby
# 例外発生
config.middleware.insert_before(Rack::Lock, ActionDispatch::BestStandardsSupport)
```

環境設定も確認し、`config.action_dispatch.best_standards_support`がある場合は削除します。

* Rails 4.0のアセットのプリコンパイルでは、`vendor/assets`および`lib/assets`にある非JS/CSSアセットを自動的にはコピーしなくなりました。Railsアプリケーションとエンジンの開発者は、これらのアセットを手動で`app/assets`に置き、`config.assets.precompile`を設定してください。

* Rails 4.0では、リクエストされたフォーマットがアクションで扱えなかった場合に`ActionController::UnknownFormat`が発生するようになりました。デフォルトでは、この例外は406 Not Acceptable応答として扱われますが、この動作をオーバーライドすることができます。Rails 3では常に406 Not Acceptableが返されます。オーバーライドはできません。

* Rails 4.0では、`ParamsParser`がリクエストパラメータをパースできなかった場合に一般的な`ActionDispatch::ParamsParser::ParseError`例外が発生するようになりました。`MultiJson::DecodeError`のような低レベルの例外の代りにこの例外をレスキューすることができます。

* Rails 4.0では、URLプレフィックスで指定されたアプリケーションにエンジンがマウントされている場合に`SCRIPT_NAME`が正しく入れ子になるようになりました。今後はURLプレフィックスの上書きを回避するために`default_url_options[:script_name]`を設定する必要はありません。

* Rails 4.0では`ActionDispatch::Integration`の導入に伴い`ActionController::Integration`が非推奨となりました。
* Rails 4.0では`ActionDispatch::IntegrationTest`の導入に伴い`ActionController::IntegrationTest`は非推奨となりました。
* Rails 4.0では`ActionDispatch::PerformanceTest`の導入に伴い`ActionController::PerformanceTest`が非推奨となりました。
* Rails 4.0では`ActionDispatch::Request`の導入に伴い`ActionController::AbstractRequest`が非推奨となりました。
* Rails 4.0では`ActionDispatch::Request`の導入に伴い`ActionController::Request`が非推奨となりました。
* Rails 4.0では`ActionDispatch::Response`の導入に伴い`ActionController::AbstractResponse`が非推奨となりました。
* Rails 4.0では`ActionDispatch::Response`の導入に伴い`ActionController::Response`が非推奨となりました。
* Rails 4.0では`ActionDispatch::Routing`の導入に伴い`ActionController::Routing`が非推奨となりました。

### Active Support

Rails 4.0では`ERB::Util#json_escape`のエイリアス`j`が廃止されました。このエイリアス`j`は既に`ActionView::Helpers::JavaScriptHelper#escape_javascript`で使用されているためです。

### ヘルパーの読み込み順序

Rails 4.0では複数のディレクトリからのヘルパーの読み込み順が変更されました。以前はすべてのヘルパーをいったん集めてからアルファベット順にソートしていました。Rails 4.0にアップグレードすると、ヘルパーは読み込まれたディレクトリの順序を保持し、ソートは各ディレクトリ内でのみ行われます。`helpers_path`パラメータを明示的に使用している場合を除いて、この変更はエンジンからヘルパーを読み込む方法にしか影響しません。ヘルパー読み込みの順序に依存している場合は、アップグレード後に正しいメソッドが使用できているかどうかを確認する必要があります。エンジンが読み込まれる順序を変更したい場合は、`config.railties_order=` メソッドを使用できます。

### Active Record ObserverとAction Controller Sweeper

`Active Record Observer`と`Action Controller Sweeper`は`rails-observers` gemに切り出されました。これらの機能が必要な場合は`rails-observers` gemを追加してください。

### sprockets-rails

* `assets:precompile:primary`および`assets:precompile:all`は削除されました。`assets:precompile`を代りに使用してください。
* `config.assets.compress`オプションは、たとえば以下のように`config.assets.js_compressor` に変更する必要があります。

```ruby
config.assets.js_compressor = :uglifier
```

### sass-rails

* 引数を2つ使用する`asset-url`は非推奨となりました。たとえば、`asset-url("rails.png", image)`は`asset-url("rails.png")`とする必要があります。

Rails 3.1からRails 3.2へのアップグレード
-------------------------------------

Railsアプリケーションのバージョンが3.1より前の場合、まず3.1へのアップグレードを完了してからRails 3.2へのアップグレードにとりかかってください。

Railsアプリケーションのバージョンが3.1よりも古い場合、まず3.1へのアップグレードを完了してからRails 3.2へのアップグレードを開始してください。

### Gemfile

`Gemfile`を以下のように変更します。

```ruby
gem 'rails', '3.2.21'

group :assets do
  gem 'sass-rails',   '~> 3.2.6'
  gem 'coffee-rails', '~> 3.2.2'
  gem 'uglifier',     '>= 1.0.3'
end
```

### config/environments/development.rb

development環境にいくつかの新しい設定を追加する必要があります。

```ruby
# Active Recordのモデルをマスアサインメントから保護するために例外を発生する
config.active_record.mass_assignment_sanitizer = :strict

# クエリの実行計画 (クエリプラン) を現在より多く出力する
# (SQLite、MySQL、PostgreSQLで動作)
config.active_record.auto_explain_threshold_in_seconds = 0.5
```

### config/environments/test.rb

`mass_assignment_sanitizer`設定を`config/environments/test.rb`にも追加する必要があります。

```ruby
# Active Recordのモデルをマスアサインメントから保護するために例外を発生する
config.active_record.mass_assignment_sanitizer = :strict
```

### vendor/plugins

`vendor/plugins` はRails 3.2で非推奨となり、Rails 4.0では完全に削除されました。Rails 3.2へのアップグレードでは必須ではありませんが、今のうちにプラグインをgemにエクスポートしてGemfileに追加するのがよいでしょう。理由があってプラグインをgemにしないのであれば、プラグインを`lib/my_plugin/*`に移動し、適切な初期化の記述を`config/initializers/my_plugin.rb`に書いてください。

### Active Record

`:dependent => :restrict`オプションは`belongs_to`から削除されました。関連付けられたオブジェクトがある場合にこのオブジェクトを削除したくない場合は、`:dependent => :destroy`を設定し、関連付けられたオブジェクトのdestroyコールバックとの関連付けがあるかどうかを確認してから`false`を返すようにします。

Rails 3.0からRails 3.1へのアップグレード
-------------------------------------

Railsアプリケーションのバージョンが3.0より前の場合、まず3.0へのアップグレードを完了してからRails 3.1へのアップグレードにとりかかってください。

以下の変更は、Rails 3.1.xの最新版であるRails 3.1.12にアップグレードするためのものです。

### Gemfile

`Gemfile`を以下のように変更します。

```ruby
gem 'rails', '3.1.12'
gem 'mysql2' 

# 新しいアセットパイプラインで必要
group :assets do
  gem 'sass-rails',   '~> 3.1.7'
  gem 'coffee-rails', '~> 3.1.1'
  gem 'uglifier',     '>= 1.0.3'
end

# Rails 3.1からjQueryがデフォルトのJavaScriptライブラリになる
gem 'jquery-rails'
```

### config/application.rb

アセットパイプラインを使用するために以下の変更が必要です。

```ruby
config.assets.enabled = true
config.assets.version = '1.0'
```

Railsアプリケーションでリソースのルーティングに"/assets"ルートを使用している場合、コンフリクトを避けるために以下の変更を加えます。

```ruby
# '/assets'のデフォルト
config.assets.prefix = '/asset-files'
``` 

### config/environments/development.rb

RJS の設定`config.action_view.debug_rjs = true`を削除してください。

アセットパイプラインを有効にしている場合は以下の設定を追加します。

```ruby
# 開発環境ではアセットを圧縮しない
config.assets.compress = false

# アセットで読み込んだ行を展開する
config.assets.debug = true
``` 

### config/environments/production.rb

以下の変更はほとんどがアセットパイプライン用です。詳細については [アセットパイプライン](asset_pipeline.html) ガイドを参照してください。

```ruby
# JavaScriptとCSSを圧縮する
config.assets.compress = true

# プリコンパイル済みのアセットが見当たらない場合にアセットパイプラインにフォールバックしない
config.assets.compile = false

# アセットURLのダイジェストを生成する
config.assets.digest = true

# Rails.root.join("public/assets")へのデフォルト
# config.assets.manifest = 該当するパス

# 追加のアセット (application.js、application.cssおよびすべての非JS/CSSが追加済み) をプリコンパイルする
# config.assets.precompile += %w( search.js )

# アプリケーションへのすべてのアクセスを強制的にSSLにし、Strict-Transport-Securityとセキュアクッキーを使用する
# config.force_ssl = true
``` 

### config/environments/test.rb

テスト環境に以下を追加することでテストのパフォーマンスが向上します。

```ruby
# Cache-Controlを使用するテストで静的アセットサーバーを構成し、パフォーマンスを向上させる
config.serve_static_assets = true
config.static_cache_control = 'public, max-age=3600'
```

### config/initializers/wrap_parameters.rb

ネストしたハッシュにパラメータを含めたい場合は、このファイルに以下のコンテンツを含めて追加します。新しいアプリケーションではこれがデフォルトになります。

```ruby
# このファイルを変更後サーバーを必ず再起動してください。
# このファイルにはActionController::ParamsWrapper用の設定が含まれており
# デフォルトでオンになっています。

# JSON用にパラメータをラップします。:formatに空配列を設定することで無効にできます。
ActiveSupport.on_load(:action_controller) do
  wrap_parameters format: [:json]
end

# JSONのルート要素をデフォルトで無効にする
ActiveSupport.on_load(:active_record) do
  self.include_root_in_json = false
end
```

### config/initializers/session_store.rb

何らかの新しいセッションキーを設定するか、すべてのセッションを削除するかのどちらかにする必要があります。

```ruby
# config/initializers/session_store.rbに以下を設定する
AppName::Application.config.session_store :cookie_store, key: 'SOMETHINGNEW'
```

または

```bash
$ bin/rake db:sessions:clear
```

### ビューのアセットヘルパー参照から:cacheオプションと:concatオプションを削除する

* Asset Pipelineの:cacheオプションと:concatは廃止されました。ビューからこれらのオプションを削除してください。