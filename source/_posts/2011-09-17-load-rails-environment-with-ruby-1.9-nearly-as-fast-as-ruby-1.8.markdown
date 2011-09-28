---
layout: post
title: "Load Rails Environment with Ruby 1.9 Nearly as Fast as Ruby 1.8"
date: 2011-09-17 00:44
comments: true
categories: [Ruby, Rails]
---

## Use Xavier Shay's patched Ruby 1.9.3dev

There were [two patches for Ruby 1.9](http://www.rubyinside.com/ruby-1-9-3-faster-loading-times-require-4927.html) to resolve the load performance issue. Xavier Shay's approach, which use a hash data structure to store loaded files, absolutely can result much better performance.

Use Xavier Shay's patch may not easy, here is a small trick to install his fork of Ruby with rvm:

```bash Shell commands
$ cd ~/.rvm/repos
$ git clone git://github.com/xaviershay/ruby.git ruby-head
$ rvm install ruby-head --branch require-performance-fix

$ rvm use ruby-head
$ ruby --version
ruby 1.9.3dev (2011-05-31 trunk 31827) [x86_64-linux]
```

Yeah, it will be installed as `ruby-head`.

Note: Absolutely this should only be used in development environment, since it's a development version of Ruby 1.9.3.

### Updated on Sep 28, 2011:

The official Ruby 1.9.3 rc1 was released recently, it has almost same load performance compared to Xavier Shay's patched Ruby 1.9.3dev. Which make this trick no longer necessary.

## Tweak Gemfile to not require unnecessary gems immediately

Most gems in `development` and `test` group are unnecessary to be required immediately, gems for test can be required in `spec_helper.rb`. Here is part of my Gemfile for a small project:

```ruby Gemfile
group :development do
  gem 'bond', require: nil
  gem 'irbtools', require: nil
  gem 'irb_rocket', require: nil
  gem 'capistrano', require: nil
  gem 'rails3-generators', require: nil
  # Only used for mo/po file generation in development, see rake -T gettext.
  gem 'gettext', '>= 1.9.3', require: nil
end

group :development, :test do
  gem 'awesome_print', require: 'ap'
  gem 'factory_girl_rails'
  gem 'rspec-rails', '>= 2.5.0', require: nil
  gem 'capybara', '>= 0.3.6', require: nil
  gem 'capybara-webkit', '>= 1.0.0.beta4', require: nil
  gem 'launchy', require: nil
  gem 'cucumber-rails', require: nil
end

group :test do
  gem 'spork', '>= 0.9.0.rc5', require: nil
  gem 'rspec', '>= 2.5.0', require: nil
  gem 'remarkable', '>= 4.0.0.alpha4', require: nil
  gem 'remarkable_activemodel', '>= 4.0.0.alpha4', require: nil
  gem 'remarkable_mongoid', require: nil
  gem 'database_cleaner', '>= 0.5.0', require: nil

  gem 'guard', require: nil
  gem 'guard-bundler', require: nil
  gem 'guard-spork', require: nil
  gem 'guard-rspec', require: nil
  gem 'guard-cucumber', require: nil
end

group :linux do
  gem 'rb-inotify', require: nil
  gem 'libnotify', require: nil
  gem 'therubyracer', require: nil
end
```

## How about the result?

My recent project using Xavier Shay's Ruby 1.9.3dev:

```bash
$ time rake about
/home/rainux/.rvm/gems/ruby-head/gems/activesupport-3.0.10/lib/active_support/dependencies.rb:239:in `block in require': iconv will be deprecated in the future, use String#encode instead.
About your application's environment
Ruby version              1.9.3 (x86_64-linux)
RubyGems version          1.8.10
Rack version              1.2
Rails version             3.0.10
Action Pack version       3.0.10
Active Resource version   3.0.10
Action Mailer version     3.0.10
Active Support version    3.0.10
Middleware                ActionDispatch::Static, Rack::Lock, ActiveSupport::Cache::Strategy::LocalCache, Rack::Runtime, Rails::Rack::Logger, ActionDispatch::ShowExceptions, ActionDispatch::RemoteIp, Rack::Sendfile, ActionDispatch::Callbacks, ActionDispatch::Cookies, ActionDispatch::Session::CookieStore, ActionDispatch::Flash, ActionDispatch::ParamsParser, Rack::MethodOverride, ActionDispatch::Head, ActionDispatch::BestStandardsSupport, Warden::Manager, Sass::Plugin::Rack, Rack::Mongoid::Middleware::IdentityMap, Barista::Filter, Barista::Server::Proxy
Application root          /home/rainux/devel/a_rails_app
Environment               development
rake about  5.15s user 0.35s system 99% cpu 5.501 total
```

Compare to another project using Ruby 1.8.7 (actually, ree-1.8.7-2011.03), without tweak Gemfile:

```bash
$ time rake about
About your application's environment
Ruby version              1.8.7 (x86_64-linux)
RubyGems version          1.6.2
Rack version              1.2
Rails version             3.0.9
Active Record version     3.0.9
Action Pack version       3.0.9
Active Resource version   3.0.9
Action Mailer version     3.0.9
Active Support version    3.0.9
Application root          /home/rainux/devel/yet_another_rails_app
Environment               development
rake about  3.65s user 0.92s system 99% cpu 4.585 total
```
