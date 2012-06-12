# 序幕

> 风格是从伟大事物中分离出的美好事物。 <br/>
> -- Bozhidar Batsov

这份指南目的于演示一整套 Rails 3 开发的风格惯例及最佳实践。这是一份与由现存社群所驱动的[Ruby 编码风格指南](https://github.com/bbatsov/ruby-style-guide)互补的指南。

而本指南中[测试 Rails 应用](#testing)小节摆在[开发 Rails 应用](#developing)之后，因为我相信[行为驱动开发](http://en.wikipedia.org/wiki/Behavior_Driven_Development)
(BDD) 是最佳的软体开发之道。铭记在心吧。

Rails 是一个坚持己见的框架，而这也是一份坚持己见的指南。在我的心里，我坚信 [RSpec](https://www.relishapp.com/rspec) 优于 Test::Unit，[Sass](http://sass-lang.com/) 优于 CSS 以及
[Haml](http://haml-lang.com/)，([Slim](http://slim-lang.com/)) 优于 Erb。所以不要期望在这里找到 Test::Unit, CSS 及 Erb 的忠告。

某些忠告仅适用于 Rails 3.1+ 以上版本。

你可以使用 [Transmuter](https://github.com/TechnoGate/transmuter) 来产生本指南的一份 PDF 或 HTML 复本。

# 目录

* [开发 Rails 应用程序](#开发-rails-应用程序)
    * [配置](#配置)
    * [路由](#路由)
    * [控制器](#控制器)
    * [模型](#模型)
    * [迁移](#迁移)
    * [视图](#视图)
    * [Assets](#assets)
    * [Mailers](#mailers)
    * [Bundler](#bundler)
    * [无价的 Gems](#无价的-gems)
    * [缺陷的 Gems](#缺陷的-gems)
    * [管理进程](#管理进程)
* [测试 Rails 应用](#测试-rails-应用)
    * [Cucumber](#cucumber)
    * [RSpec](#rspec)

# 开发 Rails 应用程序

## 配置

* 把惯用的初始化代码放在 `config/initializers`。 在 initializers 内的代码于应用启动时执行。
* 每一个 gem 相关的初始化代码应当使用同样的名称，放在不同的文件里，如： `carrierwave.rb`, `active_admin.rb`, 等等。
* 相应调整配置开发、测试及生产环境（在 `config/environments/` 下对应的文件）
  * 标记额外的资产给（如有任何）预编译：

        ```Ruby
        # config/environments/production.rb
        # 预编译额外的资产(application.js, application.css, 以及所有已经被加入的非 JS 或 CSS 的文件)
        config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
        ```

* 创立一个与生产环境(production enviroment)相似的额外 `staging` 环境。

## 路由

* 当你需要加入一个或多个动作至一个 RESTful 资源时（你真的需要吗？），使用 `member` and `collection` 路由。

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

* 若你需要定义多个 `member/collection` 路由时，使用替代的区块语法(block syntax)。

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

* 使用嵌套路由(nested routes)来更佳地表达与 ActiveRecord 模型的关系。

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

* 使用命名空间路由来群组相关的行为。

    ```Ruby
    namespace :admin do
      # Directs /admin/products/* to Admin::ProductsController
      # (app/controllers/admin/products_controller.rb)
      resources :products
    end
    ```

* 不要在控制器里使用留给后人般的疯狂路由(legacy wild controller route)。这种路由会让每个控制器的动作透过 GET 请求存取。

    ```Ruby
    # 非常差
    match ':controller(/:action(/:id(.:format)))'
    ```

## 控制器

* 让你的控制器保持苗条 ― 它们应该只替视图层取出数据且不包含任何业务逻辑（所有业务逻辑应当放在模型里）。
* 每个控制器的行动应当（理想上）只调用一个除了初始的 find 或 new 方法。
* 控制器与视图之间共享不超过两个实例变量(instance variable)。

## 模型

* 自由地引入不是 ActiveRecord 的类别吧。
* 替模型命名有意义（但简短）且不带缩写的名字。 
* 如果你需要支援 ActiveRecord 像是验证行为的模型对象，使用 [ActiveAttr](https://github.com/cgriego/active_attr) gem。

    ```Ruby
    class Message
      include ActiveAttr::Model

      attribute :name
      attribute :email
      attribute :content
      attribute :priority

      attr_accessible :name, :email, :content

      validates_presence_of :name
      validates_format_of :email, :with => /^[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}$/i
      validates_length_of :content, :maximum => 500
    end
    ```

    更完整的示例，参考 [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr)。

### ActiveRecord

* 避免改动缺省的 ActiveRecord（表的名字、主键，等等），除非你有一个非常好的理由（像是不受你控制的数据库）。
* 把宏风格的方法放在类别定义的前面（`has_many`, `validates`, 等等）。
* 偏好 `has_many :through` 胜于 `has_and_belongs_to_many`。 使用 `has_many :through` 允许在 join 模型有附加的属性及验证

    ```Ruby
    # 使用 has_and_belongs_to_many
    class User < ActiveRecord::Base
      has_and_belongs_to_many :groups
    end

    class Group < ActiveRecord::Base
      has_and_belongs_to_many :users
    end

    # 偏好方式 - using has_many :through
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
* 当一个惯用的验证使用超过一次或验证是某个正则表达映射时，创建一个惯用的 validator 文件。

    ```Ruby
    # 差
    class Person
      validates :email, format: { with: /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i }
    end

    # 好
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
      end
    end

    class Person
      validates :email, email: true
    end
    ```

* 所有惯用的验证器应放在一个共享的 gem 。
* 自由地使用命名的作用域(scope)。
* 当一个由 lambda 及参数定义的作用域变得过于复杂时，更好的方式是建一个作为同样用途的类别方法，并返回 `ActiveRecord::Relation` 对象。
* 注意 `update_attribute` 方法的行为。它不运行模型验证（不同于 `update_attributes` ）并且可能把模型状态给搞砸。
* 使用用户友好的网址。在网址显示具描述性的模型属性，而不只是 `id` 。
有不止一种方法可以达成：
  * 覆写模型的 `to_param` 方法。这是 Rails 用来给对象建构网址的方法。缺省的实作会以字串形式返回该 `id` 的记录。它可被另一个具人类可读的属性覆写。

        ```Ruby
        class Person
          def to_param
            "#{id} #{name}".parameterize
          end
        end
        ```
        为了要转换成对网址友好 (URL-friendly)的数值，字串应当调用 `parameterize` 。 对象的 `id` 要放在开头，以便给 ActiveRecord 的 `find` 方法查找。

  * 使用此 `friendly_id` gem。它允许藉由某些具描述性的模型属性，而不是用 `id` 来创建人类可读的网址。

        ```Ruby
        class Person
          extend FriendlyId
          friendly_id :name, use: :slugged
        end
        ```

        查看 [gem 文档](https://github.com/norman/friendly_id)获得更多关于使用的信息。

### ActiveResource

* 当 HTTP 响应是一个与存在的格式不同的格式时（XML 和 JSON），需要某些额外的格式解析，创一个你惯用的格式，并在类别中使用它。惯用的格式应当实作下列方法：`extension`, `mime_type`,
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
              # 数据以新格式编码并返回
            end

            def decode(csv)
              # 数据以新格式解码并返回
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

* 若 HTTP 请求应当不扩展发送时，覆写 `ActiveResource::Base` 的 `element_path` 及 `collection_path` 方法，并移除扩展的部份。

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

    如有任何改动网址的需求时，这些方法也可以被覆写。

## 迁移

* 把 `schema.rb` 保存在版本管控之下。
* 使用 `rake db:scheme:load` 取代 `rake db:migrate` 来初始化空的数据库。
* 使用 `rake db:test:prepare` 来更新测试数据库的 schema。
* 避免在表里设置缺省数据。使用模型层来取代。

    ```Ruby
    def amount
      self[:amount] or 0
    end
    ```

    然而 `self[:attr_name]` 的使用被视为相当常见的，你也可以考虑使用更罗嗦的（争议地可读性更高的） `read_attribute` 来取代：

    ```Ruby
    def amount
      read_attribute(:amount) or 0
    end
    ```

* 当编写建设性的迁移时（加入表或栏位），使用 Rails 3.1 的新方式来迁移 - 使用 `change` 方法取代 `up` 与 `down` 方法。


    ```Ruby
    # 过去的方式
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

## 视图

* 不要直接从视图调用模型层。
* 不要在视图构造复杂的格式，把它们输出到视图 helper 的一个方法或是模型。
* 使用 partial 模版与布局来减少重复的代码。
* 加入 [client side validation](https://github.com/bcardarella/client_side_validations) 至惯用的 validators。 要做的步骤有：
  * 声明一个由 `ClientSideValidations::Middleware::Base` 而来的自定 validator

        ```Ruby
        module ClientSideValidations::Middleware
          class Email < Base
            def response
              if request.params[:email] =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
                self.status = 200
              else
                self.status = 404
              end
              super
            end
          end
        end
        ```

  * 建立一个新文件
    `public/javascripts/rails.validations.custom.js.coffee` 并在你的 `application.js.coffee` 文件加入一个它的参照：

        ```Ruby
        # app/assets/javascripts/application.js.coffee
        #= require rails.validations.custom
        ```

  * 添加你的用户端 validator：

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

## 国际化

* 视图、模型与控制器里不应使用语言相关设置与字串。这些文字应搬到在 `config/locales` 下的语言文件里。
* 当 ActiveRecord 模型的标签需要被翻译时，使用`activerecord` 作用域:

    ```
    en:
      activerecord:
        models:
          user: Member
        attributes:
          user:
            name: "Full name"
    ```

    然后 `User.model_name.human` 会返回 "Member" ，而 `User.human_attribute_name("name")` 会返回 "Full name"。这些属性的翻译会被视图作为标签使用。

* 把在视图使用的文字与 ActiveRecord 的属性翻译分开。 把给模型使用的语言文件放在名为 `models` 的文件夹，给视图使用的文字放在名为 `views` 的文件夹。
  * 当使用额外目录的语言文件组织完成时，为了要载入这些目录，要在 `application.rb` 文件里描述这些目录。

        ```Ruby
        # config/application.rb
        config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
        ```

* 把共享的本土化选项，像是日期或货币格式，放在 `locales` 的根目录下。
* 使用精简形式的 I18n 方法： `I18n.t` 来取代 `I18n.translate` 以及使用 `I18n.l` 取代 `I18n.localize`。
* 使用 "懒惰" 查询视图中使用的文字。假设我们有以下结构：

    ```
    en:
      users:
        show:
          title: "User details page"
    ```

    `users.show.title` 的数值能这样被 `app/views/users/show.html.haml` 查询：

    ```Ruby
    = t '.title'
    ```

* 在控制器与模型使用点分隔的键，来取代指定 `:scope` 选项。点分隔的调用更容易阅读及追踪层级。

    ```Ruby
    # 这样子调用
    I18n.t 'activerecord.errors.messages.record_invalid'

    # 而不是这样
    I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]
    ```

* 关于 Rails i18n 更详细的信息可以在这里找到 [Rails Guides](http://guides.rubyonrails.org/i18n.html)。

## Assets

利用这个 [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) 来管理应用的结构。

* 保留 `app/assets` 给自定的样式表, javascripts, 或图片。
* 第三方代码如： [jQuery](http://jquery.com/) 或 [bootstrap](http://twitter.github.com/bootstrap/) 应放置在 `vendor/assets`。
* 当可能的时候，使用 gem 化的 assets 版本。(如： [jquery-rails](https://github.com/rails/jquery-rails))。

## Mailers

* 把 mails 命名为 `SomethingMailer`。 没有 Mailer 字根的话，不能立即显现哪个是一个 Mailer，以及哪个视图与它有关。
* 提供 HTML 与纯文本视图模版。
* 在你的开发环境启用信件失败发送错误。这些错误缺省是被停用的。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* 在开发模式使用 `smtp.gmail.com` 设置 SMTP 服务器（当然了，除非你自己有本地 SMTP 服务器）。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      address: 'smtp.gmail.com',
      # 更多设置
    }
    ```

* 提供缺省的配置给主机名。

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {host: "#{local_ip}:3000"}


    # config/environments/production.rb
    config.action_mailer.default_url_options = {host: 'your_site.com'}

    # 在你的 mailer 类
    default_url_options[:host] = 'your_site.com'
    ```

* 如果你需要在你的网站使用一个 email 链结，总是使用 `_url` 方法，而不是 `_path` 方法。 `_url` 方法包含了主机名，而 `_path` 方法没有。

    ```Ruby
    # 错误
    You can always find more info about this course
    = link_to 'here', url_for(course_path(@course))

    # 正确
    You can always find more info about this course
    = link_to 'here', url_for(course_url(@course))
    ```

* 正确地显示寄与收件人地址的格式。使用下列格式：

    ```Ruby
    # 在你的 mailer 类别
    default from: 'Your Name <info@your_site.com>'
    ```

* 确定测试环境的 email 发送方法设置为 `test` ：

    ```Ruby
    # config/environments/test.rb

    config.action_mailer.delivery_method = :test
    ```

* 开发与生产环境的发送方法应为 `smtp` ：

    ```Ruby
    # config/environments/development.rb, config/environments/production.rb

    config.action_mailer.delivery_method = :smtp
    ```

* 当发送 HTML email 时，所有样式应为行内样式，由于某些用户有关于外部样式的问题。某种程度上这使得更难管理及造成代码重用。有两个相似的 gem 可以转换样式，以及将它们放在对应的 html 标签里： [premailer-rails3](https://github.com/fphilipe/premailer-rails3) 和 [roadie](https://github.com/Mange/roadie)。

* 应避免页面产生响应时寄送 email。若多个 email 寄送时，造成了页面载入延迟，以及请求可能逾时。使用 [delayed_job](https://github.com/tobi/delayed_job) gem 的帮助来克服在背景处理寄送 email 的问题。

## Bundler

* 把只给开发环境或测试环境的 gem 适当地分组放在 Gemfile 文件中。
* 在你的项目中只使用公认的 gem。 如果你考虑引入某些鲜为人所知的 gem ，你应该先仔细复查一下它的源代码。
* 关于多个开发者使用不同操作系统的项目，操作系统相关的 gem 缺省会产生一个经常变动的 `Gemfile.lock` 。 在 Gemfile 文件里，所有与 OS X 相关的 gem 放在 `darwin` 群组，而所有 Linux 相关的 gem 放在 `linux` 群组：

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

    要在对的环境获得合适的 gem，添加以下代码至 `config/application.rb` ：

    ```Ruby
    platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
    Bundler.require(platform)
    ```

* 不要把 `Gemfile.lock` 文件从版本控制里移除。这不是随机产生的文件 - 它确保你所有的组员执行 `bundle install` 时，获得相同版本的 gem 。

## 无价的 Gems

一个最重要的编程理念是 "不要重造轮子！" 。若你遇到一个特定问题，你应该要在你开始前，看一下是否有存在的解决方案。下面是一些在很多 Rails 项目中 "无价的" gem 列表（全部兼容 Rails 3.1）：

* [active_admin](https://github.com/gregbell/active_admin) - 有了 ActiveAdmin，创建 Rails 应用的管理介面就像儿戏。你会有一个很好的仪表盘，图形化 CRUD 介面以及更多东西。非常灵活且可客制化。
* [capybara](https://github.com/jnicklas/capybara) - Capybara 旨在简化整合测试 Rack 应用的过程，像是 Rails、Sinatra 或 Merb。Capybara 模拟了真实用户使用 web 应用的互动。 它与你测试在运行的驱动无关，并原生搭载 Rack::Test 及 Selenium 支持。透过外部 gem 支持 HtmlUnit、WebKit 及 env.js 。与 RSpec & Cucumber 一起使用时工作良好。
* [carrierwave](https://github.com/jnicklas/carrierwave) - Rails 最后一个文件上传解决方案。支持上传档案（及很多其它的酷玩意儿的）的本地储存与云储存。图片后处理与 ImageMagick 整合得非常好。
* [client_side_validations](https://github.com/bcardarella/client_side_validations) -
  一个美妙的 gem，替你从现有的服务器端模型验证自动产生 Javascript 用户端验证。高度推荐！
* [compass-rails](https://github.com/chriseppstein/compass) - 一个优秀的 gem，添加了某些 css 框架的支持。包括了 sass mixin 的蒐集，让你减少 css 文件的代码并帮你解决浏览器兼容问题。
* [cucumber-rails](https://github.com/cucumber/cucumber-rails) - Cucumber 是一个由 Ruby 所写，开发功能测试的顶级工具。 cucumber-rails 提供了 Cucumber 的 Rails 整合。
* [devise](https://github.com/plataformatec/devise) - Devise 是 Rails 应用的一个完整解决方案。多数情况偏好使用 devise 来开始你的客制验证方案。
* [fabrication](http://fabricationgem.org/) - 一个很好的假数据产生器（编辑者的选择）。
* [factory_girl](https://github.com/thoughtbot/factory_girl) - 另一个 Fabrication 的选择。一个成熟的假数据产生器。 Fabrication 的精神领袖先驱。
* [faker](http://faker.rubyforge.org/) - 实用的 gem 来产生仿造的数据（名字、地址，等等）。
* [feedzirra](https://github.com/pauldix/feedzirra) - 非常快速及灵活的 RSS 或 Atom 种子解析器。
* [friendly_id](https://github.com/norman/friendly_id) - 透过使用某些具描述性的模型属性，而不是使用 id，允许你创建人类可读的网址。
* [guard](https://github.com/guard/guard) - 极佳的 gem 监控文件变化及任务的调用。搭载了很多实用的扩充。远优于 autotest 与 watchr。
* [haml-rails](https://github.com/indirect/haml-rails) - haml-rails 提供了 Haml 的 Rails 整合。
* [haml](http://haml-lang.com) - Haml 是一个简洁的模型语言，被很多人认为（包括我）远优于 Erb。
* [kaminari](https://github.com/amatsuda/kaminari) - 很棒的分页解决方案。
* [machinist](https://github.com/notahat/machinist) - 假数据不好玩，Machinist 才好玩。
* [rspec-rails](https://github.com/rspec/rspec-rails) - RSpec 是 Test::MiniTest 的取代者。我不高度推荐 RSpec。 rspec-rails 提供了 RSpec 的 Rails 整合。
* [simple_form](https://github.com/plataformatec/simple_form) - 一旦用过 simple_form（或 formatastic），你就不想听到关于 Rails 缺省的表单。它是一个创造表单很棒的DSL。
* [simplecov-rcov](https://github.com/fguillen/simplecov-rcov) - 为了 SimpleCov 打造的 RCov formatter。若你想使用 SimpleCov 搭配 Hudson 持续整合服务器，很有用。
* [simplecov](https://github.com/colszowka/simplecov) - 代码覆盖率工具。不像 RCov，完全兼容 Ruby 1.9。产生精美的报告。必须用！
* [slim](http://slim-lang.com) - Slim 是一个简洁的模版语言，被视为是远远优于 HAML(Erb 就更不用说了)的语言。唯一会阻止我大规模地使用它的是，主流 IDE 及编辑器对它的支持不好。但它的效能是非凡的。
* [spork](https://github.com/timcharper/spork) - 一个给测试框架（RSpec 或 现今 Cucumber）用的 DRb 服务器，每次运行前确保分支出一个乾净的测试状态。 简单的说，预载很多测试环境的结果是大幅降低你的测试启动时间，绝对必须用！
* [sunspot](https://github.com/sunspot/sunspot) - 基于 SOLR 的全文检索引擎。

这不是完整的清单，以及其它的 gem 也可以在之后加进来。以上清单上的所有 gems 皆经测试，处于活跃开发阶段，有社群以及代码的质量很高。

## 缺陷的 Gems

这是一个有问题的或被别的 gem 取代的 gem 清单。你应该在你的项目里避免使用它们。

* [rmagick](http://rmagick.rubyforge.org/) - 这个 gem 因大量消耗内存而声名狼藉。使用 [minimagick](https://github.com/probablycorey/mini_magick) 来取代。
* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - 自动测试的老旧解决方案。远不如 guard 及 [watchr](https://github.com/mynyml/watchr)。
* [rcov](https://github.com/relevance/rcov) - 代码覆盖率工具，不兼容 Ruby 1.9。使用 [SimpleCov](https://github.com/colszowka/simplecov) 来取代。
* [therubyracer](https://github.com/cowboyd/therubyracer) - 极度不鼓励在生产模式使用这个 gem，它消耗大量的内存。我会推荐使用 [Mustang](https://github.com/nu7hatch/mustang) 来取代。

这仍是一个完善中的清单。请告诉我受人欢迎但有缺陷的 gems 。

## 管理进程

* 若你的项目依赖各种外部的进程使用 [foreman](https://github.com/ddollar/foreman) 来管理它们。

# 测试 Rails 应用

也许 BDD 方法是实作一个新功能最好的方法。你从开始写一些高阶的测试（通常使用 Cucumber），然后使用这些测试来驱使你实作功能。一开始你给功能的视图写测试，并使用这些测试来创建相关的视图。之后，你创建丢给视图数据的控制器测试来实现控制器。最后你实作模型的测试以及模型自身。

## Cucumber

* 用 `@wip` （工作进行中）标签标记你未完成的场景。这些场景不纳入考虑，且不标记为测试失败。当完成一个未完成场景且功能测试通过时，为了把此场景加至测试套件里，应该移除 `@wip` 标签。 
* 配置你的缺省配置文件，排除掉标记为 `@javascript` 的场景。它们使用浏览器来测试，推荐停用它们来增加一般场景的执行速度。
* 替标记著 `@javascript` 的场景配置另一个配置文件。
  * 配置文件可在 `cucumber.yml` 文件里配置。

        ```Ruby
        # 配置文件的定义：
        profile_name: --tags @tag_name
        ```

  * 带指令运行一个配置文件：

        ```
        cucumber -p profile_name
        ```

* 若使用 [fabrication](http://fabricationgem.org/) 来替换假数据 (fixtures)，使用预定义的 [fabrication steps](http://fabricationgem.org/#!cucumber-steps)。
* 不要使用旧版的 `web_steps.rb` 步骤定义！[最新版 Cucumber 已移除 web steps](http://aslakhellesoy.com/post/11055981222/the-training-wheels-came-off)，使用它们导致冗赘的场景，而且它并没有正确地反映出应用的领域。
* 当检查一元素的可视文字时，检查元素的文字而不是检查 id。这样可以查出 i18n 的问题。
* 给同种类对象创建不同的功能特色：

    ```Ruby
    # 差
    Feature: Articles
    # ... 功能实作 ...

    # 好
    Feature: Article Editing
    # ... 功能实作 ...

    Feature: Article Publishing
    # ... 功能实作 ...

    Feature: Article Search
    # ... 功能实作 ...

    ```

* 每一个功能有三个主要成分：
  * Title
  * Narrative - 简短说明这个特色关于什么。
  * Acceptance criteria - 每个由独立步骤组成的一套场景。
* 最常见的格式称为 Connextra 格式。

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

这是最常见但不是要求的格式，叙述可以是依赖功能复杂度的任何文字。

* 自由地使用场景概述使你的场景备作它用 (keep your scenarios DRY)。

    ```Ruby
    Scenario Outline: User cannot register with invalid e-mail
      When I try to register with an email "<email>"
      Then I should see the error message "<error>"

    Examples:
      |email         |error                 |
      |              |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* 场景的步骤放在 `step_definitions` 目录下的 `.rb` 文件。步骤文件命名惯例为 `[description]_steps.rb`。步骤根据不同的标准放在不同的文件里。每一个功能可能有一个步骤文件 (`home_page_steps.rb`)
。也可能给每个特定对象的功能，建一个步骤文件 (`articles_steps.rb`)。
* 使用多行步骤参数来避免重复

    ```Ruby
    场景: User profile
      Given I am logged in as a user "John Doe" with an e-mail "user@test.com"
      When I go to my profile
      Then I should see the following information:
        |First name|John         |
        |Last name |Doe          |
        |E-mail    |user@test.com|

    # 步骤:
    Then /^I should see the following information:$/ do |table|
      table.raw.each do |field, value|
        find_field(field).value.should =~ /#{value}/
      end
    end
    ```

* 使用复合步骤使场景备作它用 (Keep your scenarios DRY)

    ```Ruby
    # ...
    When I subscribe for news from the category "Technical News"
    # ...

    # 步骤:
    When /^I subscribe for news from the category "([^"]*)"$/ do |category|
      steps %Q{
        When I go to the news categories page
        And I select the category #{category}
        And I click the button "Subscribe for this category"
        And I confirm the subscription
      }
    end
    ```
* 总是使用 Capybara 否定匹配来取代正面情况搭配 should_not，它们会在给定的超时时重试匹配，允许你测试 ajax 动作。 见 [Capybara 的 读我文件](https://github.com/jnicklas/capybara)获得更多说明。

## RSpec

* 一个例子仅用一个期望值。

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
* 如下地替 `describe` 区块命名：
  * 非方法使用 "description" 
  * 实例方法使用井字号 "#method" 
  * 类别方法使用点 ".method"

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

* 使用 [fabricators](http://fabricationgem.org/) 来创建测试对象。
  
* 大量使用 mocks 与 stubs。

    ```Ruby
    # mocking 一个模型
    article = mock_model(Article)

    # stubbing 一个方法
    Article.stub(:find).with(article.id).and_return(article)
    ```

* 当 mocking 一个模型时，使用 `as_null_object` 方法。它告诉输出仅监听我们预期的讯息，并忽略其它的讯息。

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* 使用 `let` 区块而不是 `before(:each)` 区块替 spec 例子创建数据。`let` 区块会被懒惰求值。

    ```Ruby
    # 使用这个：
    let(:article) { Fabricate(:article) }

    # ... 而不是这个：
    before(:each) { @article = Fabricate(:article) }
    ```

* 当可能时，使用 `subject`。

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* 如果可能的话，使用 `specify`。它是 `it` 的同义词，但在没 docstring 的情况下可读性更高。

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

* 当可能时，使用 `its` 。

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

### 视图

* 视图测试的目录结构要与 `app/views` 之中的相符。 举例来说，在 `app/views/users` 视图被放在 `spec/views/users`。 
* 视图测试的命名惯例是添加 `_spec.rb` 至视图名字之后，举例来说，视图 `_form.html.haml` 有一个对应的测试叫做 `_form.html.haml_spec.rb`。
* 每个视图测试文件都需要 `spec_helper.rb`。
* 外部描述区块使用不含 `app/views` 部分的视图路径。 `render` 方法没有传入参数时，是这么使用的。

    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* 永远在视图测试来 mock 模型。视图的目的只是显示信息。
* `assign` 方法提供由控制器提供视图使用的实例变量(instance variable)。

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

* 偏好 capybara 否定情况选择器，胜于搭配正面情况的 should_not 。

    ```Ruby
    # 差
    page.should_not have_selector('input', type: 'submit')
    page.should_not have_xpath('tr')

    # 好
    page.should have_no_selector('input', type: 'submit')
    page.should have_no_xpath('tr')
    ```

* 当一个视图使用 helper 方法时，这些方法需要被 stubbed。Stubbing 这些 helper 方法是在 `template` 完成的：

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

* 在 `spec/helpers` 目录的 helper specs 是与视图 specs 分开的。

### 控制器

* Mock 模型及 stub 他们的方法。测试控制器时不应依赖建模。
* 仅测试控制器需负责的行为：
  * 执行特定的方法
  * 从动作返回的数据 - assigns, 等等。
  * 从动作返回的结果 - template render, redirect, 等等。

        ```Ruby
        # 常用的控制器 spec 示例
        # spec/controllers/articles_controller_spec.rb
        # 我们只对控制器应执行的动作感兴趣
        # 所以我们 mock 模型及 stub 它的方法
        # 并且专注在控制器该做的事情上

        describe ArticlesController do
          # 模型将会在测试中被所有控制器的方法所使用
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

* 当控制器根据不同参数有不同行为时，使用 context。

    ```Ruby
    # 一个在控制器中使用 context 的典型例子是，对象正确保存时，使用创建，保存失败时更新。

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

* 不要在自己的测试里 mock 模型。
* 使用捏造的东西来创建真的对象
* Mock 别的模型或子对象是可接受的。
* 在测试里建立所有例子的模型来避免重复。

    ```Ruby
    describe Article
      let(:article) { Fabricate(:article) }
    end
    ```

* 加入一个例子确保捏造的模型是可行的。

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* 当测试验证时，使用 `have(x).errors_on` 来指定要被验证的属性。使用 `be_valid` 不保证问题在目的的属性。

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

* 替每个有验证的属性加另一个 `describe`。

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

* 当测试模型属性的独立性时，把其它对象命名为 `another_object`。

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

* 在 Mailer 测试的模型应该要被 mock。 Mailer 不应依赖建模。
* Mailer 的测试应该确认如下：
  * 这个 subject 是正确的
  * 这个 receiver e-mail 是正确的
  * 这个 e-mail 寄送至对的邮件地址
  * 这个 e-mail 包含了需要的信息

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

* 我们如何测试上传器是否正确地调整大小。这里是一个 [carrierwave](https://github.com/jnicklas/carrierwave) 图片上传器的示例 spec：

    ```Ruby

    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # 在执行例子前启用图片处理
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # 创建一个新的 uploader。模型被模仿为不依赖建模时的上传及调整图片。
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # 执行完例子时停用图片处理
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # 测试图片是否不比给定的维度长
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # 测试图片是否有确切的维度
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end

    ```

# 延伸阅读

有几个绝妙讲述 Rails 风格的资源，若有闲暇时应当考虑延伸阅读：

* [The Rails 3 Way](http://tr3w.com/)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)

# 贡献

在本指南所写的每个东西都不是定案。这只是我渴望想与同样对 Rails 编码风格有兴趣的大家一起工作，以致于最终我们可以替整个 Ruby 社群创造一个有益的资源。

欢迎开票或发送一个带有改进的更新请求。在此提前感谢你的帮助！

# 口耳相传

一份社群驱动的风格指南，对一个社群来说，只是让人知道有这个社群。微博转发这份指南，分享给你的朋友或同事。我们得到的每个注解、建议或意见都可以让这份指南变得更好一点。而我们想要拥有的是最好的指南，不是吗？
