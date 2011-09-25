# Abstract

The goal of this guide is to present a set of best practices and style
prescriptions for Ruby on Rails 3 development. It's a complimentary
guide to the already existing community-driven
[Ruby coding style guide](https://github.com/bbatsov/ruby-style-guide).

While in the guide the section ** Testing Rails applications ** is
after ** Developing Rails applications ** I'm a firm believer that BDD
is the best way to develop software. Keep that in mind.

Rails is an opinionated framework and this is an opinionated guide. In
my mind I'm totally certain that RSpec is superior to Test::Unit, Sass
is superior to CSS and Haml (Slim) is superior to Erb. So don't expect
to find any Test::Unit, CSS or Erb advice in here.

Some of the advice here is applicable only to Rails 3.1.

** This is guide is in a pre-alpha state! Quite A LOT is missing at
   this point. **

# Developing Rails applications

## Configuration

* Put custom initialization code in `config/initializers`.

## Routing



## Controllers

* Keep the controllers skinny - they should only retrieve data for the
  view layer and shouldn't contain any business logic (all the
  business logic should naturally reside in the model).

## Models

* Introduce non-ActiveRecord model classes freely.

### ActiveRecord

* Avoid altering ActiveRecord defaults (table names, primary key, etc)
  unless you have a very good reason (like a database that's not under
  your control).
* Group macro-style methods (`has_many`, `validates`, etc) in the
  beginning of the class definition.
* Always use the new
  ["sexy" validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/).

## Migrations

* Keep the schema.rb under version control.
* Avoid setting defaults in tables themselves. Use the model layer
  instead.

    ```Ruby
    def amount
      read_attributed(:amount) or 0
    end
    ```

## Views

* Never call the model layer directly from a view.
* Mitigate code duplication by using partial templates and layouts.

## Mailers

* Name the mailers `SomethingMailer`.
* Provide both HTML and plain-text view templates.

## Bundler

* Put gems used only for development or testing in the appropriate group in the Gemfile.
* Avoid OS specific gems. Bundler has no facilities to deal with those properly.
* Use only established gems in your projects. If you're contemplating
on including some little-known gem you should do a careful review of
its source code first.
* Do not remove the `Gemfile.lock` from version control.

# Testing Rails applications

The best approach to implementing new features is probably the BDD
approach. You start out by writing some high level feature tests (
generally written using Cucumber ), then you use this tests to drive
out the implementation of the feature. First you write view specs for
the feature and use those specs to create the relevant
views. Afterwards you create the specs for the controller(s) that will
be feeding data to the views and use those specs to implement the
controller. Finally you implement the models specs and the models
themselves.

## Cucumber

## RSpec

* Use just one expectation per example.
* make heavy use of describe and context
* use fabricators to create test objects
* make heavy use of mocks and stubs
* use `let` blocks instead of `before(:all)` blocks to create data for
  the spec examples. `let` blocks get lazily evaluated.

### Views

### Controllers

### Models

### Mailers

# Contributing

Nothing written in this guide is set in stone. It's my desire to work
together with everyone interested in Ruby coding style, so that we could
ultimately create a resource that will be beneficial to the entire Ruby
community.

Feel free to open tickets or send pull requests with improvements. Thanks in
advance for your help!

# Spread the Word

A community-driven style guide is of little use to a community that
doesn't know about its existence. Tweet about the guide, share it with
your friends and colleagues. Every comment, suggestion or opinion we
get makes the guide just a little bit better. And we want to have the
best possible guide, don't we?
