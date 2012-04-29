# 序幕

> 風格是從偉大事物中分離出的美好事物。 <br/>
> -- Bozhidar Batsov

這份指南目的於演示一整套 Rails 3 開發的風格慣例及最佳實踐。這是一份與由現存社群所驅動的 [Ruby 編碼風格指南](https://github.com/bbatsov/ruby-style-guide)互補的指南。

而本指南中[測試 Rails 應用](#testing)小節擺在[開發 Rails 應用](#developing)之後，因為我相信[行為驅動開發](http://en.wikipedia.org/wiki/Behavior_Driven_Development)
(BDD) 是最佳的軟體開發之道。銘記在心吧。

Rails 是一個堅持己見的框架，而這也是一份堅持己見的指南。在我的心裡，我堅信 [RSpec](https://www.relishapp.com/rspec) 優於 Test::Unit，[Sass](http://sass-lang.com/) 優於 CSS 以及
[Haml](http://haml-lang.com/)，([Slim](http://slim-lang.com/)) 優於 Erb。所以不要期望在這裡找到 Test::Unit, CSS 及 Erb 的忠告。

某些忠告僅適用於 Rails 3.1+ 以上版本。

你可以使用 [Transmuter](https://github.com/TechnoGate/transmuter) 來產生本指南的一份 PDF 或 HTML 複本。

# 開發 Rails 應用程式

## 配置

* 把慣用的初始化程式碼放在 `config/initializers`。在 initializers 內的程式碼於應用啟動時執行。
* 每一個 gem 相關的初始化程式碼應當使用同樣的名稱，放在不同的文件裡，如： `carrierwave.rb`, `active_admin.rb`, 等等。
* 相應調整配置開發、測試及生產環境（在 `config/environments/` 下對應的文件）
  * 標記額外的資產給（如有任何）預編譯：

        ```Ruby
        # config/environments/production.rb
        # 預編譯額外的資產(application.js, application.css, 以及所有非 JS 或 CSS 的檔案)
        config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
        ```

* 創立一個與生產環境(production enviroment)相似的額外 `staging` 環境。

## 路由

* 當你需要加入一個或多個動作至一個 RESTful 資源時（你真的需要嗎？），使用 `member` and `collection` 路由。

    ```Ruby
    # 差
    get 'subscriptions/:id/unsubscribe'
    resources :subscriptions

    # 好
    resources :subscriptions do
      get 'unsubscribe', :on => :member
    end

    # 差
    get 'photos/search'
    resources :photos

    # 好
    resources :photos do
      get 'search', :on => :collection
    end
    ```

* 若你需要定義多個 `member/collection` 路由時，使用替代的區塊語法(block syntax)。

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

* 使用巢狀路由(nested routes)來更佳地表達與 ActiveRecord 模型的關係。

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

* 不要在控制器裡使用留給後人般的瘋狂路由(legacy wild controller route)。這種路由會讓每個控制器的動作透過 GET 請求存取。

    ```Ruby
    # 非常差
    match ':controller(/:action(/:id(.:format)))'
    ```

## 控制器

* 讓你的控制器保持苗條 ― 它們應該只替視圖層取出資料且不包含任何業務邏輯（所有業務邏輯應當放在模型裡）。
* 每個控制器的行動應當（理想上）只調用一個除了初始的 find 或 new 方法。
* 控制器與視圖之間共享不超過兩個實體變數(instance variable)。

## 模型

* 自由地引入不是 ActiveRecord 的類別吧。
* 替模型命名有意義（但簡短）且不帶縮寫的名字。
* 如果你需要支援 ActiveRecord 像是驗證行為的模型物件，使用 [ActiveAttr](https://github.com/cgriego/active_attr) gem。

    ```Ruby
    class Message
      include ActiveAttr::Model

      attribute :name
      attribute :email
      attribute :content
      attribute :priority

      attr_accessible :name, :email, :content

      validates_presence_of :name
      validates_format_of :email, :with => /^[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4} $/i
      validates_length_of :content, :maximum => 500
    end
    ```

    更完整的範例，參考 [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr)。

### ActiveRecord

* 避免改動預設的 ActiveRecord（表的名字、主鍵，等等），除非你有一個非常好的理由（像是不受你控制的資料庫）。
* 把巨集風格的方法放在類別定義的前面（`has_many`, `validates`, 等等）。
* 偏好 `has_many :through` 勝於 `has_and_belongs_to_many`。使用 `has_many :through` 允許在 join 模型有附加的屬性及驗證

    ```Ruby
    # 使用 has_​​and_belongs_to_many
    class User < ActiveRecord::Base
      has_and_belongs_to_many :groups
    end

    class Group < ActiveRecord::Base
      has_and_belongs_to_many :users
    end

    # 偏好方式 - using has_​​many :through
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

* 使用新的 ["sexy" validation](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/)。
* 當一個慣用的驗證使用超過一次或驗證是某個正則表達映射時，創建一個慣用的 validator 文件。

    ```Ruby
    # 差
    class Person
      validates :email, format: { with: /^([^@\s]+)@((?:[-a-z0-9]+\.)+[az]{2,})$/i }
    end

    # 好
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /^([^@\s]+)@((?:[-a-z0- 9]+\.)+[az]{2,})$/i
      end
    end

    class Person
      validates :email, email: true
    end
    ```

* 所有慣用的驗證器應放在一個共享的 gem 。
* 自由地使用命名的作用域(scope)。
* 當一個由 lambda 及參數定義的作用域變得過於復雜時，更好的方式是建一個作為同樣用途的類別方法，並返回 `ActiveRecord::Relation` 物件。
* 注意 `update_attribute` 方法的行為。它不運行模型驗證（不同於 `update_attributes` ）並且可能把模型狀態給搞砸。
* 使用用戶友好的網址。在網址顯示具描述性的模型屬性，而不只是 `id` 。
有不止一種方法可以達成：
  * 覆寫模型的 `to_param` 方法。這是 Rails 用來給物件建構網址的方法。預設的實作會以字串形式返回該 `id` 的記錄。它可被另一個具人類可讀的屬性覆寫。

        ```Ruby
        class Person
          def to_param
            "#{id} #{name}".parameterize
          end
        end
        ```
        為了要轉換成對網址友好(URL-friendly)的數值，字串應當調用 `parameterize` 。物件的 `id` 要放在開頭，以便給 ActiveRecord 的 `find` 方法查找。

  * 使用此 `friendly_id` gem。它允許藉由某些具描述性的模型屬性，而不是用 `id` 來創建人類可讀的網址。

        ```Ruby
        class Person
          extend FriendlyId
          friendly_id :name, use: :slugged
        end
        ```

        查看 [gem 說明文件](https://github.com/norman/friendly_id)獲得更多關於使用的訊息。

### ActiveResource

* 當 HTTP 響應是一個與存在的格式不同的格式時（XML 和 JSON），需要某些額外的格式解析，創一個你慣用的格式，並在類別中使用它。慣用的格式應當實作下列方法：`extension`, `mime_type`,
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
              # 資料以新格式編碼並返回
            end

            def decode(csv)
              # 資料以新格式解碼並返回
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

* 若 HTTP 請求應當不擴展發送時，覆寫 `ActiveResource::Base` 的 `element_path` 及 `collection_path` 方法，並移除擴展的部份。

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

## 遷移

* 把 `schema.rb` 保存在版本管控之下。
* 使用 `rake db:scheme:load` 取代 `rake db:migrate` 來初始化空的資料庫。
* 使用 `rake db:test:prepare` 來更新測試資料庫的 schema。
* 避免在表裡設置預設資料。使用模型層來取代。

    ```Ruby
    def amount
      self[:amount] or 0
    end
    ```

    然而 `self[:attr_name]` 的使用被視為相當常見的，你也可以考慮使用更繁瑣的（爭議地可讀性更高的） `read_attribute` 來取代：

    ```Ruby
    def amount
      read_attribute(:amount) or 0
    end
    ```

* 當編寫建設性的遷移時（加入表或欄位），使用 Rails 3.1 的新方式來遷移 - 使用 `change` 方法取代 `up` 與 `down` 方法。


    ```Ruby
    # 過去的方式
    class AddNameToPerson < ActiveRecord::Migration
      def up
        add_column :persons, :name, :string
      end

      def down
        remove_column :person, :name
      end
    end

    # 新的偏好方式
    class AddNameToPerson < ActiveRecord::Migration
      def change
        add_column :persons, :name, :string
      end
    end
    ```

## 視圖

* 不要直接從視圖調用模型層。
* 不要在視圖構造複雜的格式，把它們輸出到視圖 helper 的一個方法或是模型。
* 使用 partial 模版與佈局來減少重複的程式碼。
* 加入 [client side validation](https://github.com/bcardarella/client_side_validations) 至慣用的 validators。要做的步驟有：
  * 聲明一個由 `ClientSideValidations::Middleware::Base` 而來的自定 validator

        ```Ruby
        module ClientSideValidations::Middleware
          class Email < Base
            def response
              if request.params[:email] =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[az]{2,})$/i
                self.status = 200
              else
                self.status = 404
              end
              super
            end
          end
        end
        ```

  * 建立一個新文件
    `public/javascripts/rails.validations.custom.js.coffee` 並在你的 `application.js.coffee` 文件加入一個它的參照：

        ```Ruby
        # app/assets/javascripts/application.js.coffee
        #= require rails.validations.custom
        ```

  * 添加你的用戶端 validator：

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

## 國際化

* 視圖、模型與控制器裡不應使用語言相關設置與字串。這些文字應搬到在 `config/locales` 下的語言文件裡。
* 當 ActiveRecord 模型的標籤需要被翻譯時，使用 `activerecord` 作用域:

    ```
    en:
      activerecord:
        models:
          user: Member
        attributes:
          user:
            name: "Full name"
    ```

    然後 `User.model_name.human` 會返回 "Member" ，而 `User.human_attribute_name("name")` 會返回 "Full name"。這些屬性的翻譯會被視圖作為標籤使用。

* 把在視圖使用的文字與 ActiveRecord 的屬性翻譯分開。把給模型使用的語言文件放在名為 `models` 的文件夾，給視圖使用的文字放在名為 `views` 的文件夾。
  * 當使用額外目錄的語言文件組織完成時，為了要載入這些目錄，要在 `application.rb` 文件裡描述這些目錄。

        ```Ruby
        # config/application.rb
        config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,ym​​l}')]
        ```

* 把共享的本土化選項，像是日期或​​貨幣格式，放在 `locales` 的根目錄下。
* 使用精簡形式的 I18n 方法： `I18n.t` 來取代 `I18n.translate` 以及使用 `I18n.l` 取代 `I18n.localize`。
* 使用 "懶惰" 查詢視圖中使用的文字。假設我們有以下結構：

    ```
    en:
      users:
        show:
          title: "User details page"
    ```

    `users.show.title` 的數值能這樣被`app/views/users/show.html.haml` 查詢：

    ```Ruby
    = t '.title'
    ```

* 在控制器與模型使用點分隔的鍵，來取代指定 `:scope` 選項。點分隔的調用更容易閱讀及追踪層級。

    ```Ruby
    # 這樣子調用
    I18n.t 'activerecord.errors.messages.record_invalid'

    # 而不是這樣
    I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]
    ```

* 關於 Rails i18n 更詳細的訊息可以在這裡找到 [Rails Guides](http://guides.rubyonrails.org/i18n.html)。

## Assets

利用這個 [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) 來管理應用的結構。

* 保留 `app/assets` 給自定的樣式表, javascripts, 或圖片。
* 第三方程式碼如： [jQuery](http://jquery.com/) 或 [bootstrap](http://twitter.github.com/bootstrap/) 應放置在`vendor/assets`。
* 當可能的時候，使用 gem 化的 assets 版本。 (如： [jquery-rails](https://github.com/rails/jquery-rails))。

## Mailers

* 把 mails 命名為 `SomethingMailer`。沒有 Mailer 字根的話，不能立即顯現哪個是一個 Mailer，以及哪個視圖與它有關。
* 提供 HTML 與純文本視圖模版。
* 在你的開發環境啟用信件失敗發送錯誤。這些錯誤預設是被停用的。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* 在開發模式使用 `smtp.gmail.com` 設置 SMTP 服務器（當然了，除非你自己有本機 SMTP 服務器）。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      address: 'smt​​p.gmail.com',
      # 更多設置
    }
    ```

* 提供預設的配置給主機名。

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {host: "#{local_ip}:3000"}


    # config/environments/production.rb
    config.action_mailer.default_url_options = {host: 'your_site.com'}

    # 在你的 mailer 類
    default_url_options[:host] = 'your_site.com'
    ```

* 如果你需要在你的網站使用一個 email 鏈結，總是使用 `_url` 方法，而不是 `_path` 方法。 `_url` 方法包含了主機名，而 `_path` 方法沒有。

    ```Ruby
    # 錯誤
    You can always find more info about this course
    = link_to 'here', url_for(course_path(@course))

    # 正確
    You can always find more info about this course
    = link_to 'here', url_for(course_url(@course))
    ```

* 正確地顯示寄與收件人地址的格式。使用下列格式：

    ```Ruby
    # 在你的 mailer 類別
    default from: 'Your Name <info@your_site.com>'
    ```

* 確定測試環境的 email 發送方法設置為 `test` ：

    ```Ruby
    # config/environments/test.rb

    config.action_mailer.delivery_method = :test
    ```

* 開發與生產環境的發送方法應為 `smtp` ：

    ```Ruby
    # config/environments/development.rb, config/environments/production.rb

    config.action_mailer.delivery_method = :smtp
    ```

* 當發送 HTML email 時，所有樣式應為行內樣式，由於某些用戶有關於外部樣式的問題。某種程度上這使得更難管理及造成程式碼重用。有兩個相似的 gem 可以轉換樣式，以及將它們放在對應的html 標籤裡： [premailer-rails3](https://github.com/fphilipe/premailer-rails3) 和 [roadie](https:// github.com/Mange/roadie)。

* 應避免頁面產生響應時寄送 email。若多個 email 寄送時，造成了頁面載入延遲，以及請求可能逾時。使用 [delayed_job](https://github.com/tobi/dela​​yed_job) gem 的幫助來克服在背景處理寄送 email 的問題。

## Bundler

* 把只給開發環境或測試環境的 gem 適當地分組放在 Gemfile 文件中。
* 在你的專案中只使用公認的 gem。如果你考慮引入某些鮮為人所知的 gem ，你應該先仔細複查一下它的原始碼。
* 關於多個開發者使用不同操作系統的專案，操作系統相關的 gem 預設會產生一個經常變動的 `Gemfile.lock` 。在 Gemfile 文件裡，所有與 OS X 相關的 gem 放在 `darwin` 群組，而所有 Linux 相關的 gem 放在 `linux` 群組：

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

    要在對的環境獲得合適的 gem，添加以下程式碼至 `config/application.rb` ：

    ```Ruby
    platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
    Bundler.require(platform)
    ```

* 不要把 `Gemfile.lock` 文件從版本控制裡移除。這不是隨機產生的文件- 它確保你所有的成員執行`bundle install` 時，獲得相同版本的gem 。

## 無價的 Gems

一個最重要的編程理念是 "不要重造輪子！" 。若你遇到一個特定問題，你應該要在你開始前，看一下是否有存在的解決方案。下面是一些在很多 Rails 專案中 "無價的" gem 列表（全部相容 Rails 3.1）：

* [active_admin](https://github.com/gregbell/active_admin) - 有了 ActiveAdmin，創建 Rails 應用的管理介面就像兒戲。你會有一個很好的儀錶盤，圖形化 CRUD 介面以及更多東西。非常靈活且可客製化。
* [capybara](https://github.com/jnicklas/capybara) - Capybara 旨在簡化整合測試 Rack 應用的過程，像是 Rails、Sinatra 或 Merb。 Capybara 模擬了真實用戶使用 web 應用的互動。它與你測試在運行的驅動無關，並原生搭載 Rack::Test 及 Selenium 支持。透過外部 gem 支持 HtmlUnit、WebKit 及 env.js 。與 RSpec & Cucumber 一起使用時工作良好。
* [carrierwave](https://github.com/jnicklas/carrierwave) - Rails 最後一個文件上傳解決方案。支持上傳檔案（及很多其它的酷玩意的）的本機儲存與雲端儲存。圖片後處理與 ImageMagick 整合得非常好。
* [client_side_validations](https://github.com/bcardarella/client_side_validations) -
  一個美妙的 gem，替你從現有的服務器端模型驗證自動產生 Javascript 用戶端驗證。高度推薦！
* [compass-rails](https://github.com/chriseppstein/compass) - 一個優秀的 gem，添加了某些 css 框架的支持。包括了 sass mixin 的蒐集，讓你減少 css 文件的程式碼並幫你解決瀏覽器相容問題。
* [cucumber-rails](https://github.com/cucumber/cucumber-rails) - Cucumber 是一個由 Ruby 所寫，開發功能測試的頂級工具。 cucumber-rails 提供了 Cucumber 的 Rails 整合。
* [devise](https://github.com/plataformatec/devise) - Devise 是 Rails 應用的一個完整解決方案。多數情況偏好使用 devise 來開始你的客制驗證方案。
* [fabrication](http://fabricationgem.org/) - 一個很好的假資料產生器（編輯者的選擇）。
* [factory_girl](https://github.com/thoughtbot/factory_girl) - 另一個 Fabrication 的選擇。一個成熟的假資料產生器。 Fabrication 的精神領袖先驅。
* [faker](http://faker.rubyforge.org/) - 實用的 gem 來產生仿造的資料（名字、地址，等等）。
* [feedzirra](https://github.com/pauldix/feedzirra) - 非常快速及靈活的 RSS 或 Atom 種子解析器。
* [friendly_id](https://github.com/norman/friendly_id) - 透過使用某些具描述性的模型屬性，而不是使用 id，允許你創建人類可讀的網址。
* [guard](https://github.com/guard/guard) - 極佳的 gem 監控文件變化及任務的調用。搭載了很多實用的擴充。遠優於 autotest 與 watchr。
* [haml-rails](https://github.com/indirect/haml-rails) - haml-rails 提供了 Haml 的 Rails 整合。
* [haml](http://haml-lang.com) - Haml 是一個簡潔的模型語言，被很多人認為（包括我）遠優於Erb。
* [kaminari](https://github.com/amatsuda/kaminari) - 很棒的分頁解決方案。
* [machinist](https://github.com/notahat/machinist) - 假資料不好玩，Machinist 才好玩。
* [rspec-rails](https://github.com/rspec/rspec-rails) - RSpec 是 Test::MiniTest 的取代者。我不高度推薦 RSpec。 rspec-rails 提供了 RSpec 的 Rails 整合。
* [simple_form](https://github.com/plataformatec/simple_form) - 一旦用過 simple_form（或 formatastic），你就不想聽到關於 Rails 預設的表單。它是一個創造表單很棒的 DSL。
* [simplecov-rcov](https://github.com/fguillen/simplecov-rcov) - 為了 SimpleCov 打造的 RCov formatter。若你想使用 SimpleCov 搭配 Hudson 持續整合服務器，很有用。
* [simplecov](https://github.com/colszowka/simplecov) - 程式碼覆蓋率工具。不像 RCov，完全相容 Ruby 1.9。產生精美的報告。必須用！
* [slim](http://slim-lang.com) - Slim 是一個簡潔的模版語言，被視為是遠遠優於 HAML(Erb 就更不用說了)的語言。唯一會阻止我大規模地使用它的是，主流 IDE 及編輯器對它的支持不好。但它的效能是非凡的。
* [spork](https://github.com/timcharper/spork) - 一個給測試框架（RSpec 或現今 Cucumber）用的 DRb 服務器，每次運行前確保分支出一個乾淨的測試狀態。簡單的說，預載很多測試環境的結果是大幅降低你的測試啟動時間，絕對必須用！
* [sunspot](https://github.com/sunspot/sunspot) - 基於 SOLR 的全文檢索引擎。

這不是完整的清單，以及其它的 gem 也可以在之後加進來。以上清單上的所有 gems 皆經測試，處於活躍開發階段，有社群以及程式碼的品質很高。

## 缺陷的 Gems

這是一個有問題的或被別的 gem 取代的 gem 清單。你應該在你的專案裡避免使用它們。

* [rmagick](http://rmagick.rubyforge.org/) - 這個 gem 因大量消耗記憶體而聲名狼藉。使用 [minimagick](https://github.com/probablycorey/mini_magick) 來取代。
* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - 自動測試的老舊解決方案。遠不如 guard 及 [watchr](https://github.com/mynyml/watchr)。
* [rcov](https://github.com/relevance/rcov) - 程式碼覆蓋率工具，不相容 Ruby 1.9。使用 [SimpleCov](https://github.com/colszowka/simplecov) 來取代。
* [therubyracer](https://github.com/cowboyd/therubyracer) - 極度不鼓勵在生產模式使用這個gem，它消耗大量的記憶體。我會推薦使用 [Mustang](https://github.com/nu7hatch/mustang) 來取代。

這仍是一個完善中的清單。請告訴我受人歡迎但有缺陷的 gems 。

## 管理進程

* 若你的專案依賴各種外部的進程使用 [foreman](https://github.com/ddollar/foreman) 來管理它們。

# 測試 Rails 應用

也許 BDD 方法是實作一個新功能最好的方法。你從開始寫一些高階的測試（通常使用 Cucumber），然後使用這些測試來驅使你實作功能。一開始你給功能的視圖寫測試，並使用這些測試來創建相關的視圖。之後，你創建丟給視圖資料的控制器測試來實現控制器。最後你實作模型的測試以及模型自身。

## Cucumber

* 用 `@wip` （工作進行中）標籤標記你未完成的場景。這些場景不納入考慮，且不標記為測試失敗。當完成一個未完成場景且功能測試通過時，為了把此場景加至測試套件裡，應該移除 `@wip` 標籤。
* 配置你的預設配置文件，排除掉標記為 `@javascript` 的場景。它們使用瀏覽器來測試，推薦停用它們來增加一般場景的執行速度。
* 替標記著 `@javascript` 的場景配置另一個配置文件。
  * 配置文件可在 `cucumber.yml` 文件裡配置。

        ```Ruby
        # 配置文件的定義：
        profile_name: --tags @tag_name
        ```

  * 帶指令運行一個配置文件：

        ```
        cucumber -p profile_name
        ```

* 若使用 [fabrication](http://fabricationgem.org/) 來替換假資料(fixtures)，使用預定義的 [fabrication steps](http://fabricationgem.org/#!cucumber-steps)。
* 不要使用舊版的 `web_steps.rb` 步驟定義！[最新版 Cucumber 已移除 web steps](http://aslakhellesoy.com/post/11055981222/the-training-wheels-came-off) ，使用它們導致冗贅的場景，而且它並沒有正確地反映出應用的領域。
* 當檢查一元素的可視文字時，檢查元素的文字而不是檢查 id。這樣可以查出 i18n 的問題。
* 給同種類物件創建不同的功能特色：

    ```Ruby
    # 差
    Feature: Articles
    # ... 功能實作 ...

    # 好
    Feature: Article Editing
    # ... 功能實作 ...

    Feature: Article Publishing
    # ... 功能實作 ...

    Feature: Article Search
    # ... 功能實作 ...

    ```

* 每一個功能有三個主要成分：
  * Title
  * Narrative - 簡短說明這個特色關於什麼。
  * Acceptance criteria - 每個由獨立步驟組成的一套場景。
* 最常見的格式稱為 Connextra 格式。

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

這是最常見但不是要求的格式，敘述可以是依賴功能複雜度的任何文字。

* 自由地使用場景概述使你的場景備作它用(keep your scenarios DRY)。

    ```Ruby
    Scenario Outline: User cannot register with invalid e-mail
      When I try to register with an email "<email>"
      Then I should see the error message "<error>"

    Examples:
      |email |error |
      | |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* 場景的步驟放在 `step_definitions` 目錄下的 `.rb` 文件。步驟文件命名慣例為 `[description]_steps.rb`。步驟根據不同的標準放在不同的文件裡。每一個功能可能有一個步驟文件(`home_page_steps.rb`)
。也可能給每個特定物件的功能，建一個步驟文件(`articles_steps.rb`)。
* 使用多行步驟參數來避免重複

    ```Ruby
    場景: User profile
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

* 使用複合​​步驟使場景備作它用(Keep your scenarios DRY)

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
* 總是使用 Capybara 否定匹配來取代正面情況搭配 should_not，它們會在給定的超時時重試匹配，允許你測試 ajax 動作。見 [Capybara 的讀我文件](https://github.com/jnicklas/capybara)獲得更多說明。

## RSpec

* 一個例子僅用一個期望值。

    ```Ruby
    # 差
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

    # 好
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

* 大量使用 `descibe` 及 `context` 。
* 如下地替 `describe` 區塊命名：
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
    describe Article
      describe '#summary'
        #...
      end

      describe '.latest'
        #...
      end
    end
    ```

* 使用 [fabricators](http://fabricationgem.org/) 來創建測試物件。
  
* 大量使用 mocks 與 stubs。

    ```Ruby
    # mocking 一個模型
    article = mock_model(Article)

    # stubbing 一個方法
    Article.stub(:find).with(article.id).and_return(article)
    ```

* 當 mocking 一個模型時，使用 `as_null_object` 方法。它告訴輸出僅監聽我們預期的訊息，並忽略其它的訊息。

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* 使用 `let` 區塊而不是 `before(:all)` 區塊替 spec 例子創建資料。 `let` 區塊會被懶惰求值。

    ```Ruby
    # 使用這個：
    let(:article) { Fabricate(:article) }

    # ... 而不是這個：
    before(:each) { @article = Fabricate(:article) }
    ```

* 當可能時，使用 `subject`。

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* 如果可能的話，使用 `specify`。它是 `it` 的同義詞，但在沒 docstring 的情況下可讀性​​更高。

    ```Ruby
    # 差
    describe Article do
      before { @article = Fabricate(:article) }

      it 'is not published on creation' do
        @article.should_not be_published
      end
    end

    # 好
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* 當可能時，使用 `its` 。

    ```Ruby
    # 差
    describe Article do
      subject { Fabricate(:article) }

      it 'has the current date as creation date' do
        subject.creation_date.should == Date.today
      end
    end

    # 好
    describe Article do
      subject { Fabricate(:article) }
      its(:creation_date) { should == Date.today }
    end
    ```

### 視圖

* 視圖測試的目錄結構要與 `app/views` 之中的相符。舉例來說，在 `app/views/users` 視圖被放在 `spec/views/users`。
* 視圖測試的命名慣例是添加`_spec.rb` 至視圖名字之後，舉例來說，視圖 `_form.html.haml` 有一個對應的測試叫做 `_form.html.haml_spec.rb`。
* 每個視圖測試文件都需要`spec_helper.rb`。
* 外部描述區塊使用不含 `app/views` 部分的視圖路徑。 `render` 方法沒有傳入參數時，是這麼使用的。

    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* 永遠在視圖測試來 mock 模型。視圖的目的只是顯示訊息。
* `assign` 方法提供由控制器提供視圖使用的實體變數(instance variable)。

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

* 偏好 capybara 否定情況選擇器，勝於搭配正面情況的 should_not 。

    ```Ruby
    # 差
    page.should_not have_selector('input', type: 'submit')
    page.should_not have_xpath('tr')

    # 好
    page.should have_no_selector('input', type: 'submit')
    page.should have_no_xpath('tr')
    ```

* 當一個視圖使用 helper 方法時，這些方法需要被 stubbed。 Stubbing 這些 helper 方法是在 `template` 完成的：

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
    describe 'articles/show.html.html' do
      it 'displays the formatted date of article publishing'
        article = mock_model(Article, published_at: Date.new(2012, 01, 01))
        assign(:article, article)

        template.stub(:formatted_date).with(article.published_at).and_return '01.01.2012'

        render
        rendered.should have_content('Published at: 01.01.2012')
      end
    end
    ```

* 在 `spec/helpers` 目錄的 helper specs 是與視圖 specs 分開的。

### 控制器

* Mock 模型及 stub 他們的方法。測試控制器時不應依賴建模。
* 僅測試控制器需負責的行為：
  * 執行特定的方法
  * 從動作返回的資料 - assigns, 等等。
  * 從動作返回的結果 - template render, redirect, 等等。

        ```Ruby
        # 常用的控制器 spec 範例
        # spec/controllers/articles_controller_spec.rb
        # 我們只對控制器應執行的動作感興趣
        # 所以我們 mock 模型及 stub 它的方法
        # 並且專注在控制器該做的事情上

        describe ArticlesController do
          # 模型將會在測試中被所有控制器的方法所使用
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

* 當控制器根據不同參數有不同行為時，使用 context。

    ```Ruby
    # 一個在控制器中使用 context 的典型例子是，物件正確保存時，使用創建，保存失敗時更新。

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

### 模型

* 不要在自己的測試裡 mock 模型。
* 使用捏造的東西來創建真的物件
* Mock 別的模型或子物件是可接受的。
* 在測試裡建立所有例子的模型來避免重複。

    ```Ruby
    describe Article
      let(:article) { Fabricate(:article) }
    end
    ```

* 加入一個例子確保捏造的模型是可行的。

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* 當測試驗證時，使用 `have(x).errors_on` 來指定要被驗證的屬性。使用 `be_valid` 不保證問題在目的的屬性。

    ```Ruby
    # 差
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # 偏好
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

* 替每個有驗證的屬性加另一個 `describe`。

    ```Ruby
    describe Article
      describe '#title'
        it 'is required' do
          article.title = nil
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

* 當測試模型屬性的獨立性時，把其它物件命名為 `another_object`。

    ```Ruby
    describe Article
      describe '#title'
        it 'is unique' do
          another_article = Fabricate.build(:article, title: article.title)
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

### Mailers

* 在 Mailer 測試的模型應該要被 mock。 Mailer 不應依賴建模。
* Mailer 的測試應該確認如下：
  * 這個 subject 是正確的
  * 這個 receiver e-mail 是正確的
  * 這個 e-mail 寄送至對的郵件地址
  * 這個 e-mail 包含了需要的訊息

     ```Ruby
     describe SubscriberMailer
       let(:subscriber) { mock_model(Subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email'
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

* 我們如何測試上傳器是否正確地調整大小。這裡是一個 [carrierwave](https://github.com/jnicklas/carrierwave) 圖片上傳器的範例 spec：

    ```Ruby

    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # 在執行例子前啟用圖片處理
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # 創建一個新的 uploader。模型被模仿為不依賴建模時的上傳及調整圖片。
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # 執行完例子時停用圖片處理
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # 測試圖片是否不比給定的維度長
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # 測試圖片是否有確切的維度
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end

    ```

# 延伸閱讀

有幾個絕妙講述 Rails 風格的資源，若有閒暇時應當考慮延伸閱讀：

* [The Rails 3 Way](http://tr3w.com/)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)

# 貢獻

在本指南所寫的每個東西都不是定案。這只是我渴望想與同樣對 Rails 編碼風格有興趣的大家一起工作，以致於最終我們可以替整個 Ruby 社群創造一個有益的資源。

歡迎開票或發送一個帶有改進的更新請求。在此提前感謝你的幫助！

# 口耳相傳

一份社群驅動的風格指南，對一個社群來說，只是讓人知道有這個社群。推特轉發這份指南，分享給你的朋友或同事。我們得到的每個註解、建議或意見都可以讓這份指南變得更好一點。而我們想要擁有的是最好的指南，不是嗎？