---
layout: post
title: Working with Locales and Time Zones in Rails 🌏
date: '2015-06-08 10:20:00'
tags:
- rails
---

[Rails handles internationalization (i18n) really well](http://guides.rubyonrails.org/i18n.html), and you can set the default locale and time zone for your application very simply:

```ruby
# config/application.rb
module I18nTest
  class Application < Rails::Application
    config.i18n.default_locale = 'en'
    config.time_zone = 'London'
  end
end
```

## Setting the locale from the URL

First you need to wrap your applications routes in a scope defining the `:locale` parameter. Your application URLs will then look like `/en/controller/action` with the first segment telling Rails which locale it should use. The following regular expression just places a constraint on the locale parameter so that it must match one of the locales the application knows about.

```ruby
# config/routes.rb
scope ':locale', locale: /#{I18n.available_locales.join("|")}/ do
  # application routes...
end

# Catch all requests without a locale and redirect to the default...
match '*path', to: redirect("/#{I18n.default_locale}/%{path}"), constraints: lambda { |req| !req.path.starts_with? "/#{I18n.default_locale}/" }
match '', to: redirect("/#{I18n.default_locale}")
```

Contrary to the Rails guides examples, I prefer to use the [`around_action`](http://api.rubyonrails.org/classes/AbstractController/Callbacks/ClassMethods.html#method-i-around_action) hook with [`I18n.with_locale`](https://github.com/svenfuchs/i18n/blob/master/lib/i18n.rb#L252) class method to set the locale. Even though we may not be running in a multi-threaded environment, I feel it is more responsible way to handle the locale on a per-request basis like this.

We also override the `#default_url_options` method so the locale is automatically set when we use any of the `*_url` or `*_path` helper methods in our controller or views.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :with_locale

  private

  def with_locale
    I18n.with_locale(params[:locale]) { yield }
  end

  def default_url_options(options = {})
    { locale: I18n.locale }
  end
end
```

And there we have it, the application can respond with the correct content based on the URL. To make use of this in your application use &nbsp;the [`t`](http://api.rubyonrails.org/classes/AbstractController/Translation.html#method-i-t) and [`l`](http://api.rubyonrails.org/classes/AbstractController/Translation.html#method-i-l) helpers, these can be used in your controllers and views.

## Using i18n in your application

I recommend using them for everything, it will save **a lot** of pain if you ever need to handle multiple locales in the future; or you might just want to change the format of your dates.

First in Rails 4.1 you’ll want to `raise_on_missing_translations` so Rails will shout at you if any translations are missing.

```ruby
# config/environments/{test,development}.rb
Rails.application.configure do |config|
  config.action_view.raise_on_missing_translations = true
end
```

The `t` or `translate` helper accepts a simple `dot.notation` string to specify the translation you are trying to access (there are short hand options available but [check the Rails documentation for more information](http://guides.rubyonrails.org/i18n.html)). You can also pass in a hash to replace named keys in the translation string:

```yaml
# config/locales/en.yml

en:
  dot:
    notation: "Hello, my name is %{name}"
```
```ruby
# Example
t('dot.notation', name: 'Pete') #=> Hello, my name is Pete
```

The `l` or `localize` helper doesn’t get as much attention as it deserves, it’s very useful for localising dates, numbers, money and so on.

```yaml
# config/locales/en.yml

en:
  time:
    formats:
      short: '%d %b %H:%M'
```
```ruby
# Example
l(Time.current, format: :short) #=> 25 Apr 11:40
```

**Note:** Rails only provides the US English locale by default, but you can get hold of you locale from the [rails-i18n](https://github.com/svenfuchs/rails-i18n) gem.

## Find missing or unused i18n translations

As your application grows you’ll find the translation files can quickly become large difficult to keep track of everything in them. Thankfully [i18n-tasks](https://github.com/glebm/i18n-tasks) can ease the pain and help you track down missing or unused translations (credit to [Jessie Young](https://twitter.com/jessieay) for letting me know about this one):

    i18n-tasks health

Check out the readme for [i18n-tasks](https://github.com/glebm/i18n-tasks) for more details on using the tool, along with the [configuration](https://github.com/glebm/i18n-tasks#configuration) options available.

## Setting the time zone

It’s recommended to use the `around_action` hook and [`Time.use_zone`](http://api.rubyonrails.org/classes/Time.html#method-i-use_zone) as `Time.zone` is known to leak into other threads. In the example below the time zone is stored as an attribute on the user; a list of valid time zone values can be found by running `rake time:zones:all`.

**Note:** The `Time.use_zone` method is a core extension provided by ActiveSupport.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :with_time_zone, if: :current_user

  private

  def with_time_zone
    Time.use_zone(current_user.time_zone) { yield }
  end
end
```

Working with time zones can be tricky, Rails helps you in some places and then leaves you hanging in others (for good reason, but it can still catch you out). Let’s examine some of the gotchas:

### Working with ActiveRecord attributes and time zones

Good news! ActiveRecord helps you out by converting all values to UTC for storing in the database. When it fetches the record the value is converted to the expect time zone for that request (either from `config.time_zone` or if you’re using `Time.use_zone`).

### Working with the `Time` class

If you are using the [`Time`](http://www.ruby-doc.org/core-2.1.2/Time.html) class it’s important to specify the time zone you want to use. The easiest way to do this is to include the `zone` with `Time.zone`; this will use the expected time zone. If you just use `Time` then it will use the servers system time zone. For example [Rails provides some helpers](http://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html):

```ruby
Time.current # Rails helper for Time.zone.now
Time.zone.parse('2014-04-25 11:30:00')
```

Method calls like `2.hours.ago` use the time zone you’ve configured so these are safe to use as well. Even if you’re not building an application that cares about time zones at the moment, it could save you some pain in the future to use the time zone safe methods.
