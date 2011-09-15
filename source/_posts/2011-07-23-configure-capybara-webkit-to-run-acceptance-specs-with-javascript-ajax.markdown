---
layout: post
title: "Configure capybara-webkit to Run Acceptance Specs With Javascript/AJAX"
date: 2011-07-23 20:09
comments: true
categories: [Ruby, Rails, Capybara, capybara-webkit, RSpec]
---

* Add `capybara-webkit` to your `Gemfile` and let [Guard::Bundler](https://github.com/guard/guard-bundler) install it automatically (or manually via `bundle install` if you don't use [Guard](https://github.com/guard/guard)).

``` ruby Gemfile
gem 'capybara-webkit', '>= 1.0.0.beta4'
```

* Set Javascript driver to `:webkit` for Capybara in `spec_helper.rb`.

``` ruby spec_helper.rb
Capybara.javascript_driver = :webkit
```

* Configure RSpec use non-transactional fixtures, configure [Database Cleaner](https://github.com/bmabey/database_cleaner) in `spec_helper.rb`.

  Notice with this setup, we'll only use truncation strategy when driver is not `:rack_test`. this will make normal specs run faster.

``` ruby spec_helper.rb
config.use_transactional_fixtures = false

config.before :each do
  if Capybara.current_driver == :rack_test
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
  else
    DatabaseCleaner.strategy = :truncation
  end
end

config.after :each do
  DatabaseCleaner.clean
end
```

* Tag your scenarios in `spec/acceptance/*_spec.rb` to use Javascript driver if necessary.

``` ruby some_spec.rb
scenario 'Create a lolita via AJAX', :js => true do
end
```

* Wait for any AJAX call to be completed in your specs. This is very important, or you will get many strange issues like no database record found, AJAX call get empty response with 0 status code, etc.

  For example if you have a simple AJAX form, the `success` callback will simply redirect browser to another page via `location.href = '/yet_another_page';`. You can use the following code to wait for it done.

``` ruby another_spec.rb
scenario 'Create a lolita via AJAX', :js => true do
  visit new_lolita_path

  click_on 'Submit'

  wait_until { page.current_path == lolita_path(Lolita.last) }

  # Your expections for the new page
end
```
