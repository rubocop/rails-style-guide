# 시작하며

> 롤 모델이 중요하다. <br/>
> -- Officer Alex J. Murphy / RoboCop

이 가이드는 루비 온 레일즈 4 개발을 위한 좋은 예제들과 코딩 스타일 규칙을 전달하고자 작성되었으며,
커뮤니티 기반 [루비 코딩 스타일 가이드](https://github.com/bbatsov/ruby-style-guide)을 보완하는 가이드입니다.

여기서 작성된 몇몇 예제들은 레일즈 4.0 이상의 버전에서만 적용될 수 있으니 참고하세요.

[Transmuter](https://github.com/TechnoGate/transmuter)를 이용하시면 이 가이드를 PDF 또는 HTML형식으로 만들 수 있습니다.

다음의 언어들로도 이 가이드를 읽어 볼 수 있습니다.

* [Chinese Simplified](https://github.com/JuanitoFatas/rails-style-guide/blob/master/README-zhCN.md)
* [Chinese Traditional](https://github.com/JuanitoFatas/rails-style-guide/blob/master/README-zhTW.md)
* [Japanese](https://github.com/satour/rails-style-guide/blob/master/README-jaJA.md)
* [Russian](https://github.com/arbox/rails-style-guide/blob/master/README-ruRU.md)
* [Turkish](https://github.com/tolgaavci/rails-style-guide/blob/master/README-trTR.md)
* [Korean](https://github.com/pureugong/rails-style-guide/blob/master/README-koKO.md)

# 레일즈 스타일 가이드

이 가이드는 다른 레일즈 개발자들과 유지가능한 코드를 작성하는 모범 예제들을 적용할 것을 추천합니다. 아래의 내용들은 실제 사용방법을 반영하기 때문에, 너무 이상적으로 만들어져 적용이 어렵다고 느끼는 사람들은 사용하지 않는 것이 좋습니다. - 아무리 좋다고 해도 말이다.

서로 관련있는 규칙에 따라 다양한 섹션으로 구분 지었으며, (자명하다고 판단된 것을 제외하고는)규칙에 대한 근거를 덧붙이려 노력하였습니다.

이 모든 규칙들이 하루 아침에 만들어 진것은 아닙니다. 대부분 저의 전문분야인 소프트웨어 엔지니어로서의 수 많은 경험과 레일즈 커뮤니티에서의 피드백과 제안, 또한 높은 평가를 받고 있는 레일즈 프로그래밍 리소스를 기반으로 하였습니다.

## 목차

* [Configuration](#configuration)
* [Routing](#routing)
* [Controllers](#controllers)
* [Models](#models)
  * [ActiveRecord](#activerecord)
  * [ActiveRecord Queries](#activerecord-queries)
* [Migrations](#migrations)
* [Views](#views)
* [Internationalization](#internationalization)
* [Assets](#assets)
* [Mailers](#mailers)
* [Time](#time)
* [Bundler](#bundler)
* [Flawed Gems](#flawed-gems)
* [Managing processes](#managing-processes)

## Configuration

* <a name="config-initializers"></a>
  `config/initializers`에 초기 설정 코드 둘 것. 그 코드들은 어플리케이션이 처음 구동 될 때 실행된다.
<sup>[[link](#config-initializers)]</sup>

* <a name="gem-initializers"></a>
  젬(gem)별로 각각의 초기 설정파일은 젬과 같은 이름으로 파일을 구분하여 둘 것.
  예를들면 `carrierwave.rb`, `active_admin.rb`, 등.
<sup>[[link](#gem-initializers)]</sup>

* <a name="dev-test-prod-configs"></a>
  개발, 테스트 그리고 배포 환경을 위해 설정들을 적절히 조절 할 것.
  (`config/environments/`아래에 각각에 상응하도록)
<sup>[[link](#dev-test-prod-configs)]</sup>

  * 사전 컴파일(있다면)에 대한 추가적인 Assets을 표시할 것:

    ```Ruby
    # config/environments/production.rb
    # Precompile additional assets (application.js, application.css,
    #and all non-JS/CSS are already added)
    config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
    ```

* <a name="app-config"></a>
  모든 환경에 적용되어야 하는 설정들은 `config/application.rb` 파일에 둘 것.
<sup>[[link](#app-config)]</sup>

* <a name="staging-like-prod"></a>
  배포 환경과 아주 유사한 추가적인 'staging' 환경을 생성할 것.
<sup>[[link](#staging-like-prod)]</sup>

* <a name="yaml-config"></a>
  다른 추가적인 설정들은 'config/'디렉토리 아래의 YAML파일에 둘 것.
<sup>[[link](#yaml-config)]</sup>

  레일즈 4.2에서 YAML 설정 파일들은 새로운 'config_for'매서드를 통해 쉽게 로드 될 수 있다:

  ```Ruby
  Rails::Application.config_for(:yaml_file)
  ```

## Routing

* <a name="member-collection-routes"></a>
  RESTful 리소스에 더 많은 액션(actions)을 추가할 필요가 있다면, (정말로 그게 다 필요한가??;;;) `member` 와 `collection` 라우트를 사용할 .
<sup>[[link](#member-collection-routes)]</sup>

  ```Ruby
  # 나쁜 예
  get 'subscriptions/:id/unsubscribe'
  resources :subscriptions

  # 좋은 예
  resources :subscriptions do
    get 'unsubscribe', on: :member
  end

  # 나쁜 예
  get 'photos/search'
  resources :photos

  # 좋은 예
  resources :photos do
    get 'search', on: :collection
  end
  ```

* <a name="many-member-collection-routes"></a>
  여러개의 'member/collection' 라우트를 정의해야 한다면, 
  block 문법을 대안으로 사용할 것.
<sup>[[link](#many-member-collection-routes)]</sup>

  ```Ruby
  resources :subscriptions do
    member do
      get 'unsubscribe'
      # more routes
    end
  end

  resources :photos do
    collection do
      get 'search'
      # more routes
    end
  end
  ```

* <a name="nested-routes"></a>
  엑티브 레코드(ActiveRecord) 모델간의 관계를 더 분명하게 표현하기 위해서는 중첩 라우트(routes)를 사용할 것.
<sup>[[link](#nested-routes)]</sup>

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

* <a name="namespaced-routes"></a>
  그룹과 관련된 액션(actions)은 namespace를 사용한 라우터를 이용할 것. 
<sup>[[link](#namespaced-routes)]</sup>

  ```Ruby
  namespace :admin do
    # Directs /admin/products/* to Admin::ProductsController
    # (app/controllers/admin/products_controller.rb)
    resources :products
  end
  ```

* <a name="no-wild-routes"></a>
  레거시(legacy)가 발생 할 수 있는 라우터를 절대 사용하지 말 것.
  GET방식의 요청으로 모든 컨트롤러에 모든 액션(actions)에 접근 할 수 있게 된다.
<sup>[[link](#no-wild-routes)]</sup>

  ```Ruby
  # 아주 나쁜 예
  match ':controller(/:action(/:id(.:format)))'
  ```

* <a name="no-match-routes"></a>
  '[:get, :post, :patch, :put, :delete]'중 여러 종류의 요청을 ':via'옵션을 사용하여 하나의 액션에 매핑할 필요가 있는 것이 아니라면 'match'를 라우터를 명시하는데 사용하지 말 것.
<sup>[[link](#no-match-routes)]</sup>

## Controllers

* <a name="skinny-controllers"></a>
  컨트롤러는 최대한 간결하게 - 컨트롤러는 단지 뷰 레이어를 위해 데이터를 가져오는 역할을 하고 어떠한 비즈니스 로직을 포함해서는 안된다. (모든 비즈니스 로직은 당연히 모델 안에서 구현되어야 한다.)
<sup>[[link](#skinny-controllers)]</sup>

* <a name="one-method"></a>
  컨트롤러의 각각의 액션은 (원칙적으로는) 단 하나의 매서드만을 호출 해야한다.
<sup>[[link](#one-method)]</sup>

* <a name="shared-instance-variables"></a>
  하나의 컨트롤러와 뷰 사이에서 두개 이상의 인스턴스 변수들을 공유하지 않는다.
<sup>[[link](#shared-instance-variables)]</sup>

## Models

* <a name="model-classes"></a>
  Introduce non-ActiveRecord model classes freely.
<sup>[[link](#model-classes)]</sup>

* <a name="meaningful-model-names"></a>
  약어를 쓰지않고 의미 있는 이름으로 (짧게) 모델을 명명한다.
<sup>[[link](#meaningful-model-names)]</sup>

* <a name="activeattr-gem"></a>
  엑티브 레코드의 데이터베이스 기능을 제외한 validation과 같은 기능만 필요한 모델 오브젝트가 필요하다면 [ActiveAttr](https://github.com/cgriego/active_attr) gem을 사용할 것.
<sup>[[link](#activeattr-gem)]</sup>

  ```Ruby
  class Message
    include ActiveAttr::Model

    attribute :name
    attribute :email
    attribute :content
    attribute :priority

    attr_accessible :name, :email, :content

    validates :name, presence: true
    validates :email, format: { with: /\A[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}\z/i }
    validates :content, length: { maximum: 500 }
  end
  ```

  더 많은 예제는 링크를 참조.
  [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr).

### ActiveRecord

* <a name="keep-ar-defaults"></a>
  특별한 이유가 아니라면(legacy가 있다든지..) ActiveRecord의 기본설정(테이블 이름, 기본키, 등)을 가능한 변경하지 말것.
<sup>[[link](#keep-ar-defaults)]</sup>

  ```Ruby
  # 나쁜 예 - 스키마 변경을 한다면 하지 말것!!
  class Transaction < ActiveRecord::Base
    self.table_name = 'order'
    ...
  end
  ```

* <a name="macro-style-methods"></a>
  클래스 상단에 매크로 성격의 매서드('has_many', 'validates', 등)를 그룹화 해둘 것.
<sup>[[link](#macro-style-methods)]</sup>

  ```Ruby
  class User < ActiveRecord::Base
    # keep the default scope first (if any)
    default_scope { where(active: true) }

    # constants come up next
    COLORS = %w(red green blue)

    # afterwards we put attr related macros
    attr_accessor :formatted_date_of_birth

    attr_accessible :login, :first_name, :last_name, :email, :password

    # followed by association macros
    belongs_to :country

    has_many :authentications, dependent: :destroy

    # and validation macros
    validates :email, presence: true
    validates :username, presence: true
    validates :username, uniqueness: { case_sensitive: false }
    validates :username, format: { with: /\A[A-Za-z][A-Za-z0-9._-]{2,19}\z/ }
    validates :password, format: { with: /\A\S{8,128}\z/, allow_nil: true}

    # next we have callbacks
    before_save :cook
    before_save :update_username_lower

    # other macros (like devise's) should be placed after the callbacks

    ...
  end
  ```

* <a name="has-many-through"></a>
  'has_and_belongs_to_many'보다 'has_many :through'를 사용할 것.
  'has_many:through' 사용하면 추가적인 속성과 validation을 조인하는 모델을 통해 사용할 수 있다.
<sup>[[link](#has-many-through)]</sup>

  ```Ruby
  # 그렇게 좋진 않은 예 - has_and_belongs_to_many 사용할 것.
  class User < ActiveRecord::Base
    has_and_belongs_to_many :groups
  end

  class Group < ActiveRecord::Base
    has_and_belongs_to_many :users
  end

  # 선호되는 방식 - has_many :through를 사용할 것..
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

* <a name="read-attribute"></a>
  `read_attribute(:attribute)`보다 `self[:attribute]` 을 사용할 것.
<sup>[[link](#read-attribute)]</sup>

  ```Ruby
  # 나쁜 예
  def amount
    read_attribute(:amount) * 100
  end

  # 좋은 예
  def amount
    self[:amount] * 100
  end
  ```

* <a name="write-attribute"></a>
  `write_attribute(:attribute, value)`보다 `self[:attribute] = value`를 사용할 것.
<sup>[[link](#write-attribute)]</sup>

  ```Ruby
  # 나쁜 예
  def amount
    write_attribute(:amount, 100)
  end

  # 좋은 예
  def amount
    self[:amount] = 100
  end
  ```

* <a name="sexy-validations"></a>
  항상 새로운 ["섹시한"
  validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/)을 사용할 것.
<sup>[[link](#sexy-validations)]</sup>

  ```Ruby
  # 나쁜 예
  validates_presence_of :email
  validates_length_of :email, maximum: 100

  # 좋은 예
  validates :email, presence: true, length: { maximum: 100 }
  ```

* <a name="custom-validator-file"></a>
  커스텀 validation을 한 번 이상 사용 한다거나 정규표현식을 이용한다면, 커스텀 validator 파일을 생성할 것.
<sup>[[link](#custom-validator-file)]</sup>

  ```Ruby
  # 나쁜 예
  class Person
    validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i }
  end

  # 좋은 예
  class EmailValidator < ActiveModel::EachValidator
    def validate_each(record, attribute, value)
      record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
    end
  end

  class Person
    validates :email, email: true
  end
  ```
* <a name="app-validators"></a>
  `app/validators`아래에 커스텀 validator파일을 둔다..
<sup>[[link](#app-validators)]</sup>

* <a name="custom-validators-gem"></a>
  여러 어플리케이션에서 같이 사용되는 커스텀 validator와 충분히 일반화 된 validators는 분리해서 gem으로 공유하여 사용할 것.
<sup>[[link](#custom-validators-gem)]</sup>

* <a name="named-scopes"></a>
  Use named scopes freely.
<sup>[[link](#named-scopes)]</sup>

  ```Ruby
  class User < ActiveRecord::Base
    scope :active, -> { where(active: true) }
    scope :inactive, -> { where(active: false) }

    scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
  end
  ```

* <a name="named-scope-class"></a>
  람다와 파라미터와 함께 정의 된 이름 있는 scope이 너무 복잡할 때, 'ActiveRecord::Relation'을 리턴하는 같은 기능을 수행하는 클래스 매서드를 만드는 것을 선호할 것. 
  아마 틀림없이 아래처럼 간단한 scope을 정의할 수 있을 것이다.

<sup>[[link](#named-scope-class)]</sup>

  ```Ruby
  class User < ActiveRecord::Base
    def self.with_orders
      joins(:orders).select('distinct(users.id)')
    end
  end
  ```
  같은 방식으로 이러한 코드 스타일은 이러한 방식의 매서드와 연결(chain)되어 질 수 없다는 것을 참고. 예를들어..

  ```Ruby
  # unchainable
  class User < ActiveRecord::Base
    def User.old
      where('age > ?', 80)
    end

    def User.heavy
      where('weight > ?', 200)
    end
  end
  ```
  여기서 'old'와 'heavy'는 개별적인 작업으로 분리되어 있지만 'User.old.heavy'라는 방법으로 이어질 수는 없다. 따라서 연결(chain)하기 위해서는 아래처럼 코딩할 것.

  ```Ruby
  # chainable
  class User < ActiveRecord::Base
    scope :old, -> { where('age > 60') }
    scope :heavy, -> { where('weight > 200') }
  end
  ```

* <a name="beware-update-attribute"></a>
  [`update_attribute`](http://api.rubyonrails.org/classes/ActiveRecord/Persistence.html#method-i-update_attribute)매서드의 작동 방법에 대하여 이해할 것.
  (`update_attributes`와는 다르게)모델 validation를 실행하지 않고 모델의 상태에 오류가 발생할 수 있다.
<sup>[[link](#beware-update-attribute)]</sup>

* <a name="user-friendly-urls"></a>
  사용자 편의의 URL을 사용할 것. 'id'보다는 URL에 모델의 기술적인 속성을 보여줄 것.
  여러가지 방법이 있을 수 있다.
<sup>[[link](#user-friendly-urls)]</sup>

  * 모델의 'to_param' 매서드를 오버라이드(override)한다. 레일즈에서 object에 해당하는 URL을 생성하기 위해 사용된다. 기본적으로 레코드의 'id'를 String으로 리턴한다. 이것은 오버라이드하여 사람이 읽을 수 있는 속성으로 바꾸어 질 수 있다.

      ```Ruby
      class Person
        def to_param
          "#{id} #{name}".parameterize
        end
      end
      ```
  이것을 URL친화적인 값으로 바꾸기 위해서는 'parameterize'가 string으로 불러져야 한다.
  객체의 id는 앞부분에 있어야만 ActiveRecord의 find 매서드에 의해 찾아 질 수 있다.

  * 'friendly_id' 젬을 사용할 것. 이것은 모델의 'id'대신 기술적인 숙성을 사용하여 사람이 읽을 수 있는 URL을 생성할 수 있게 해준다.
      ```Ruby
      class Person
        extend FriendlyId
        friendly_id :name, use: :slugged
      end
      ```

  사용법에 대한 더 많은 정보는 [gem documentation](https://github.com/norman/friendly_id)를 참고 할 것.

* <a name="find-each"></a>
  ActiveRecord 객체의 컬랙션(collection)의 반복하기 위해서는 'find_each'를 사용할 것.
  데이터베이스에서 레코드 컬랙션을 돌리기(loop) 것은 (예를 들어 'all'매서드 를 사용하여) 한번에 모든 객체들을 인스턴스화 하기 때문에 매우 비효율 적이다.
  그러한 경우에는 배치 작업 매서드를 통해 레코드들이 배치에서 돌게 하면서 메모리 소비를 줄일 수 있다.
<sup>[[link](#find-each)]</sup>


  ```Ruby
  # 나쁜 예
  Person.all.each do |person|
    person.do_awesome_stuff
  end

  Person.where('age > 21').each do |person|
    person.party_all_night!
  end

  # 좋은 예
  Person.find_each do |person|
    person.do_awesome_stuff
  end

  Person.where('age > 21').find_each do |person|
    person.party_all_night!
  end
  ```

* <a name="before_destroy"></a>
  [Rails creates callbacks for dependent
  associations](https://github.com/rails/rails/issues/3458)때문에, 항상 
  'prepend: true'와 validation을 수행하는 'before_destroy' 콜백을 호출할 것.
<sup>[[link](#before_destroy)]</sup>

  ```Ruby
  # 나쁜 예 (roles will be deleted automatically even if super_admin? is true)
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end

  # 좋은 예
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable, prepend: true

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end
  ```

### ActiveRecord Queries

* <a name="avoid-interpolation"></a>
  SQL injection 공격에 취약할 수 있기 때문에, 쿼리 안에 문자를 직접 넣는 것을 피할 것.
<sup>[[link](#avoid-interpolation)]</sup>

  ```Ruby
  # 나쁜 예 - 어떠한 파라미터든지 들어 갈 수 있음
  Client.where("orders_count = #{params[:orders]}")

  # 좋은 예 - 적절히 맞는 파라미터만 들어 갈 수 있음
  Client.where('orders_count = ?', params[:orders])
  ```

* <a name="named-placeholder"></a>
  쿼리에 하나 이상의 placeholder를 사용할 때는
  부분적인 placeholder말고 이름지어진 placeholder의 사용을 고려 할 것.
<sup>[[link](#named-placeholder)]</sup>

  ```Ruby
  # OK할만한...
  Client.where(
    'created_at >= ? AND created_at <= ?',
    params[:start_date], params[:end_date]
  )

  # 좋은 예
  Client.where(
    'created_at >= :start_date AND created_at <= :end_date',
    start_date: params[:start_date], end_date: params[:end_date]
  )
  ```

* <a name="find"></a>
  id를 통해 하나의 값을 조회할 때는 'where' 보다 'find' 사용할 것.
<sup>[[link](#find)]</sup>

  ```Ruby
  # 나쁜 예
  User.where(id: id).take

  # 좋은 예
  User.find(id)
  ```

* <a name="find_by"></a>
  어떠한 속성을 통해 하나의 값을 조회할 필요가 있따면 'where'보단 'find_by'를 사용할 것.
<sup>[[link](#find_by)]</sup>

  ```Ruby
  # 나쁜 예
  User.where(first_name: 'Bruce', last_name: 'Wayne').first

  # 좋은 예
  User.find_by(first_name: 'Bruce', last_name: 'Wayne')
  ```

* <a name="find_each"></a>
  많은 레코드의 프로세스가 필요하다면 'find_each'를 사용할 것.
<sup>[[link](#find_each)]</sup>

  ```Ruby
  # 나쁜 예 - loads all the records at once
  # users 테이블이 수천개의 행을 가지고있다면 매우 비효율적이다.
  User.all.each do |user|
    NewsMailer.weekly(user).deliver_now
  end

  # 좋은 예 - 배치(batch) 안에서 레코드를 가져오게 된다.
  User.find_each do |user|
    NewsMailer.weekly(user).deliver_now
  end
  ```

* <a name="where-not"></a>
  SQL에 'where.not'을 사용할 것.
<sup>[[link](#where-not)]</sup>

  ```Ruby
  # 나쁜 예
  User.where("id != ?", id)

  # 좋은 예
  User.where.not(id: id)
  ```

## Migrations

* <a name="schema-version"></a>
  'schema.rb' (또는 'structure.sql') 파일의 버전관리를 할 것.
<sup>[[link](#schema-version)]</sup>

* <a name="db-schema-load"></a>
  빈 database를 초기화 위해서는 'rake db:migrate'대신에 'rake db:schema:load'를 사용할 것.
<sup>[[link](#db-schema-load)]</sup>

* <a name="default-migration-values"></a>
  default 값들은 어플리케이션 층보다 마이그레이션 자체에서 지정되도록 할 것.
<sup>[[link](#default-migration-values)]</sup>

  ```Ruby
  # 나쁜 예 - 어플리케이션 층에서 default값을 정한 경우 
  def amount
    self[:amount] or 0
  end
  ```

  레일즈에서 테이블 기본 설정을 강요하는 것이 많은 레일즈 개발자들이 추천하지만 데이터가 많은 어플리케이션의 버그에 노출될 수 있는 아주 불안정한 접근방법이다. 그리고 대부분의 중요한 어플리케이션은 다들 어플리케이션과 하나의 데이터베이스를 공유하고 있기 때문에 레일즈 어플리케이션으로부터 데이터 무결성을 부여하는 것이 불가능한 사실을 고려해야 할 것이다.

* <a name="foreign-key-constraints"></a>식
  외래키 제약을 강요할 것. 레일즈 4.2의 ActiveRecord는 외래키 제약을 지원한다.
  <sup>[[link](#foreign-key-constraints)]</sup>

* <a name="change-vs-up-down"></a>
  (테이블 또는 컬럼을 추가하는) 구조적인 마이그레이션을 작성할 때는 'up'과 'down' 매서드 대신 'change'매서드를 사용할 것.
  <sup>[[link](#change-vs-up-down)]</sup>

  ```Ruby
  # 옛날 방식
  class AddNameToPeople < ActiveRecord::Migration
    def up
      add_column :people, :name, :string
    end

    def down
      remove_column :people, :name
    end
  end

  # 새롭게 선호되는 방식
  class AddNameToPeople < ActiveRecord::Migration
    def change
      add_column :people, :name, :string
    end
  end
  ```

* <a name="no-model-class-migrations"></a>
  마이그레이션에서 모델 클래스를 사용하지 말것. The model classes are constantly
  evolving and at some point in the future migrations that used to work might
  stop, because of changes in the models used.
<sup>[[link](#no-model-class-migrations)]</sup>

## Views

* <a name="no-direct-model-view"></a>
  view에서 직접적으로 모델 계층을 call 하지 말것.
<sup>[[link](#no-direct-model-view)]</sup>

* <a name="no-complex-view-formatting"></a>
  view에서는 절대 복잡한 구조를 만들지 말고, 그러한 구조를 view helper의 매서드나 모델쪽으로 분리할 것.
<sup>[[link](#no-complex-view-formatting)]</sup>

* <a name="partials"></a>
  템플릿과 레이아웃을 이용하여 코드의 중복을 최소화 할 것.
<sup>[[link](#partials)]</sup>

## Internationalization

* <a name="locale-texts"></a>
  뷰, 모델 그리고 컨트롤러에서는 로케일(locale)관련 설정이나 직접적인 strings 문자를 사용하지 않는다. 이러한 문자들은 'config/locales' 디렉터리 아래의 로케일 파일로 옮길 것.
<sup>[[link](#locale-texts)]</sup>

* <a name="translated-labels"></a>
  ActiveRecord 모델의 라벨에 대한 번역이 필요할 때는 'activerecord' scope을 사용할 것.
<sup>[[link](#translated-labels)]</sup>

  ```
  en:
    activerecord:
      models:
        user: Member
      attributes:
        user:
          name: 'Full name'
  ```
  그러면 'User.model_name.human'은 'Member'를 return하고
  'User.human_attribute_name("name")'은 "Full name"을 리턴 한다.
  이러한 속성들에 대한 번역은 뷰에서 라벨로 사용되어 진다.

* <a name="organize-locale-files"></a>
  Separate the texts used in the views from translations of ActiveRecord
  attributes. Place the locale files for the models in a folder `models` and the
  texts used in the views in folder `views`.
<sup>[[link](#organize-locale-files)]</sup>

  * 새로 만든 디렉터리에서 로케일(locale) 파일이 만들었다면, 로드 될 수 있도록 'application.rb' 파일에 설정해 주어야 한다.

      ```Ruby
      # config/application.rb
      config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
      ```

* <a name="shared-localization"></a>
  Place the shared localization options, such as date or currency formats, in
  files under the root of the `locales` directory.
<sup>[[link](#shared-localization)]</sup>

* <a name="short-i18n"></a>
  I18n의 짧은 형식의 매서드를 사용할 것. 
  'I18n.translate' =>'I18n.t'
  'I18n.localize' => 'I18n.l'
<sup>[[link](#short-i18n)]</sup>

* <a name="lazy-lookup"></a>
  뷰에서 사용되는 텍트트에 대해 'lazy' lookup을 사용하라.
  예를들어 아래와 같은 구조가 있다고 하자.
<sup>[[link](#lazy-lookup)]</sup>

  ```
  en:
    users:
      show:
        title: 'User details page'
  ```

  `users.show.title`의 값은 `app/views/users/show.html.haml`템플릿에서 아래와 같이 사용될 수 있다.

  ```Ruby
  = t '.title'
  ```

* <a name="dot-separated-keys"></a>
  컨트롤러와모델에서 scope 옵션보다 점으로 분리된 key를 사용할 것.
  읽기도 쉽고, 계층에서 찾아내기가 더 쉽다.
<sup>[[link](#dot-separated-keys)]</sup>

  ```Ruby
  # 나쁜 예
  I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]

  # 좋은 예
  I18n.t 'activerecord.errors.messages.record_invalid'
  ```

* <a name="i18n-guides"></a>
  Rails I18n과 관련된 더 상세한 정보는 [Rails
  Guides](http://guides.rubyonrails.org/i18n.html)를 참고하라.
<sup>[[link](#i18n-guides)]</sup>

## Assets

Use the [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) to leverage organization within
your application.

* <a name="reserve-app-assets"></a>
  Reserve `app/assets` for custom stylesheets, javascripts, or images.
<sup>[[link](#reserve-app-assets)]</sup>

* <a name="lib-assets"></a>
  어플리케이션 범위에 맞지 않는 직접 만들어진 라이브러리는 'lib/assets'에 두어라.
<sup>[[link](#lib-assets)]</sup>

* <a name="vendor-assets"></a>
  [jQuery](http://jquery.com/) 또는
  [bootstrap](http://twitter.github.com/bootstrap/)와 같은 
  제 3자에 의한 라이브러리는 'vendor/assets'에 둘 것.
<sup>[[link](#vendor-assets)]</sup>

* <a name="gem-assets"></a>
  가능하다면, 젬으로 만들어진 assets을 사용할 것(e.g.
  [jquery-rails](https://github.com/rails/jquery-rails),
  [jquery-ui-rails](https://github.com/joliss/jquery-ui-rails),
  [bootstrap-sass](https://github.com/thomas-mcdonald/bootstrap-sass),
  [zurb-foundation](https://github.com/zurb/foundation)).
<sup>[[link](#gem-assets)]</sup>

## Mailers

* <a name="mailer-name"></a>
  'SomethingMailer'와 같이 mailer의 이름을 정할 것.
  이러한 접미사가 없으면 어떠한 뷰가 어떠한 mailer에 관련되어 있는지 직관적이지 못하다.
<sup>[[link](#mailer-name)]</sup>

* <a name="html-plain-email"></a>
  HTML과 순수 텍스트 기반의 템플릿을 준비할 것.
<sup>[[link](#html-plain-email)]</sup>

* <a name="enable-delivery-errors"></a>
  개발환경에서 메일 전송에 대한 실패에 대해 에러가 발생할 수 있도록(true) 할 것.
  기본설정(default)은 발생하지 않게(false) 되어 있다.
  The errors are disabled by default.
<sup>[[link](#enable-delivery-errors)]</sup>

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.raise_delivery_errors = true
  ```

* <a name="local-smtp"></a>
  개발환경에서는 [Mailcatcher](https://github.com/sj26/mailcatcher)와 같은 로컬 SMTP를 사용할 것.
<sup>[[link](#local-smtp)]</sup>

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.smtp_settings = {
    address: 'localhost',
    port: 1025,
    # more settings
  }
  ```

* <a name="default-hostname"></a>
  호스트 이름을 설정할 것.
<sup>[[link](#default-hostname)]</sup>

  ```Ruby
  # config/environments/development.rb
  config.action_mailer.default_url_options = { host: "#{local_ip}:3000" }

  # config/environments/production.rb
  config.action_mailer.default_url_options = { host: 'your_site.com' }

  # in your mailer class
  default_url_options[:host] = 'your_site.com'
  ```

* <a name="url-not-path-in-email"></a>
  email에 사이트의 링크를 달고 싶다면, '_path'대신 '_url'매서드를 사용할 것.
  '_url'매서드는 '_path'매서드가 갖고 있지 않는 호스트 이름을 가지고 있다.
<sup>[[link](#url-not-path-in-email)]</sup>

  ```Ruby
  # 나쁜 예
  You can always find more info about this course
  <%= link_to 'here', course_path(@course) %>

  # 좋은 예
  You can always find more info about this course
  <%= link_to 'here', course_url(@course) %>
  ```

* <a name="email-addresses"></a>
  주소 형식의 폼을 적절하게 사용할 것.
  아래의 형식을 따라라.
<sup>[[link](#email-addresses)]</sup>

  ```Ruby
  # in your mailer class
  default from: 'Your Name <info@your_site.com>'
  ```

* <a name="delivery-method-test"></a>
  테스트 환경에서 e-mail 전송 매서드가 테스트로 설정(:test)이 되어 있는지 확인 할 것.
<sup>[[link](#delivery-method-test)]</sup>

  ```Ruby
  # config/environments/test.rb

  config.action_mailer.delivery_method = :test
  ```

* <a name="delivery-method-smtp"></a>
  개발 및 배포 환경에서는 e-mail 전송 매서드가 :smtp로 설정되어 있어야 함.
<sup>[[link](#delivery-method-smtp)]</sup>

  ```Ruby
  # config/environments/development.rb, config/environments/production.rb

  config.action_mailer.delivery_method = :smtp
  ```

* <a name="inline-email-styles"></a>
  html형식의 email을 전송할 때, 몇몇 클라이언트에서는 외부 스타일시트를 참조하는데 문제가 발생할 수 있기때문에, css는 인라인(inline)으로 작성되어야 한다.
  하지만 유지 보수가 힘들고 코드 중복이 발생할 수 있으므로,
  스타일을 변형하여 html와 반응하게 만들어주는 두 가지 gem을 참고하도록 하자.
  [premailer-rails](https://github.com/fphilipe/premailer-rails)와
  [roadie](https://github.com/Mange/roadie).
<sup>[[link](#inline-email-styles)]</sup>

* <a name="background-email"></a>
  페이지를 생성하면서 이메일을 전송하는 것을 피하도록 하자.
  페이지 로딩에 지연을 발생시키고, 여러 메일이 동시에 전송된다면 요청시간이 만료될 수 있다.
  따라서 [sidekiq](https://github.com/mperham/sidekiq) gem을 통해 background 작업처리로 이메일을 전송하도록 한다.
<sup>[[link](#background-email)]</sup>

## Time

* <a name="tz-config"></a>
  'application.rb'에 알맞게 timezone을 설정할 것.
<sup>[[link](#time-now)]</sup>

  ```Ruby
  config.time_zone = 'Eastern European Time'
  # 옵션 - note it can be only :utc or :local (default is :utc)
  config.active_record.default_timezone = :local
  ```

* <a name="time-parse"></a>
  `Time.parse`를 사용하지 말 것.
<sup>[[link](#time-parse)]</sup>

  ```Ruby
  # 나쁜 예
  Time.parse('2015-03-02 19:05:37') 
  # => 이 매서드는 시스템 timezone에서 시간이 주어진 것으로 가정함

  # 좋은 예
  Time.zone.parse('2015-03-02 19:05:37') 
  # => Mon, 02 Mar 2015 19:05:37 EET +02:00
  ```

* <a name="time-now"></a>
  'Time.now'를 사용하지 말 것.
<sup>[[link](#time-now)]</sup>

  ```Ruby
  # 나쁜 예
  Time.now # => timezone설정과 무관하게 system의 시간을 리턴

  # 좋은 예
  Time.zone.now # => Fri, 12 Mar 2014 22:04:47 EET +02:00
  Time.current # 위와 같지만 더 짧은 방법.
  ```

## Bundler

* <a name="dev-test-gems"></a>
  Gemfile의 각각의 그룹에 맞추어 개발 또는 테스트에 맞는 gem을 설정 할 것.
<sup>[[link](#dev-test-gems)]</sup>

* <a name="only-good-gems"></a>
  확실하게 알려진 gem을 사용 할 것.
  잘 알려지지 않은 gem에 대하여 고민중이라면 우선 소스코드와 리뷰에 대하여 잘 살펴볼 것.
<sup>[[link](#only-good-gems)]</sup>

* <a name="os-specific-gemfile-locks"></a>
  다른 OS를 사용하는 개발자들과 함께 프로젝트를 진행하게 되면 지속적으로 'Gemfile.lock'의 변경이 발생된다.
  따라서 Gemfile의 'darwin'그룹에는 OS X 기반의 gem을 명시하고, 'linux'그룹에는 'linux' 기반의 gem을 두어라.
<sup>[[link](#os-specific-gemfile-locks)]</sup>

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

  올바른 환경에서 적절한 gem을 요구하기위해서는 'config/application.rb'에 아래와 같이 추가하라.

  ```Ruby
  platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
  Bundler.require(platform)
  ```

* <a name="gemfile-lock"></a>
  'Gemfile.lock'파일은 버전관리에서 제거하지 말 것.
  이 파일은 무작위로 생성된 것이 아니라, 프로젝트를 진행하는 다른 개발자가 'bundle install'명령을 하였을 때 같은 버전의 gem을 사용할 수 있게 한다.
<sup>[[link](#gemfile-lock)]</sup>

## Flawed Gems
아래의 젬들은 문제가 있거나 다른 젬들에 의해 필요가 없어진 것들이다.
프로젝트에서 사용하는 것을 피할 것!

* [rmagick](http://rmagick.rubyforge.org/) - 메모리 소모로 악명이 높으므로 
  [minimagick](https://github.com/minimagick/minimagick)를 사용할 것.

* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - 자동으로 테스트를 실행하는 오래된 솔루션이다.
  [guard](https://github.com/guard/guard)와 
  [watchr](https://github.com/mynyml/watchr)을 사용할 것.

* [rcov](https://github.com/relevance/rcov) - 루비 1.9와 호환되지 않음. 대신 [SimpleCov](https://github.com/colszowka/simplecov)를 사용할 것.

* [therubyracer](https://github.com/cowboyd/therubyracer) - 메모리 소모가 크기 때문에, 대신 `node.js`를 사용할 것을 추천 함.

위의 젬 목록은 여전히 업데이트 중이다. 유명하지만 흠이 있는 젬을 알고 있다면 알려달라.

## Managing processes

* <a name="foreman"></a>
  프로젝트가 다양한 외부 프로세스에 의존적이라면 [foreman](https://github.com/ddollar/foreman)을 사용하여 관리할 것.
<sup>[[link](#foreman)]</sup>

# Further Reading

시간이 있다면 더 참고할만한 몇몇 훌륭한 레일즈 스타일과 관련된 자료들이 있다.

* [The Rails 4 Way](http://www.amazon.com/The-Rails-Addison-Wesley-Professional-Ruby/dp/0321944275)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)
* [The Cucumber Book](http://pragprog.com/book/hwcuc/the-cucumber-book)
* [Everyday Rails Testing with RSpec](https://leanpub.com/everydayrailsrspec)
* [Better Specs for RSpec](http://betterspecs.org)

# Contributing

이 문서는 아직 완성된 것이 아닙니다. 저는 이 가이드가 레일즈 코딩 스타일에 흥미 있는 사람들과 함께 만들어 가면서, 모든 루비 커뮤니티에게 유용한 자료가 되었으면 합니다.

개선을 위한 풀 리퀘스(pull requests)를 마음껏 보내주시고, 감사의 말씀을 미리 드립니다.

또한 이 프로젝트(와 RuboCop)에 기부를 원하시는 분은 [gittip](https://www.gittip.com/bbatsov)을 통해 참여할 수 있습니다.

[![Support via Gittip](https://rawgithub.com/twolfson/gittip-badge/0.2.0/dist/gittip.png)](https://www.gittip.com/bbatsov)

## How to Contribute?

[contribution guidelines](https://github.com/bbatsov/rails-style-guide/blob/master/CONTRIBUTING.md)을 참고하여 프로젝트에 쉽게 기여할 수 있습니다.

# License

![Creative Commons License](http://i.creativecommons.org/l/by/3.0/88x31.png)
[Creative Commons Attribution 3.0 Unported License](http://creativecommons.org/licenses/by/3.0/deed.en_US)의 라이센스가 적용되어 있습니다.

# Spread the Word

이 문서의 존재에 대하여 모른다면, 커뮤니티 기반의 스타일 가이드는 아무런 의미가 없습니다.
이 가이드에 대하여 트윗하거나, 친구와 동료들에게 공유하세요. 댓글, 의견 그리고 제안들을 주시면 조금 더 나은 가이드를 만드는데 도움이 될 것입니다. 
더 좋은 가이드를 만들고 싶지 않으신가요?

화이팅<br/>
[Bozhidar](https://twitter.com/bbatsov)
