# Abstract

The goal of this guide is to present a set of best practices and style
prescriptions for Ruby on Rails 3 development. It's a complementary
guide to the already existing community-driven
[Ruby coding style guide](https://github.com/bbatsov/ruby-style-guide).

While in the guide the section [Testing Rails applications](#testing) is after
[Developing Rails applications](#developing) I truly believe that
Behaviour-Driven Development (BDD) is the best way to develop
software. Keep that in mind.

Rails is an opinionated framework and this is an opinionated guide. In
my mind I'm totally certain that
[RSpec](https://www.relishapp.com/rspec) is superior to Test::Unit,
[Sass](http://sass-lang.com/) is superior to CSS and
[Haml](http://haml-lang.com/) ([Slim](http://slim-lang.com/)) is
superior to Erb. So don't expect to find any Test::Unit, CSS or Erb
advice in here.

Some of the advice here is applicable only to Rails 3.1.

You can generate a PDF or an HTML copy of this guide using
[Transmuter](https://github.com/TechnoGate/transmuter).

## Table of Contents

* [Developing Rails Applications](#developing)
    * [Configuration](#configuration)
    * [Routing](#routing)
    * [Controllers](#controllers)
    * [Models](#models)
        * [ActiveRecord](#activerecord)
    * [Migrations](#migrations)
    * [Views](#views)
    * [Assets](#assets)
    * [Mailers](#mailers)
    * [Bundler](#bundler)
    * [Priceless Gems](#priceless)
    * [Flawed Gems](#flawed)
    * [Managing processes](#processes)
* [Testing Rails Applications](#testing)
    * [Cucumber](#cucumber)
    * [RSpec](#rspec)
        * [Views](#rspec_views)
        * [Contollers](#rspec_controllers)
        * [Models](#rspec_models)
        * [Mailers](#rspec_mailers)
        * [Uploaders](#rspec_uploaders)
* [Further Reading](#reading)
* [Contributing](#contributing)
* [Spread the word](#spreadtheword)

<a name="developing"/>
# Developing Rails applications

<a name="configuration"/>
## Configuration

* Put custom initialization code in `config/initializers`. The code in
  initializers executes on application startup.
* The initialization code for each gem should be in a separate file
  with the same name as the gem, for example `carrierwave.rb`,
  `rails_admin.rb`, etc.
* Adjust accordingly the settings for development, test and production
  environment (in the corresponding files under `config/environments/`)
  * Mark additional assets for precompilation (if any):

        ```Ruby
        # config/environments/production.rb
        # Precompile additional assets (application.js, application.css, and all non-JS/CSS are already added)
        config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
        ```

* While not strictly related to style, in order to use
[carrierwave](https://github.com/jnicklas/carrierwave) for the files
upload and [fog](https://github.com/geemus/fog) for file storage, some
configuration needs to be applied in the
`config/initializers/carrierwave.rb` file:

    ```Ruby
    # config/initializers/carrierwave.rb

    # Store the files locally for test environment
    if Rails.env.test?
      CarrierWave.configure do |config|
        config.storage = :file
        config.enable_processing = false
      end
    end

    # Using Amazon S3 for Development and Production
    if Rails.env.development? || Rails.env.production?
      CarrierWave.configure do |config|
        config.root = Rails.root.join('tmp')
        config.cache_dir = 'uploads'

        config.storage = :fog
        config.fog_credentials = {
            provider: 'AWS',
            aws_access_key_id: 'your_access_key_id',
            aws_secret_access_key: 'your_secret_access_key',
        }
        config.fog_directory = 'your_bucket'
      end
    end
   ```

  * Do not use `fog` for the test environment, use `file` storage instead.
  * Use `fog` for the development environment. This will prevent
    unexpected problems on production.

<a name="routing"/>
## Routing

* When you need to add more actions to a RESTful resource (do you
  really need them at all?) use `member` and `collection` routes.

    ```Ruby
    # bad
    get 'subscriptions/:id/unsubscribe'
    resources :subscriptions

    # good
    resources :subscriptions do
      get 'unsubscribe', :on => :member
    end

    # bad
    get 'photos/search'
    resources :photos

    # good
    resources :photos do
      get 'search', :on => :collection
    end
    ```

* If you need to define multiple `member/collection` routes use the
  alternative block syntax.

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

* Use nested routes to express better the relationship between
  ActiveRecord models.

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

* Use namespaced routes to group related actions.

    ```Ruby
    namespace :admin do
      # Directs /admin/products/* to Admin::ProductsController
      # (app/controllers/admin/products_controller.rb)
      resources :products
    end
    ```

* Never use the legacy wild controller route. This route will make all
  actions in every controller accessible via GET requests.

    ```Ruby
    # very bad
    match ':controller(/:action(/:id(.:format)))'
    ```

<a name="controllers"/>
## Controllers

* Keep the controllers skinny - they should only retrieve data for the
  view layer and shouldn't contain any business logic (all the
  business logic should naturally reside in the model).
* Each controller action should (ideally) invoke only one method other
  than an initial find or new.
* Share no more than two instance variables between a controller and a view.

<a name="models"/>
## Models

* Introduce non-ActiveRecord model classes freely.
* Name the models with meaningful (but short) names without abbreviations.

<a name="activerecord"/>
### ActiveRecord

* Avoid altering ActiveRecord defaults (table names, primary key, etc)
  unless you have a very good reason (like a database that's not under
  your control).
* Group macro-style methods (`has_many`, `validates`, etc) in the
  beginning of the class definition.
* Always use the new
  ["sexy" validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/).
* When a custom validation is used more than once or the validation is some regular expression mapping,
create a custom validator file.

    ```Ruby
    # bad
    class Person
      validates :email, format: { with: /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i }
    end

    # good
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
      end
    end


    class Person
      validates :email, email: true
    end
    ```

* All custom validators should be moved to a shared gem.
* Use named scopes freely.
* When a named scope, defined with a lambda and parameters, becomes too
complicated it is preferable to make a class method instead which serves
the same purpose of the named scope and returns and
`ActiveRecord::Relation` object.
* Beware of the behavior of the `update_attribute` method. It doesn't
  run the model validations (unlike `update_attributes`) and could easily corrupt the model state.

<a name="migrations"/>
## Migrations

* Keep the `schema.rb` under version control.
* Use `rake db:schema:load` instead of `rake db:migrate` to initialize
  an empty database.
* Avoid setting defaults in the tables themselves. Use the model layer
  instead.

    ```Ruby
    def amount
      read_attribute(:amount) or 0
    end
    ```

* When writing constructive migrations (adding tables or columns), use
  the new Rails 3.1 way of doing the migrations - use the `change`
  method instead of `up` and `down` methods.

<a name="views"/>
## Views

* Never call the model layer directly from a view.
* Never make complex formatting in the views, export the formatting to
  a method in the view helper or the model.
* Mitigate code duplication by using partial templates and layouts.
* Add
  [client side validation](https://github.com/bcardarella/client_side_validations)
  for the custom validators. The steps to do this are:
  * Declare a custom validator which extends `ClientSideValidations::Middleware::Base`

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

  * Create a new file
    `public/javascripts/rails.validations.custom.js.coffee` and add a
    reference to it in your `application.js.coffee` file:

        ```Ruby
        # app/assets/javascripts/application.js.coffee
        #= require rails.validations.custom
        ```

  * Add your client-side validator:

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

<a name="assets"/>
## Assets

Use the [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) to leverage organization within
your application.

* Reserve `app/assets` for custom stylesheets, javascripts, or images.
* Third party code such as [jQuery](http://jquery.com/) or [bootstrap](http://twitter.github.com/bootstrap/)
  should be placed in `vendor/assets`.
* When possible, use gemified versions of assets (e.g. [jquery-rails](https://github.com/rails/jquery-rails)).

<a name="mailers"/>
## Mailers

* Name the mailers `SomethingMailer`. Without the Mailer suffix it
  isn't immediately apparent what's a mailer and which views are
  related to the mailer.
* Provide both HTML and plain-text view templates.
* Enable errors raised on failed mail delivery in your development environment. The errors are disabled by default.

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* Use `smtp.gmail.com` for SMTP server in the development environment
  (unless you have local SMTP server, of course).

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      address: 'smtp.gmail.com',
      # more settings
    }
    ```

* Provide default settings for the host name.

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {host: "#{local_ip}:3000"}


    # config/environments/production.rb
    config.action_mailer.default_url_options = {host: 'your_site.com'}

    # in your mailer class
    default_url_options[:host] = 'your_site.com'
    ```

* If you need to use a link to your site in an email, always use the
  `_url`, not `_path` methods. The `_url` methods include the host
  name and the `_path` methods don't.

    ```Ruby
    # wrong
    You can always find more info about this course
    = link_to 'here', url_for(course_path(@course))

    # right
    You can always find more info about this course
    = link_to 'here', url_for(course_url(@course))
    ```

* Format the from and to addresses properly. Use the following format:

    ```Ruby
    # in your mailer class
    default from: 'Your Name <info@your_site.com>'
    ```

* Make sure that the e-mail delivery method for your test environment is set to `test`:

    ```Ruby
    # config/environments/test.rb

    config.action_mailer.delivery_method = :test
    ```

* The delivery method for development and production should be `smtp`:

    ```Ruby
    # config/environments/development.rb, config/environments/production.rb

    config.action_mailer.delivery_method = :smtp
    ```

<a name="bundler"/>
## Bundler

* Put gems used only for development or testing in the appropriate group in the Gemfile.
* Use only established gems in your projects. If you're contemplating
on including some little-known gem you should do a careful review of
its source code first.
* OS-specific gems will by default result in a constantly changing `Gemfile.lock`
for projects with multiple developers using different operating systems.
Add all OS X specific gems to a `darwin` group in the Gemfile, and all Linux
specific gems to a `linux` group:

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

    To require the appropriate gems in the right environment, add the
    following to `config/application.rb`:

    ```Ruby
    platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
    Bundler.require(platform)
    ```

* Do not remove the `Gemfile.lock` from version control. This is not
  some randomly generated file - it makes sure that all of your team
  members get the same gem versions when they do a `bundle install`.

<a name="priceless"/>
## Priceless Gems

One of the most important programming principles is "Don't reinvent
the wheel!". If you're faced with a certain task you should always
look around a bit for existing solutions, before unrolling your
own. Here's a list of some "priceless" gems (all of them Rails 3.1
compliant) that are useful in many Rails projects:

* [rspec-rails](https://github.com/rspec/rspec-rails) - RSpec is a
  replacement for Test::MiniTest. I cannot recommend highly enough
  RSpec. rspec-rails provides Rails integration for RSpec.
* [cucumber-rails](https://github.com/cucumber/cucumber-rails) -
  Cucumber is the premium tool to develop feature tests in
  Ruby. cucumber-rails provides Rails integration for Cucumber.
* [haml](http://haml-lang.org) - HAML is a concise templating language, considered by many
  (including your truly) far superior to Erb.
* [slim](http://slim-lang.com) - Slim is a concise templating
  language, considered by many far superior to HAML (not to mention
  Erb). The only thing stopping me from using Slim massively is the
  lack of good support in major editors/IDEs. Its performance is
  phenomenal.
* [simple_form](https://github.com/plataformatec/simple_form) - once you've used simple_form (or formtastic)
  you'll never want to hear about Rails's default forms. It has a
  great DSL for building forms and no opinion on markup.
* [fabrication](http://fabricationgem.org/) - a great fixture
  replacement (editor's choice).
* [factory_girl](https://github.com/thoughtbot/factory_girl) - an
  alternative to fabrication. Nice and mature fixture
  replacement. Spiritual ancestor of fabrication.
* [machinist](https://github.com/notahat/machinist) - Fixtures aren't
  fun. Machinist is.
* [faker](http://faker.rubyforge.org/) - handy gem to generate dummy data (names, addresses, etc).
* [guard](https://github.com/guard/guard) - fantastic gem that monitors file changes and invokes
tasks based on them. Loaded with lots of useful extension. Far
superior to autotest and watchr.
* [spork](https://github.com/timcharper/spork) - A DRb server for
  testing frameworks (RSpec / Cucumber currently) that forks before
  each run to ensure a clean testing state. Simply put it preloads a
  lot of test environment and as consequence the startup time of your
  tests in greatly decreased. Absolute must have!
* [simplecov](https://github.com/colszowka/simplecov) - code coverage tool. Unlike RCov it's fully
compatible with Ruby 1.9. Generates great reports. Must have!
* [simplecov-rcov](https://github.com/fguillen/simplecov-rcov) - RCov formatter for SimpleCov. Useful if you're
  trying to use SimpleCov with the Hudson contininous integration
  server.
* [capybara](https://github.com/jnicklas/capybara) - Capybara aims to
  simplify the process of integration testing Rack applications, such
  as Rails, Sinatra or Merb. Capybara simulates how a real user would
  interact with a web application. It is agnostic about the driver
  running your tests and currently comes with Rack::Test and Selenium
  support built in. HtmlUnit, WebKit and env.js are supported through
  external gems. Works great in combination with RSpec & Cucumber.
* [devise](https://github.com/plataformatec/devise) - Devise is
  full-featured authentication solution for Rails applications. In
  most cases it's preferable to use devise to unrolling your custom
  authentication solution.
* [carrierwave](https://github.com/jnicklas/carrierwave) - the
  ultimate file upload solution for Rails. Support both local and
  cloud storage for the uploaded files (and many other cool
  things). Integrates great with ImageMagick for image post-processing.
* [kaminari](https://github.com/amatsuda/kaminari) - Great paginating solution.
* [feedzirra](https://github.com/pauldix/feedzirra) - Very fast and flexible RSS/Atom feed parser.
* [sunspot](https://github.com/sunspot/sunspot) - SOLR powered
    full-text search engine.
* [client_side_validations](https://github.com/bcardarella/client_side_validations) - Fantastic gem
  that automatically creates JavaScript client-side validations from
  your existing server-side model validations. Highly recommended!
* [rails_admin](https://github.com/sferik/rails_admin) - With Rails Admin the creating of admin interface
  for your Rails app is child's play. You get a nice dashboard, CRUD
  UI and lots more. Very flexible and customizable.

This list is not exhaustive and other gems might be added to it along
the road. All of the gems on the list are field tested, have active
development and community and are known to be of good code quality.

<a name="flawed"/>
## Flawed Gems

This is a list of gems that are either problematic or superseded by
other gems. You should avoid using them in your projects.

* [rmagick](http://rmagick.rubyforge.org/) - this gem is notorious for its memory consumption. Use
[minimagick](https://github.com/probablycorey/mini_magick) instead.
* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - old solution for running tests automatically. Far
inferior to guard and [watchr](https://github.com/mynyml/watchr).
* [rcov](https://github.com/relevance/rcov) - code coverage tool, not
  compatible with Ruby 1.9. Use SimpleCov instead.
* [therubyracer](https://github.com/cowboyd/therubyracer) - the use of
  this gem in production is strongly discouraged as it uses a very large amount of
  memory.

This list is also a work in progress. Please, let me know if you know
other popular, but flawed gems.

<a name="processes"/>
## Managing processes

* If your projects depends on various external processes use
  [foreman](https://github.com/ddollar/foreman) to manage them.

<a name="testing"/>
# Testing Rails applications

The best approach to implementing new features is probably the BDD
approach. You start out by writing some high level feature tests
(generally written using Cucumber), then you use these tests to drive
out the implementation of the feature. First you write view specs for
the feature and use those specs to create the relevant
views. Afterwards you create the specs for the controller(s) that will
be feeding data to the views and use those specs to implement the
controller. Finally you implement the models specs and the models
themselves.

<a name="cucumber"/>
## Cucumber

* Tag your pending scenarios with `@wip` (work in progress).  These
scenarios will not be taken into account and will not be marked as
failing.  When finishing the work on a pending scenario and
implementing the functionality it tests, the tag `@wip` should be
removed in order to include this scenario in the test suite.
* Setup your default profile to exclude the scenarios tagged with
`@javascript`.  They are testing using the browser and disabling them
is recommended to increase the regular scenarios execution speed.
* Setup a separate profile for the scenarios marked with `@javascript` tag.
  * The profiles can be configured in the `cucumber.yml` file.

        ```Ruby
        # definition of a profile:
        profile_name: --tags @tag_name
        ```

  * A profile is run with the command:

        ```
        cucumber -p profile_name
        ```

* If using [fabrication](http://fabricationgem.org/) for fixtures
  replacement, use the predefined
  [fabrication steps](http://fabricationgem.org/#!cucumber-steps)
* Do not use the old `web_steps.rb` step definitions!
[The web steps were removed from the latest version of Cucumber.](http://aslakhellesoy.com/post/11055981222/the-training-wheels-came-off) Their
usage leads to the creation of verbose scenarios that do not properly
reflect the application domain.
* When checking for the presence of an element with visible text
  (link, button, etc.) check for the text, not the element id. This
  can detect problems with the i18n.
* Create separate features for different functionality regarding the same kind of objects:

    ```Ruby
    # bad
    Feature: Articles
    # ... feature  implementation ...

    # good
    Feature: Article Editing
    # ... feature  implementation ...

    Feature: Article Publishing
    # ... feature  implementation ...

    Feature: Article Search
    # ... feature  implementation ...

    ```

* Each feature has three main components
  * Title
  * Narrative - a short explanation what the feature is about.
  * Acceptance criteria - the set of scenarios each made up of individual steps.
* The most common format is known as the Connextra format.

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

This format is the most common but is not required, the narrative can
be free text depending on the complexity of the feature.

* Use Scenario Outlines freely to keep the scenarios DRY.

    ```Ruby
    Scenario Outline: User cannot register with invalid e-mail
      When I try to register with an email "<email>"
      Then I should see the error message "<error>"

    Examples:
      |email         |error                 |
      |              |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* The steps for the scenarios are in `.rb` files under the
`step_definitions` directory. The naming convention for the steps file
is `[description]_steps.rb`.  The steps can be separated into
different files based on different criterias. It is possible to have
one steps file for each feature (`home_page_steps.rb`).  There also
can be one steps file for all features for a particular object
(`articles_steps.rb`).
* Use multiline step arguments to avoid repetition

    ```Ruby
    Scenario: User profile
      Given I am logged in as a user "John Doe" with an e-mail "user@test.com"
      When I go to my profile
      Then I should see the following information:
        |First name|John         |
        |Last name |Doe          |
        |E-mail    |user@test.com|

    # the step:
    Then /^I should see the following information:$/ do |table|
      table.raw.each do |field, value|
        find_field(field).value.should =~ /#{value}/
      end
    end
    ```

* Use compound steps to keep the scenario DRY

    ```Ruby
    # ...
    When I subscribe for news from the category "Technical News"
    # ...

    # the step:
    When /^I subscribe for news from the category "([^"]*)"$/ do |category|
      steps %Q{
        When I go to the news categories page
        And I select the category #{category}
        And I click the button "Subscribe for this category"
        And I confirm the subscription
      }
    end
    ```

<a name="rspec"/>
## RSpec

* Use just one expectation per example.

    ```Ruby
    # bad
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

    # good
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

* Make heavy use of `describe` and `context`
* Name the `describe` blocks as follows:
  * use "description" for non-methods
  * use pound "#method" for instance methods
  * use dot ".method" for class methods

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

* Use [fabricators](http://fabricationgem.org/) to create test
  objects.
* Make heavy use of mocks and stubs

    ```Ruby
    # mocking a model
    article = mock_model(Article)

    # stubbing a method
    Article.stub(:find).with(article.id).and_return(article)
    ```

* When mocking a model, use the `as_null_object` method. It tells the
  output to listen only for messages we expect and ignore any other
  messages.

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* Use `let` blocks instead of `before(:all)` blocks to create data for
  the spec examples. `let` blocks get lazily evaluated.

    ```Ruby
    # use this:
    let(:article) { Fabricate(:article) }

    # ... instead of this:
    before(:each) { @article = Fabricate(:article) }
    ```

* Use `subject` when possible

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* Use `specify` if possible. It is a synonym of `it` but is more readable when there is no docstring.

    ```Ruby
    # bad
    describe Article do
      before { @article = Fabricate(:article) }

      it 'is not published on creation' do
        @article.should_not be_published
      end
    end

    # good
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* Use `its` when possible

    ```Ruby
    # bad
    describe Article do
      subject { Fabricate(:article) }

      it 'has the current date as creation date' do
        subject.creation_date.should == Date.today
      end
    end

    # good
    describe Article do
      subject { Fabricate(:article) }
      its(:creation_date) { should == Date.today }
    end
    ```

<a name="rspec_views"/>
### Views

* The directory structure of the view specs `spec/views` matches the
  one in `app/views`. For example the specs for the views in
  `app/views/users` are placed in `spec/views/users`.
* The naming convention for the view specs is adding `_spec.rb` to the
  view name, for example the view `_form.html.haml` has a
  corresponding spec `_form.html.haml_spec.rb`.
* `spec_helper.rb` need to be required in each view spec file.
* The outer `describe` block uses the path to the view without the
  `app/views` part. This is used by the `render` method when it is
  called without arguments.

    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.html' do
      # ...
    end
    ```

* Always mock the models in the view specs. The purpose of the view is
  only to display information.
* The method `assign` supplies the instance variables which the view
  uses and are supplied by the controller.

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

* Prefer the capybara negative selectors over should_not with the positive.

    ```Ruby
    # bad
    page.should_not have_selector('input', type: 'submit')
    page.should_not have_xpath('tr')

    # good
    page.should have_no_selector('input', type: 'submit')
    page.should have_no_xpath('tr')
    ```

* When a view uses helper methods, these methods need to be
  stubbed. Stubbing the helper methods is done on the `template`
  object:

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

* The helpers specs are separated from the view specs in the `spec/helpers` directory.

<a name="rspec_controllers"/>
### Controllers

* Mock the models and stub their methods. Testing the controller should not depend on the model creation.
* Test only the behaviour the controller should be responsible about:
  * Execution of particular methods
  * Data returned from the action - assigns, etc.
  * Result from the action - template render, redirect, etc.

        ```Ruby
        # Example of a commonly used controller spec
        # spec/controllers/articles_controller_spec.rb
        # We are interested only in the actions the controller should perform
        # So we are mocking the model creation and stubbing its methods
        # And we concentrate only on the things the controller should do

        describe ArticlesController do
          # The model will be used in the specs for all methods of the controller
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

* Use context when the controller action has different behaviour depending on the received params.

    ```Ruby
    # A classic example for use of contexts in a controller spec is creation or update when the object saves successfully or not.

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

<a name="rspec_models"/>
### Models

* Do not mock the models in their own specs.
* Use fabrication to make real objects.
* It is acceptable to mock other models or child objects.
* Create the model for all examples in the spec to avoid duplication.

    ```Ruby
    describe Article
      let(:article) { Fabricate(:article) }
    end
    ```

* Add an example ensuring that the fabricated model is valid.

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* Add a separate `describe` for each attribute which has validations.

    ```Ruby
    describe Article
      describe '#title'
        it 'is required' do
          article.title = nil
          article.should_not be_valid
        end
      end
    end
    ```

* When testing uniqueness of a model attribute, name the other object `another_object`.

    ```Ruby
    describe Article
      describe '#title'
        it 'is unique' do
          another_article = Fabricate.build(:article, title: article.title)
          another_article.should_not be_valid
        end
      end
    end
    ```

<a name="rspec_mailers"/>
### Mailers

* The model in the mailer spec should be mocked. The mailer should not depend on the model creation.
* The mailer spec should verify that:
  * the subject is correct
  * the receiver e-mail is correct
  * the e-mail is sent to the right e-mail address
  * the e-mail contains the required information

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

<a name="rspec_uploaders"/>
### Uploaders

* What we can test about an uploader is whether the images are resized correctly.
Here is a sample spec of a [carrierwave](https://github.com/jnicklas/carrierwave) image uploader:

    ```Ruby

    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # Enable images processing before executing the examples
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # Create a new uploader. The model is mocked as the uploading and resizing images does not depend on the model creation.
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # Disable images processing after executing the examples
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # Testing whether image is no larger than given dimensions
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # Testing whether image has the exact dimensions
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end

    ```

<a name="reading"/>
# Further Reading

There are a few excellent resources on Rails style, that you should
consider if you have time to spare:

* [The Rails 3 Way](http://tr3w.com/)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)

<a name="contributing"/>
# Contributing

Nothing written in this guide is set in stone. It's my desire to work
together with everyone interested in Rails coding style, so that we could
ultimately create a resource that will be beneficial to the entire Ruby
community.

Feel free to open tickets or send pull requests with improvements. Thanks in
advance for your help!

<a name="spreadtheword"/>
# Spread the Word

A community-driven style guide is of little use to a community that
doesn't know about its existence. Tweet about the guide, share it with
your friends and colleagues. Every comment, suggestion or opinion we
get makes the guide just a little bit better. And we want to have the
best possible guide, don't we?
