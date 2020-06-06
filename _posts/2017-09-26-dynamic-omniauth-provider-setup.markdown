---
layout: post
title: Dynamic OmniAuth Provider Setup 🔐
date: '2017-09-26 03:04:00'
tags:
- rails
---

[OmniAuth](https://github.com/intridea/omniauth) is very flexible gem by design but I think some of it’s more powerful features get overlooked. One of these features is the setup phase; the setup phase allows for request-time modification of an OmniAuth strategy. In this article I’m going to demonstrate how to dynamically set the strategy options so your application users could specify the API credentials used by OmniAuth.

We’ll be using Rails 4 and Omniauth 1.2 (versions at the time of writing). We’ll use the [Twitter strategy](https://github.com/arunagw/omniauth-twitter) for the examples but the same principles can be applied to any strategy.

The [OmniAuth wiki provides an overview of the different ways you can use the setup phase](https://github.com/intridea/omniauth/wiki/Setup-Phase), in the wiki examples a `lambda` is used but we’ll be moving it into a class called `OmniauthSetup`. Let’s configure OmniAuth to use the class.

```ruby
# config/initializers/omniauth.rb
require 'omniauth_setup'

Rails.application.config.middleware.use OmniAuth::Builder do
  provider :twitter, setup: OmniauthSetup
end
```

## Writing the class

The `setup` key tells OmniAuth to call our new class for the setup phase. You can still include the `key` and `secret` here, they would be used as defaults but you can leave them out if you don’t want to have a default defined here.

Time to expand our `OmniauthSetup` class to do something useful. For this example, let’s assume we have a model called `Account` with an attribute called `subdomain` where we store the subdomain for this account. We use a subdomain because the OmniAuth routes aren’t easy to change, so we use the subdomain in the request to find the account record in our database.

```ruby
# lib/omniauth_setup.rb
class OmniauthSetup
  # OmniAuth expects the class passed to setup to respond to the #call method.
  # env - Rack environment
  def self.call(env)
    new(env).setup
  end

  # Assign variables and create a request object for use later.
  # env - Rack environment
  def initialize(env)
    @env = env
    @request = ActionDispatch::Request.new(env)
  end

  private

  # The main purpose of this method is to set the consumer key and secret.
  def setup
    @env['omniauth.strategy'].options.merge!(custom_credentials)
  end

  # Use the subdomain in the request to find the account with credentials
  def custom_credentials
    account = Account.find_by!(subdomain: @request.subdomain)
    {
      client_id: account.twitter_consumer_key,
      client_secret: account.twitter_consumer_secret
    }
  end
end
```

Let’s break this class down and see what’s going on:

1. OmniAuth expects our class to respond to `.call` and passes `env` as a parameter.
2. When we instantiate our class we create a request instance from the `env`. This will allow us to get the `#subdomain` later on.
3. The `#setup` method reaches inside the environment hash for `omniauth.strategy` to grab the current options. This would be the values you’ve already set in the omniauth initializer. It then merges in the hash returned from the hash returned by `#custom_credentials`.
4. We obtain the accounts custom credentials by simply finding the account in our database with a matching recording and then building a hash with those values.

So now if you start the OmniAuth sign in process from the accounts subdomain, OmniAuth sets up the oauth handshake to Twitter using the credentials stored against the account record in the database.

While this is a basic example of what you can do with the OmniAuth setup phase, hopefully it will inspire you to creatively exploit the OmniAuth setup phase in your own projects.
