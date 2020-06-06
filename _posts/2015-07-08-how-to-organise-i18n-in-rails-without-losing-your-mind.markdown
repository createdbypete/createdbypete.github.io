---
layout: post
title: How to organise i18n in Rails without losing your mind 🤯
date: '2015-07-08 03:43:00'
tags:
- rails
---

I’ve written before about [working with Locales and Time Zones in Rails](/working-with-locales-and-time-zones-in-rails/), but I often feel the i18n library (short for internationalisation) is underused (appreciated?). Perhaps it is even avoided because of the perception it is more effort to develop with and harder to maintain.

This article will, I hope, open your mind to the idea that you will be better off using i18n in your application (even for a single language) and that it can be maintainable with some simple organisational pointers.

# Organisation

The number of i8n keys that your application accumulates can become overwhelming as your application develops. In my experience, the biggest pain point has been finding the key(s) you’re looking to update if everything is in one huge file such as the default `en.yml`.

Thankfully, you are not restricted to using a single file. We can adjust the i18n `load_path` in Rails and break up our translation files into more logically grouped files.

Rails doesn’t care or place special significance on this structure, all it cares about is the key hierarchy it eventually stores after parsing and merging the resulting data structure.

```ruby
# config/application.rb
config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
```
Now we organise our locale files to mirror the structure of our `app/`directory. So for every view we can have a corresponding locale file, for example:

    |- app
    |- - views
    |- - - home
    |- - - - _widget.html.erb
    |- - - - index.html.erb
    |- config
    |- - locales
    |- - - views
    |- - - - home
    |- - - - - _widget.en.yml
    |- - - - - index.en.yml

This pairing applies even to partials; in fact I prefix them with an underscore as well. Now I can easily find the translations for any view.

**Tip:** When you add a new locale file you will need to restart your Rails server as it will not be loaded automatically.

# The “Lazy” lookup

To compliment our new directory structure we can make a hierarchy in our locale file the same as the path to the view and use a more convenient way to to look up locales.

This removes the need for you to think up a hierarchy yourself and instead take advantage of a convention everyone can understand/lookup in documentation.
```yaml
# config/locales/views/home/index.en.yml
en:
  home:
    index:
      heading: Welcome to my homepage
```
```erb
# app/views/home/index.html.erb
<h1><%= t('.heading') %></h1>
```

The same pattern works for partials too. For example:

```yaml
# config/locales/views/home/_widget.en.yml
en:
  home:
    widget:
      heading: My Widget
```
```erb
# app/views/home/_widget.html.erb
<h3><%= t('.heading') %></h3>
```

# Global translations

What if you have some translations that need to be “global” to your application and don’t fit in a particular view or class, perhaps they are changeable and appear in many locations so having them repeated would be inconvenient.

The idea can be carefully applied at a global level too when it’s needed.

    |- config
    |- - locales
    |- - - global.en.yml

If you’ve only got a few keys then a single file is fine, if you start finding the file hard to read then break it up into smaller files inside a `global`directory or whatever makes sense for your domain.

# Using i18n outside of views

This approach is not limited to views, I find it really useful to use for validation messages in form object and models as well. For example:

```ruby
# app/forms/user/sign_up_form.rb
class User::SignUpForm
  include ActiveModel::Model
  attr_accessor :email

  validates :email, presence: true
end
```

Given a simple form object with basic presence validation as an example, the locale file below shows how you might customise the validation message. This makes use of Rails “lazy” lookup in a similar way to views and will affect this form only.

Global error messages can be set as well, check out the [rails-i18n gem](https://github.com/svenfuchs/rails-i18n/blob/master/rails/locale/en.yml) for an exhaustive list of the defaults you can change in Rails.

```yaml
# config/locales/forms/user/sign_up_form.en.yml
en:
  activemodel:
    errors:
      models:
        user/sign_up_form:
          attributes:
            email:
              blank: "must be supplied or you can't sign up!!!"
```

Taking care to create our locale file in a corresponding location to our class makes it easy to find these translations in the future.

# Naming keys

Naming things is hard. So don’t over think it, with our file structure this key is already confined to a single view and locale file. If it turns out to be a poor choice you can confidently go and change it.

My recommendation for naming your keys is to choose a name after the purpose of the key and not just use an underscored version of your translation. One exception to this would be things like models attribute labels where it makes sense to just use the translation as the key too.

```yaml
en:
  avoid:
    keys_like_this_are_fragile: Keys like this are fragile

  good:
    heading: This is a better key
    call_to_action_message: "I expect this translation to change so I haven't based my key on it"
```

Hopefully in the example above you can see the point I am trying to make. This isn’t a rule; more of a guide to help you make a more meaningful choice early on.

# Remove text from HAML and Slim templates

If you use, or have ever used a template engine such as [Slim](http://slim-lang.com/) or [HAML](http://haml.info/) they are great for structure but I find if you want to start adding text things can get out of hand.

```haml
# app/views/home/_widget.html.haml
%section
  %h1 My #{article.name}
  %p
    Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod
    tempor incididunt ut <strong>#{article.important_info}</strong> magna aliqua.
  %ul
    %li Ut enim quis nostrud exercitation
    %li ullamco laboris nisi ut aliquip ex ea commodo
    %li non proident, sunt in culpa qui
```

While the example above does not look like chaos it should be seen as a code “smell” in my opinion and one that’s easily solved with i18n:

```haml
# app/views/home/_widget.html.haml
%section
  %h1= t('.heading', name: article.name)
  %p= t('.body_html', important_info: article.important_info)
  %ul
    - t('.features').each do |key, translation|
      %li= translation
```

That’s better! You might notice I call `#each` on the `.feature` translation, this is so I can quickly mention [namespace lookups](http://guides.rubyonrails.org/i18n.html#bulk-and-namespace-lookup): if you call a key with children it will return a `Hash` with the translation as the value so it can used to create the list.

Also `.body_html` is considered [a HTML safe translation](http://guides.rubyonrails.org/i18n.html#using-safe-html-translations) so Rails won’t escape the `<strong>` tag.

In my opinion this keeps your focus on good structure without the noise and distraction of interpolated strings. This can be used for ERb templates too.

# Bonus level: Pluralization

I just wanted to share one of my favourite uses for i18n, handling pluralization. Not simply changing a singular form to a plural, I’m talking about adapting the entire sentence based on the count. For example, given the following locale file:

```yaml
en:
  comments:
    number_of_comments:
      zero: No one has left a comment yet
      one: There is %{count} comment
      other: There are %{count} comments
```

You can probably see what’s going on just from that example; depending on the `count` you pass to the translation it will select the appropriate response.

It should be noted that you don’t need to include all those options. You might not need to include a special case for `zero` in which case it will fallback to `other`.

```ruby
I18n.t('comments.number_of_comments', count: @comments.size)
```

This can help reduce a lot of unnecessary code in your view, not to mention the benefit of providing users with a better message.

The [documentation for i18n in Rails](http://guides.rubyonrails.org/i18n.html) has heaps more information about its features and creative uses. This article is more about how to better manage your locales and really only scratches the surface of what you can do with i18n.
