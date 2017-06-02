# 序幕

> Role models are important. <br/>
> -- 機器戰警 Alex J. Murphy

這份指南目的於示範一整套 Rails 3 開發的風格慣例及最佳實踐。這是一份與由現存社群所驅動的 [Ruby 程式碼風格指南](https://github.com/bbatsov/ruby-style-guide)互補的指南。

而本指南中[測試 Rails 應用](#testing)小節擺在[開發 Rails 應用](#developing)之後，因為我相信[行為驅動開發](http://en.wikipedia.org/wiki/Behavior_Driven_Development)
(BDD) 是最佳的軟體開發之道。銘記在心吧。

Rails 是一個堅持己見的框架，而這也是一份堅持己見的指南。在我的心裡，我堅信 [RSpec](https://www.relishapp.com/rspec) 優於 Test::Unit，[Sass](http://sass-lang.com/) 優於 CSS 以及
[Haml](http://haml-lang.com/)，([Slim](http://slim-lang.com/)) 優於 Erb。所以不要期望在這裡找到 Test::Unit, CSS 及 Erb 的忠告。

某些忠告僅適用於 Rails 3.1+ 以上版本。

你可以使用 [Transmuter](https://github.com/TechnoGate/transmuter) 來產生本指南的一份 PDF 或 HTML 複本。

本指南被翻譯成下列語言：

* [简体中文](https://github.com/JuanitoFatas/rails-style-guide/blob/master/README-zhCN.md)
* [英文原版](https://github.com/JuanitoFatas/rails-style-guide/blob/master/README.md)

# 目錄

* [開發 Rails 應用程式](#-rails-)
    * [設定檔 (Config)](#-2)
    * [路由 (Routes)](#-routes)
    * [控制器 (Controller)](#-controller)
    * [資料模型 (Model)](#-model)
    * [遷移 (Migrations)](#-migrations)
    * [視圖 (Views)](#-views)
    * [國際化 (I18n)](#-3)
    * [資產 (Assets)](#assets)
    * [郵件 (Mailers)](#mailers)
    * [打包 (Bundler)](#bundler)
    * [貴重的 Gems](#-priceless-gems)
    * [缺陷的 Gems](#-gems)
    * [管理處理程序 (Process)](#-process)
* [測試 Rails 應用程式](#-rails--1)
    * [Cucumber](#cucumber)
    * [RSpec](#rspec)


# 開發 Rails 應用程式

## 設定檔 (config)

* 把自訂的初始化程式碼放在 `config/initializers`。在 initializers 內的程式碼會在應用程式啟動時執行。
* 每一個 gem 相關的初始化程式碼應當使用同樣的名稱，放在不同的文件裡，如： `carrierwave.rb`, `active_admin.rb`, 等等。
* 為開發、測試及生產環境分別調整設定（在 `config/environments/` 下對應的文件）
  * 標記額外的資產 (assets) 給預編譯（如果有的話）：

        ```Ruby
        # config/environments/production.rb
        # Precompile additional assets (application.js, application.css, and all non-JS/CSS are already added)
        config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
        ```

* 將所有環境都通用的設定檔放在 `config/application.rb` 檔案。
* 另外開一個與生產環境(production enviroment)幾乎相同的 `staging` 環境。

## 路由 (Routes)

* 當你需要加入一個或多個動作 (action) 至一個 RESTful 資源時（你真的需要嗎？），使用 `member` and `collection` 路由。

    ```Ruby
    # 劣
    get 'subscriptions/:id/unsubscribe'
    resources :subscriptions

    # 優
    resources :subscriptions do
      get 'unsubscribe', on: :member
    end

    # 劣
    get 'photos/search'
    resources :photos

    # 優
    resources :photos do
      get 'search', on: :collection
    end
    ```

* 若需要定義多個 `member/collection` 路由，請改用區塊語法(block syntax)。

    ```Ruby
    resources :subscriptions do
      member do
        get 'unsubscribe'
        # 更多路由
      end
    end

    resources :photos do
      collection do
        get 'search'
        # 更多路由
      end
    end
    ```

* 使用巢狀路由(nested routes)來更佳地表達各 ActiveRecord 資料模型之間的關係。

    ```Ruby
    class Post < ActiveRecord::Base
      has_many :comments
    end

    class Comments < ActiveRecord::Base
      belongs_to :post
    end

    # routes.rb
    resources :posts do
      resources :comments
    end
    ```

* 使用命名空間路由來分類相關的行為。

    ```Ruby
    namespace :admin do
      # Directs /admin/products/* to Admin::ProductsController
      # (app/controllers/admin/products_controller.rb)
      resources :products
    end
    ```

* 不要使用地圖砲路由。這種路由會讓每個控制器的動作透過 GET 請求存取。

    ```Ruby
    # 極劣
    match ':controller(/:action(/:id(.:format)))'
    ```

## 控制器 (Controller)

* 讓你的控制器保持苗條──它們應該只替視圖層取出資料且不包含任何業務邏輯（所有業務邏輯應當放在資料模型裡）。
* 每個控制器裡的動作 (action) 應當只呼叫一個除了初始的 find 或 new 以外的方法（理想狀態）。
* 控制器與視圖之間共享不超過兩個實體變數 (instance variable)。

## 資料模型 (Model)

* 請任意引入不是 ActiveRecord 的資料模型。
* 替資料模型命名有意義（但簡短）且不帶縮寫的名字。
* 如果你需要普通的資料模型有著 ActiveRecord 的行為，比方說驗證，可使用 [ActiveAttr](https://github.com/cgriego/active_attr) gem。

    ```Ruby
    class Message
      include ActiveAttr::Model

      attribute :name
      attribute :email
      attribute :content
      attribute :priority

      attr_accessible :name, :email, :content

      validates_presence_of :name
      validates_format_of :email, :with => /\A[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}\z/i
      validates_length_of :content, :maximum => 500
    end
    ```

    更完整的範例，參考 [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr)。

### ActiveRecord

* 避免改動預設的 ActiveRecord（表的名字、主鍵，等等），除非你有一個非常好的理由（像是不受你控制的資料庫）。
* 把巨集風格的方法放在類別定義的前面（`has_many`, `validates`, 等等）。

    ```Ruby
    class User < ActiveRecord::Base
      # 預設的 scope 放在最前面（如果有的話）
      default_scope { where(active: true) }

      # 接下來是常數
      GENDERS = %w(male female)

      # 然後放一些 attr 相關的巨集
      attr_accessor :formatted_date_of_birth

      attr_accessible :login, :first_name, :last_name, :email, :password

      # 緊接著是關聯的巨集
      belongs_to :country

      has_many :authentications, dependent: :destroy

      # 以及巨集的驗證
      validates :email, presence: true
      validates :username, presence: true
      validates :username, uniqueness: { case_sensitive: false }
      validates :username, format: { with: /\A[A-Za-z][A-Za-z0-9._-]{2,19}\z/ }
      validates :password, format: { with: /\A\S{8,128}\z/, allow_nil: true}

      # 接著是回呼
      before_save :cook
      before_save :update_username_lower

      # 其它的巨集 (像 devise 的) 應該放在回呼的後面

      ...
    end
    ```

* 偏好 `has_many :through` 勝於 `has_and_belongs_to_many`。使用 `has_many :through` 允許在 join 資料模型有附加的屬性及驗證

    ```Ruby
    # 使用 has_and_belongs_to_many
    class User < ActiveRecord::Base
      has_and_belongs_to_many :groups
    end

    class Group < ActiveRecord::Base
      has_and_belongs_to_many :users
    end

    # 建議的寫法 - 使用 has_many :through
    class User < ActiveRecord::Base
      has_many :memberships
      has_many :groups, through: :memberships
    end

    class Membership < ActiveRecord::Base
      belongs_to :user
      belongs_to :group
    end

    class Group < ActiveRecord::Base
      has_many :memberships
      has_many :users, through: :memberships
    end
    ```

* 務必使用新的 ["sexy" validation](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/)。
* 如果一個自訂的驗證程序使用超過一次，或驗證程序是透過某個正則表達式的時候，請建立一個自訂的 validator 檔。

    ```Ruby
    # 劣
    class Person
      validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i }
    end

    # 優
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
      end
    end

    class Person
      validates :email, email: true
    end
    ```

* 所有自訂的驗證器應放在一個共享的 gem 。
* 可任意使用具名的作用域 (scope)。

    ```Ruby
    class User < ActiveRecord::Base
      scope :active, -> { where(active: true) }
      scope :inactive, -> { where(active: false) }

      scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
    end
    ```

* 將具名的作用域包在 `lambda` 裡使其延遲初始化。

    ```Ruby
    # 劣
    class User < ActiveRecord::Base
      scope :active, where(active: true)
      scope :inactive, where(active: false)

      scope :with_orders, joins(:orders).select('distinct(users.id)')
    end

    # 優
    class User < ActiveRecord::Base
      scope :active, -> { where(active: true) }
      scope :inactive, -> { where(active: false) }

      scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
    end
    ```

*按：在 Rails 4 會強制使用 lambda*

* 當一個由 lambda 及參數定義的作用域變得過於複雜時，更好的方式是建立一個作為同樣用途的類別方法，並回傳一個 `ActiveRecord::Relation` 物件。你也可以這麼定義更精簡的作用域。

    ```Ruby
    class User < ActiveRecord::Base
      def self.with_orders
        joins(:orders).select('distinct(users.id)')
      end
    end
    ```

* 注意 `update_attribute` 方法的行為。它不會執行資料模型驗證（不同於 `update_attributes` ）並且可能把資料模型狀態給搞砸。
* 使用用戶友好的網址。在網址顯示具描述性的資料模型屬性，而不只是 `id` 。
有不止一種方法可以達成：
  * 覆寫資料模型的 `to_param` 方法。這是 Rails 用來給物件建立網址的方法。預設的實作會以字串形式回傳該 `id` 的記錄。它可以用另一個人類可讀的屬性來覆寫。

        ```Ruby
        class Person
          def to_param
            "#{id} #{name}".parameterize
          end
        end
        ```

    為了要轉換成對網址友好 (URL-friendly)的值，字串應當呼叫 `parameterize` 。物件的 `id` 要放在開頭，以便給 ActiveRecord 的 `find` 方法查找。

  * 使用 `friendly_id` gem。它允許藉由某些具描述性的資料模型屬性來建立人類可讀的網址，而不是用 `id` 。

        ```Ruby
        class Person
          extend FriendlyId
          friendly_id :name, use: :slugged
        end
        ```

        查看 [gem 說明文件](https://github.com/norman/friendly_id)獲得更多關於使用的資訊。

### ActiveResource

* 如果 HTTP 回應是一個與現有的格式（XML 和 JSON）不同的格式，或是需要某些額外的格式解析，這時候請建立一個自訂格式，並在類別中使用它。自訂格式應當實作下列方法：`extension`, `mime_type`,
`encode` 以及 `decode`。

    ```Ruby
    module ActiveResource
      module Formats
        module Extend
          module CSVFormat
            extend self

            def extension
              'csv'
            end

            def mime_type
              'text/csv'
            end

            def encode(hash, options = nil)
              # 資料以新格式編碼並回傳
            end

            def decode(csv)
              # 資料以新格式解碼並回傳
            end
          end
        end
      end
    end

    class User < ActiveResource::Base
      self.format = ActiveResource::Formats::Extend::CSVFormat

      ...
    end
    ```

* 若要讓產生的網址不包含副檔名，請覆寫 `ActiveResource::Base` 的 `element_path` 及 `collection_path` 方法，並移除副檔名。

    ```Ruby
    class User < ActiveResource::Base
      ...

      def self.collection_path(prefix_options = {}, query_options = nil)
        prefix_options, query_options = split_options(prefix_options) if query_options.nil?
        "#{prefix(prefix_options)}#{collection_name}#{query_string(query_options)}"
      end

      def self.element_path(id, prefix_options = {}, query_options = nil)
        prefix_options, query_options = split_options(prefix_options) if query_options.nil?
        "#{prefix(prefix_options)}#{collection_name}/#{URI.parser.escape id.to_s}#{query_string(query_options)}"
      end
    end
    ```

    如有任何改動網址的需求時，這些方法也可以被覆寫。

## 遷移 (Migrations)

* 把 `schema.rb` 放進版本控制系統裡面。
* 用 `rake db:scheme:load` 來初始化空的資料庫，而不是用 `rake db:migrate`。
* 用 `rake db:test:prepare` 來更新測試資料庫的 schema。
* 避免在資料表裡放預設資料。請使用資料模型層來取代。

    ```Ruby
    def amount
      self[:amount] or 0
    end
    ```

    然而 `self[:attr_name]` 卻相當常見，你也可以考慮使用更繁瑣的 `read_attribute` 來取代（有爭議，但更好讀）：

    ```Ruby
    def amount
      read_attribute(:amount) or 0
    end
    ```

* 在寫建設性的遷移時（加表格或加欄位），請使用 Rails 3.1 的新方式 - 使用 `change` 方法取代 `up` 與 `down` 方法。


    ```Ruby
    # 以前的寫法
    class AddNameToPerson < ActiveRecord::Migration
      def up
        add_column :persons, :name, :string
      end

      def down
        remove_column :person, :name
      end
    end

    # 推薦的新寫法
    class AddNameToPerson < ActiveRecord::Migration
      def change
        add_column :persons, :name, :string
      end
    end
    ```

## 視圖 (Views)

* 不要直接從視圖呼叫資料模型層 (Model)。
* 不要在視圖裡做複雜的格式化，把它們寫成方法丟到 helper 或 model 裡面。
* 使用 partial view 與佈局 (layouts) 來減少重複的程式碼。
* 給自訂的檢驗器 (validators) 加上 [瀏覽器端的驗證器](https://github.com/bcardarella/client_side_validations)。方法如下：
  * 宣告一個由 `ClientSideValidations::Middleware::Base` 繼承來的自訂 validator

        ```Ruby
        module ClientSideValidations::Middleware
          class Email < Base
            def response
              if request.params[:email] =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
                self.status = 200
              else
                self.status = 404
              end
              super
            end
          end
        end
        ```

  * 開新檔案
    `public/javascripts/rails.validations.custom.js.coffee` 並且包進 `application.js.coffee` 裡面：

        ```Ruby
        # app/assets/javascripts/application.js.coffee
        #= require rails.validations.custom
        ```

  * 加上瀏覽器端的驗證器：

        ```Ruby
        #public/javascripts/rails.validations.custom.js.coffee
        clientSideValidations.validators.remote['email'] = (element, options) ->
          if $.ajax({
            url: '/validators/email.json',
            data: { email: element.val() },
            async: false
          }).status == 404
            return options.message || 'invalid e-mail format'
        ```

## 國際化 (I18n)

* 視圖、資料模型與控制器裡都不應該使用特定語言的設定值或字串。這些文字應搬到在 `config/locales` 下的語系檔裡。
* 要翻譯 ActiveRecord 資料模型的文字標籤時，請使用 `activerecord` 作用域:

    ```
    zh-TW:
      activerecord:
        models:
          user: "會員"
        attributes:
          user:
            name: "全名"
    ```

    這樣子 `User.model_name.human` 會回傳 "會員" ，而 `User.human_attribute_name("name")` 會回傳 "全名"。這些屬性的翻譯會被視圖作為標籤使用。

* 把在視圖使用的文字與 ActiveRecord 的屬性翻譯分別放在不同的資料夾。把給資料模型使用的語系檔放在名為 `models` 的資料夾，給視圖使用的文字放在名為 `views` 的資料夾。
  * 把額外的語系檔放進各別資料夾之後，要在 `application.rb` 檔裡面指定這些資料夾，才能載入。

        ```Ruby
        # config/application.rb
        config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,ym​​l}')]
        ```

* 把共用的語系選項，像是日期或貨幣格式，直接放在 `locale` 資料夾底下。
* 請使用精簡形式的 I18n 方法： `I18n.t` ，而不是 `I18n.translate` ；使用 `I18n.l` ，而不是 `I18n.localize`。
* 使用「懶惰法」來查詢視圖中使用的文字。假設我們有以下結構：

    ```
    zh-TW:
      users:
        show:
          title: "使用者詳細資料"
    ```

    `users.show.title` 的值在 `app/views/users/show.html.haml` 裡面可以這樣子查到：

    ```Ruby
    = t '.title'
    ```

* 在控制器與資料模型使用「點分隔」的 key，來取代指定 `:scope` 選項。點分隔的呼叫方式，更容易閱讀及追蹤層級。

    ```Ruby
    # 這樣子呼叫
    I18n.t 'activerecord.errors.messages.record_invalid'

    # 而不是這樣
    I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]
    ```

* 關於 Rails i18n 更詳細的資訊可以在 [Rails Guides](http://guides.rubyonrails.org/i18n.html) 找到。

## 資產 (Assets)

在應用程式裡，利用 [Assets Pipeline](http://guides.rubyonrails.org/asset_pipeline.html) 來組織資產。

* 保留 `app/assets` 給自定的樣式表、Javascripts 或圖片。
* 把自己開發，但不屬於應用程式本身的函式庫，放在 `lib/assets`。
* 第三方程式如 [jQuery](http://jquery.com/) 或 [bootstrap](http://twitter.github.com/bootstrap/) 應放在`vendor/assets`。
* 盡可能使用包成 gem 的 assets 。 (如： [jquery-rails](https://github.com/rails/jquery-rails))。

## 郵件 (Mailers)

* 把 mails 命名為 `SomethingMailer`。沒有 Mailer 結尾的話，不能一望即知誰是 Mailer，以及跟哪個視圖有關。
* 要同時提供 HTML 與純文字的視圖模版。
* 在你的開發環境打開寄信失敗時要拋出錯誤的選項。這些錯誤預設是不會拋出的。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* 在開發環境使用 `smtp.gmail.com` 設定 SMTP 伺服器（當然了，除非你自己有本機 SMTP 伺服器）。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      address: 'smtp.gmail.com',
      # 更多設定
    }
    ```

* 要提供預設的主機名稱 (hostname)。

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {host: "#{local_ip}:3000"}


    # config/environments/production.rb
    config.action_mailer.default_url_options = {host: 'your_site.com'}

    # 在你的 mailer 類別
    default_url_options[:host] = 'your_site.com'
    ```

* 如果你需要在 email 裡設超連結到你的網站，務必使用 `_url` 方法，而不是 `_path` 方法。 `_url` 方法包含了主機名稱，而 `_path` 方法則沒有。

    ```Ruby
    # 錯誤
    You can always find more info about this course
    = link_to 'here', url_for(course_path(@course))

    # 正確
    You can always find more info about this course
    = link_to 'here', url_for(course_url(@course))
    ```

* 應把寄件人與收件人地址的格式給寫正確。格式如下：

    ```Ruby
    # 在你的 mailer 類別
    default from: 'Your Name <info@your_site.com>'
    ```

* 確認測試環境的 email 發送方法設定為 `test` ：

    ```Ruby
    # config/environments/test.rb

    config.action_mailer.delivery_method = :test
    ```

* 開發環境與生產環境的發送方法應為 `smtp` ：

    ```Ruby
    # config/environments/development.rb, config/environments/production.rb

    config.action_mailer.delivery_method = :smtp
    ```

* 當發送 HTML email 時，所有樣式應為行內樣式 (inline style)，這是由於某些 email 軟體處理外部樣式表會有問題。雖然這樣會讓程式更難維護、程式碼也容易重覆。有兩個 gem 可以把樣式表轉換成行內樣式，並將放在對應的 html 標籤裡： [premailer-rails3](https://github.com/fphilipe/premailer-rails3) 和 [roadie](https:// github.com/Mange/roadie)。

* 應避免在發送回應的同時同步寄出 email，因為這樣會造成網頁載入時間過久、而且要是寄送多個 email 還可能會造成連線逾時。請使用 [delayed_job](https://github.com/tobi/delayed_job) gem 來把寄送 email 放到背景去處理。

## 打包 (Bundler)

* 把只給開發環境或測試環境的 gem 在 Gemfile 檔裡面妥善分組。
* 在你的專案中只使用公認的 gem。如果你考慮引入某些鮮為人所知的 gem ，你應該先仔細審查它的原始碼。
* 要是開發人員各自使用不同的作業系統，那麼與作業系統相關的那些 gem 會導致 `Gemfile.lock` 經常變動。解決方法是，在 Gemfile 裡，把與 OS X 相關的 gem 放在 `darwin` 群組，與 Linux 相關的 gem 放在 `linux` 群組：

    ```Ruby
    # Gemfile
    group :darwin do
      gem 'rb-fsevent'
      gem 'growl'
    end

    group :linux do
      gem 'rb-inotify'
    end
    ```

    要在正確的環境 require 正確的 gem，請新增以下程式碼至 `config/application.rb` ：

    ```Ruby
    platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
    Bundler.require(platform)
    ```

* 不要把 `Gemfile.lock` 檔從版本控制系統裡移出。這不是隨機產生的文件──它確保你所有的成員執行 `bundle install` 時，都拿到相同版本的 gem 。

## 貴重的 Gems

一個最重要的程式設計理念是「不要重造輪子！」。若你遇到一個特定問題，你應該要在你開始手刻之前，找一下是否有現有的解決方案。以下是一些在很多 Rails 專案中的「無價至寶」 gem 列表（全部相容 Rails 3.1）：

* [active_admin](https://github.com/gregbell/active_admin) - 有了 ActiveAdmin，建立 Rails 應用的管理介面就像兒戲。你會有一個很好的後台，圖形化 CRUD 介面以及更多東西。非常靈活且可客製化。
* [better_errors](https://github.com/charliesome/better_errors) - Better Errors 用更好更有效的錯誤頁面，取代了 Rails 標準的錯誤頁面。不僅可用在 Rails，任何將 Rack 當作 middleware 的 app 都可使用。
* [bullet](https://github.com/flyerhzm/bullet) - Bullet 就是為了幫助你提升應用程式的效能而打造的 gem （藉由減少資料庫查詢）。會在你開發應用程式時，替你注意你的資料庫查詢，並在需要 eager loading (N+1 查詢) 時、或你在不必要的情況使用 eager loading 時，或是在應該要使用 counter cache 時，都會提醒你。
* [cancan](https://github.com/ryanb/cancan) - CanCan 是一個權限管理的 gem，
讓你可以管制用戶可存取的資源。所有的權限都定義在一個檔案裡（ability.rb），並提供許多方便的方法，讓你在整個應用程式裡都可以檢查及確保權限是否獲准。
* [capybara](https://github.com/jnicklas/capybara) - Capybara 旨在簡化整合測試 Rack 應用程式的流程，像是 Rails、Sinatra 或 Merb。 Capybara 模擬了真實用戶使用 web 應用程式的互動過程。它與你用的測試工具無關，並原生搭載 Rack::Test 及 Selenium 支援。透過外部 gem 支援 HtmlUnit、WebKit 及 env.js 。與 RSpec & Cucumber 一起使用時工作良好。
* [carrierwave](https://github.com/jnicklas/carrierwave) - Rails 最新的檔案上傳的解決方案。支援上傳檔案到本地儲存與雲端儲存（及很多其它的酷玩意）。良好結合了 ImageMagick 來進行圖片後處理。
* [client_side_validations](https://github.com/bcardarella/client_side_validations) -
  一個很棒的 gem，替你從現有的伺服器端資料模型驗證，自動產生 Javascript 瀏覽器端驗證。強烈推薦！
* [compass-rails](https://github.com/chriseppstein/compass) - 一個優秀的 gem，加入了某些 css 框架的支持。包括了一些 sass mixin ，讓你減少 css 檔的程式碼，並幫你解決瀏覽器相容問題。
* [cucumber-rails](https://github.com/cucumber/cucumber-rails) - Cucumber 是一個由 Ruby 所寫，開發功能測試的頂級工具。 cucumber-rails 提供了 Cucumber 的 Rails 整合。
* [devise](https://github.com/plataformatec/devise) - Devise 是 Rails 應用程式的登入系統完整解決方案。多數情況偏好使用 devise 來開始客製登入流程。
* [fabrication](http://fabricationgem.org/) - 一個很好的 fixture 測資替代品（編輯推薦）。
* [factory_girl](https://github.com/thoughtbot/factory_girl) - Fabrication 的替代品。一個成熟的 fixture 測資產生器。 Fabrication 的精神領袖先驅。
* [ffaker](https://github.com/EmmanuelOga/ffaker) - 產生假資料的實用 gem（名字、地址，等等）。
* [feedzirra](https://github.com/pauldix/feedzirra) - 非常快速、靈活的 RSS / Atom Feed 解析器。
* [friendly_id](https://github.com/norman/friendly_id) - 透過使用某些具描述性的資料模型屬性，而不是使用 id，來讓你建立人類可讀的網址。
* [globalize3](https://github.com/svenfuchs/globalize3.git) - Globalize3 是 Globalize Gem 的後繼者，針對 ActiveRecord 3.x 設計。基於新的 I18n API 打造而成，並幫 ActiveRecord 資料模型新增了交易功能 (transaction)。
* [guard](https://github.com/guard/guard) - 監控檔案變化並呼叫任務的極佳 gem。搭載了很多實用的擴充。樂勝 autotest 與 [watchr](https://github.com/mynyml/watchr)。
* [haml-rails](https://github.com/indirect/haml-rails) - haml-rails 提供了 Haml 的 Rails 整合。
* [haml](http://haml-lang.com) - Haml 是一個簡潔的資料模型語言，被很多人認為（包括我）遠優於Erb。
* [kaminari](https://github.com/amatsuda/kaminari) - 很棒的分頁解決方案。
* [machinist](https://github.com/notahat/machinist) - fixture 測資不好玩，Machinist 才好玩。
* [rspec-rails](https://github.com/rspec/rspec-rails) - RSpec 是 Test::MiniTest 的替代品。我不高度推薦 RSpec。 rspec-rails 提供了 RSpec 的 Rails 整合。
* [simple_form](https://github.com/plataformatec/simple_form) - 一旦用過 simple_form（或 formatastic），你就回不去 Rails 預設的表單產生器了。它提供很棒的 DSL 可以建立表單，讓你不必在意表單的 HTML 怎麼寫。
* [simplecov-rcov](https://github.com/fguillen/simplecov-rcov) - 為了 SimpleCov 打造的 RCov formatter。若你想使用 SimpleCov 搭配 Hudson 持續整合伺服器 (CI Server)，它很有用。
* [simplecov](https://github.com/colszowka/simplecov) - 檢查程式碼覆蓋率 (code coverage) 的工具。但不像 RCov，它完全相容 Ruby 1.9。它有精美的報表。必須用！
* [slim](http://slim-lang.com) - Slim 是一個簡潔的模版語言，被視為是遠遠優於 HAML 的程式語言 (至於 Erb 就不用說了) 。唯一會阻止我大規模地使用它的是，主流 IDE 及編輯器對它的支援不好。但它的效能是非凡的。
* [spork](https://github.com/sporkrb/spork) - 一個給測試框架（RSpec / Cucumber）用的 DRb 伺服器，每次運行前確保 fork 出一個乾淨的測試狀態。簡單的說，預載很多測試環境的結果是大幅降低你的測試啟動時間，絕對必須用！
* [sunspot](https://github.com/sunspot/sunspot) - 基於 SOLR 的全文搜尋引擎。

這不是完整的清單，其它的 gem 也可以在之後加進來。以上清單上的所有 gems 皆經測試，處於活躍開發階段，有社群，程式碼的品質很高。

## 有缺陷的 Gems

這是一個有問題的或被別的 gem 取代的 gem 清單。你應該在你的專案裡避免使用它們。

* [rmagick](http://rmagick.rubyforge.org/) - 這個 gem 因大量消耗記憶體而聲名狼藉。請改用 [minimagick](https://github.com/probablycorey/mini_magick)。
* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - 自動化測試的舊方法。遠不如 guard 及 [watchr](https://github.com/mynyml/watchr)。
* [rcov](https://github.com/relevance/rcov) - 程式碼覆蓋率工具，不相容於 Ruby 1.9。請改用 [SimpleCov](https://github.com/colszowka/simplecov)。
* [therubyracer](https://github.com/cowboyd/therubyracer) - 極度不鼓勵在生產模式使用這個 gem，它會消耗大量的記憶體。我會推薦改用 `node.js`。

這仍是一個完善中的清單。請告訴我受人歡迎但有缺陷的 gems 。

## 管理處理程序 (process)

* 若你的專案依賴各種外部的處理程序，使用 [foreman](https://github.com/ddollar/foreman) 來管理它們。

# 測試 Rails 應用程式

也許 BDD 方法是實作一個新功能最好的方法。你從開始寫一些高階的測試（通常使用 Cucumber），然後使用這些測試來驅使你實作功能。一開始你給功能的視圖寫測試，並使用這些測試來建立相關的視圖。接著，你寫控制器測試要求把資料丟給視圖用，藉此來實作控制器。最後你實作資料模型的測試，以及資料模型自身。

## Cucumber

* 用 `@wip` （工作進行中）標籤來標記尚未完成的情境 (scenario)。這些情境將不納入考慮，且不會被標記為測試失敗。當完成這個情境且功能測試通過時，為了把此情境加至測試套件裡，請移除 `@wip` 標籤。
* 修改預設的 profile，排除掉標記為 `@javascript` 的情境。它們將使用瀏覽器來測試，建議停用它們來增進一般情境的執行速度。
* 替標記著 `@javascript` 的情境，設定另一個 profile。
  * profile 可在 `cucumber.yml` 檔案裡設定。

        ```Ruby
        # profile 的定義：
        profile_name: --tags @tag_name
        ```

  * 用這個指令來執行特定的 profile：

        ```
        cucumber -p profile_name
        ```

* 若使用 [fabrication](http://fabricationgem.org/) 來替換 fixtures，請使用預先定義的 [fabrication steps](http://fabricationgem.org/#!cucumber-steps)。
* 不要使用舊的 `web_steps.rb` 來定義步驟！[最新版 Cucumber 已移除 web steps](http://aslakhellesoy.com/post/11055981222/the-training-wheels-came-off) ，用這個會導致多餘的情境，這些情境無法正確反映出應用程式的領域。
* 當檢查一元素的可見文字時（如超連結、按鈕），請檢查元素的文字而不是檢查 id。這樣可以查出 i18n 的問題。
* 為同物件的各種功能，各自建立不同的 feature：

    ```Ruby
    # 劣
    Feature: Articles
    # ... 功能實作 ...

    # 優
    Feature: Article Editing
    # ... 功能實作 ...

    Feature: Article Publishing
    # ... 功能實作 ...

    Feature: Article Search
    # ... 功能實作 ...

    ```

* 每一個 feature 有三個主要成分：
  * Title
  * Narrative - 簡短說明這個 feature 關於什麼。
  * Acceptance criteria - 每個由獨立步驟組成的一套情境。
* 最常見的格式稱為 Connextra 格式。

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

這種格式最常見，但並不強求要這樣寫， narrative 敘述句可以因功能的複雜度而任意書寫。

* 可任意使用情境概述使你的情境可備作它用(keep your scenarios DRY)。

    ```Ruby
    Scenario Outline: User cannot register with invalid e-mail
      When I try to register with an email "<email>"
      Then I should see the error message "<error>"

    Examples:
      |email |error |
      | |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* 情境的步驟放在 `step_definitions` 目錄下的 `.rb` 檔。步驟檔命名慣例為 `[description]_steps.rb`。步驟根據不同的標準放在不同的檔案裡。每一個 feature 可能有一個步驟檔(`home_page_steps.rb`)
。也可以給每個特定物件的 feature，開一個步驟檔(`articles_steps.rb`)。
* 使用多行步驟參數來避免重複

    ```Ruby
    Scenario: User profile
      Given I am logged in as a user "John Doe" with an e-mail "user@test.com"
      When I go to my profile
      Then I should see the following information:
        |First name|John |
        |Last name |Doe |
        |E-mail |user@test.com|

    # 步驟:
    Then /^I should see the following information:$/ do |table|
      table.raw.each do |field, value|
        find_field(field).value.should =~ /#{value}/
      end
    end
    ```

* 使用複合步驟來讓情境可備作它用(Keep your scenarios DRY)

    ```Ruby
    # ...
    When I subscribe for news from the category "Technical News"
    # ...

    # 步驟:
    When /^I subscribe for news from the category "([^"]*)"$/ do |category|
      steps %Q{
        When I go to the news categories page
        And I select the category #{category}
        And I click the button "Subscribe for this category"
        And I confirm the subscription
      }
    end
    ```
* 務必使用 Capybara 的否定配對來取代在肯定情況裡使用 should_not，這樣子當 ajax 操作逾時就會重試。見 [Capybara 的 README 檔](https://github.com/jnicklas/capybara)獲得更多說明。

## RSpec

* 每個測試案例應只有一個期望值 (expection)。

    ```Ruby
    # 劣
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns new article and renders the new article template' do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # 優
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns a new article' do
          get :new
          assigns[:article].should be_a_new Article
        end

        it 'renders the new article template' do
          get :new
          response.should render_template :new
        end
      end

    end
    ```

* 應大量使用 `describe` 及 `context` 。
* `describe` 區塊的命名方式應如下：
  * 非方法使用 "description"
  * 實體方法使用井字號 "#method"
  * 類別方法使用點 ".method"

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article do
      describe '#summary' do
        #...
      end

      describe '.latest' do
        #...
      end
    end
    ```

* 使用 [fabricators](http://fabricationgem.org/) 來建立測資物件。

* 應大量使用 mocks 與 stubs。

    ```Ruby
    # mocking 一個資料模型
    article = mock_model(Article)

    # stubbing 一個方法
    Article.stub(:find).with(article.id).and_return(article)
    ```

* 當 mocking 一個資料模型時，可以使用 `as_null_object` 方法，讓輸出的物件只回應我們有 stub 的方法，不理會其他方法。

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* 使用 `let` 區塊，不要用 `before(:each)` 區塊來為 spec 測試案例建立資料。 `let` 區塊會被延遲求值 (lazily evaluated)。

    ```Ruby
    # 使用這個：
    let(:article) { Fabricate(:article) }

    # ... 而不是這個：
    before(:each) { @article = Fabricate(:article) }
    ```

* 盡可能使用 `subject`。

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* 盡可能使用 `specify`。它是 `it` 的同義詞，但在沒 docstring 的情況下更好讀。

    ```Ruby
    # 劣
    describe Article do
      before { @article = Fabricate(:article) }

      it 'is not published on creation' do
        @article.should_not be_published
      end
    end

    # 優
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* 盡可能使用 `its` 。

    ```Ruby
    # 劣
    describe Article do
      subject { Fabricate(:article) }

      it 'has the current date as creation date' do
        subject.creation_date.should == Date.today
      end
    end

    # 優
    describe Article do
      subject { Fabricate(:article) }
      its(:creation_date) { should == Date.today }
    end
    ```

* 如果要建立共用的 spec 群組，請使用 `shared_examples`。

   ```Ruby
   # 劣
    describe Array do
      subject { Array.new [7, 2, 4] }

      context "initialized with 3 items" do
        its(:size) { should eq(3) }
      end
    end

    describe Set do
      subject { Set.new [7, 2, 4] }

      context "initialized with 3 items" do
        its(:size) { should eq(3) }
      end
    end

   # 優
    shared_examples "a collection" do
      subject { described_class.new([7, 2, 4]) }

      context "initialized with 3 items" do
        its(:size) { should eq(3) }
      end
    end

    describe Array do
      it_behaves_like "a collection"
    end

    describe Set do
      it_behaves_like "a collection"
    end


### 視圖 (Views)

* 視圖測試的目錄結構要與 `app/views` 裡面的結構一致。舉例來說， `app/views/users` 的視圖測試應放在 `spec/views/users`。
* 視圖測試的命名慣例是把 `_spec.rb` 加到視圖名稱的後面，舉例來說，視圖 `_form.html.haml` 有一個對應的測試叫做 `_form.html.haml_spec.rb`。
* 每個視圖測試檔都需要 `spec_helper.rb`。
* 外圍的 `describe` 區塊要使用不包含 `app/views` 前綴的視圖路徑，這在 `render` 方法沒有傳入參數的時候會用到。

    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* 務必要在視圖測試裡面 mock 資料模型。視圖的目的只有顯示資訊而已。
* 原本由控制器提供給視圖使用的實體變數(instance variable)，可以用 `assign` 方法來提供。

    ```Ruby
    # spec/views/articles/edit.html.haml_spec.rb
    describe 'articles/edit.html.haml' do
    it 'renders the form for a new article creation' do
      assign(
        :article,
        mock_model(Article).as_new_record.as_null_object
      )
      render
      rendered.should have_selector('form',
        method: 'post',
        action: articles_path
      ) do |form|
        form.should have_selector('input', type: 'submit')
      end
    end
    ```

* 最好使用 capybara 的否定情況選擇器，而非 should_not 配上正面情況。

    ```Ruby
    # 劣
    page.should_not have_selector('input', type: 'submit')
    page.should_not have_xpath('tr')

    # 優
    page.should have_no_selector('input', type: 'submit')
    page.should have_no_xpath('tr')
    ```

* 當視圖要使用 helper 方法時，要先把這些方法給 stub 掉，這件事要在 `template` 物件裡面做：

    ```Ruby
    # app/helpers/articles_helper.rb
    class ArticlesHelper
      def formatted_date(date)
        # ...
      end
    end

    # app/views/articles/show.html.haml
    = "Published at: #{formatted_date(@article.published_at)}"

    # spec/views/articles/show.html.haml_spec.rb
    describe 'articles/show.html.haml' do
      it 'displays the formatted date of article publishing' do
        article = mock_model(Article, published_at: Date.new(2012, 01, 01))
        assign(:article, article)

        template.stub(:formatted_date).with(article.published_at).and_return('01.01.2012')

        render
        rendered.should have_content('Published at: 01.01.2012')
      end
    end
    ```

* helper specs 測試檔要要從視圖 specs 測試裡面拆出來，放在 `spec/helpers` 目錄下。

### 控制器

* 請 mock 資料模型並 stub 他們的方法。測試控制器時不應依賴於資料模型的建立。
* 請只測試控制器需負責的行為：
  * 某幾個特定方法的執行
  * 從動作 (action) 回傳的資料 - assigns, 等等。
  * 動作所產生的結果 - template render, redirect, 等等。

        ```Ruby
        # 常用的控制器 spec 範例
        # spec/controllers/articles_controller_spec.rb
        # 我們只對控制器應執行的動作感興趣
        # 所以我們 mock 資料模型及 stub 它的方法
        # 並且專注在控制器該做的事情上

        describe ArticlesController do
          # 資料模型將會在測試中被所有控制器的方法所使用
          let(:article) { mock_model(Article) }

          describe 'POST create' do
            before { Article.stub(:new).and_return(article) }

            it 'creates a new article with the given attributes' do
              Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
              post :create, message: { title: 'The New Article Title' }
            end

            it 'saves the article' do
              article.should_receive(:save)
              post :create
            end

            it 'redirects to the Articles index' do
              article.stub(:save)
              post :create
              response.should redirect_to(action: 'index')
            end
          end
        end
        ```

* 當控制器根據不同參數有不同行為時，請使用 context。

    ```Ruby
    # 一個在控制器中使用 context 的典型例子是，建立或更新物件時，可能因為儲存成功與否而有不同行為。

    describe ArticlesController do
      let(:article) { mock_model(Article) }

      describe 'POST create' do
        before { Article.stub(:new).and_return(article) }

        it 'creates a new article with the given attributes' do
          Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
          post :create, article: { title: 'The New Article Title' }
        end

        it 'saves the article' do
          article.should_receive(:save)
          post :create
        end

        context 'when the article saves successfully' do
          before { article.stub(:save).and_return(true) }

          it 'sets a flash[:notice] message' do
            post :create
            flash[:notice].should eq('The article was saved successfully.')
          end

          it 'redirects to the Articles index' do
            post :create
            response.should redirect_to(action: 'index')
          end
        end

        context 'when the article fails to save' do
          before { article.stub(:save).and_return(false) }

          it 'assigns @article' do
            post :create
            assigns[:article].should be_eql(article)
          end

          it 're-renders the "new" template' do
            post :create
            response.should render_template('new')
          end
        end
      end
    end
    ```

### 資料模型

* 不要在資料模型自己的測試裡 mock 該資料模型。
* 使用 fabrication 來建立實際的物件
* 可以 mock 別的資料模型或子物件。
* 為避免重覆，請在測試裡建立可以給所有測試案例使用的資料模型。

    ```Ruby
    describe Article do
      let(:article) { Fabricate(:article) }
    end
    ```

* 新增一個測試案例來確保 fabrication 做出來的資料模型是可以用的。

    ```Ruby
    describe Article do
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* 寫跟驗證程序有關的測試案例時，請使用 `have(x).errors_on` 來指定要被驗證的屬性。使用 `be_valid` 並不能保證問題一定會發生在要被驗證的屬性。

    ```Ruby
    # 劣
    describe '#title' do
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # 推薦使用
    describe '#title' do
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

* 替每個有驗證程序的屬性，另外加另一個 `describe`。

    ```Ruby
    describe Article do
      describe '#title'
        it 'is required' do
          article.title = nil
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

* 當測試資料模型屬性的唯一性時，將另一個重覆的物件命名為 `another_object`。

    ```Ruby
    describe Article do
      describe '#title' do
        it 'is unique' do
          another_article = Fabricate.build(:article, title: article.title)
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

### Mailers

* 在 Mailer 測試的資料模型應該要被 mock 掉。 Mailer 不應依賴資料模型的建立。
* Mailer 的測試應該要檢驗這些：
  * 主旨正確
  * 收件人 e-mail 正確
  * e-mail 有寄送至正確的 e-mail 地址
  * e-mail 有包含所要寄送的訊息

     ```Ruby
     describe SubscriberMailer do
       let(:subscriber) { mock_model(Subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email' do
         subject { SubscriptionMailer.successful_registration_email(subscriber) }

         its(:subject) { should == 'Successful Registration!' }
         its(:from) { should == ['info@your_site.com'] }
         its(:to) { should == [subscriber.email] }

         it 'contains the subscriber name' do
           subject.body.encoded.should match(subscriber.name)
         end
       end
     end
     ```

### Uploaders

* 我們可以測試上傳的圖片是否有正確產生縮圖。以下是 [carrierwave](https://github.com/jnicklas/carrierwave) 圖片上傳器的範例 spec：

    ```Ruby
    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # 在執行測試案例之前，先打開圖片處理
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # 建立一個新的 uploader。要把資料模型給 mock 掉，使上傳及縮圖的時候不會依賴於資料模型的建立。
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # 執行完測試案例時，關閉圖片處理
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # 測試縮圖是否不比給定的尺寸大
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # 測試縮圖是否有完全一致的尺寸
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end
    ```

# 延伸閱讀

有幾個絕妙講述 Rails 風格的資源，若有閒暇時應當考慮閱讀之：

* [The Rails 3 Way](http://www.amazon.com/Rails-Way-Addison-Wesley-Professional-Ruby/dp/0321601661)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)
* [The Cucumber Book](http://pragprog.com/book/hwcuc/the-cucumber-book)
* [Everyday Rails Testing with RSpec](https://leanpub.com/everydayrailsrspec)

# 貢獻

在本指南所寫的每個東西都不是定案。這只是我渴望想與同樣對 Rails 程式設計風格有興趣的大家一起工作，這樣子最終我們可以創造出對整個 Ruby 社群都有益的資源。

歡迎開票或發送一個帶有改進的 Pull Request。在此提前感謝你的幫助！

# 授權

![Creative Commons License](http://i.creativecommons.org/l/by/3.0/88x31.png)
This work is licensed under a [Creative Commons Attribution 3.0 Unported License](http://creativecommons.org/licenses/by/3.0/deed.zh_TW)

# 口耳相傳

一份社群驅動的風格指南，對於沒聽過這份指南的其他社群人士來說，幾乎沒什麼用。請上 Twitter 轉貼這份指南，分享給你的朋友或同事。我們得到的每個註解、建議或意見都可以讓這份指南變得更好一點。而我們都想要有最好的指南，對吧？

共勉之，<br/>
[Bozhidar](https://twitter.com/bbatsov)
