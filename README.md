# Pretender

As an admin, there are times you want to see exactly what another user sees. Meet Pretender.

- Easily switch between users
- Minimal code changes
- Plays nicely with Action Cable and auditing tools

:boom: [Rock on](https://www.youtube.com/watch?v=SBjQ9tuuTJQ)

Pretender is flexible and lightweight - less than 100 lines of code :-)

Works with any authentication system - [Devise](https://github.com/plataformatec/devise), [Authlogic](https://github.com/binarylogic/authlogic), and [Sorcery](https://github.com/Sorcery/sorcery) to name a few.

:tangerine: Battle-tested at [Instacart](https://www.instacart.com/opensource)

[![Build Status](https://github.com/enercoop/pretendest/actions/workflows/build.yml/badge.svg)](https://github.com/enercoop/pretendest/actions)

## Pretendest

Pretendest is a friendly fork of [Pretender](https://github.com/ankane/pretender), with a handful of extra features:
- An `#impersonating_user?` helper,
- An option to impersonate a resource by another resource (e.g. a Client impersonated by an Employee).

If you don't need those features, the original Pretender gem is perfectly fine.
Otherwise, `pretendest` is a drop-in replacement for `pretender`.

## Installation

Add this line to your application’s Gemfile:

```ruby
gem "pretendest"
```

And add this to your `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  impersonates :user
end
```

## How It Works

Sign in as another user with:

```
impersonate_user(user)
```

The `current_user` method now returns the impersonated user.

You can access the true user with:

```
true_user
```

And stop impersonating with:

```ruby
stop_impersonating_user
```

### Sample Implementation

Create a controller

```ruby
class UsersController < ApplicationController
  before_action :require_admin! # your authorization method

  def index
    @users = User.order(:id)
  end

  def impersonate
    user = User.find(params[:id])
    impersonate_user(user)
    redirect_to root_path
  end

  def stop_impersonating
    stop_impersonating_user
    redirect_to root_path
  end
end
```

Add routes

```ruby
resources :users, only: [:index] do
  post :impersonate, on: :member
  post :stop_impersonating, on: :collection
end
```

Create an index view

```erb
<ul>
  <% @users.each do |user| %>
    <li>Sign in as <%= button_to user.name, impersonate_user_path(user), data: {turbo: false} %></li>
  <% end %>
</ul>
```

And show when someone is signed in as another user in your application layout

```erb
<% if impersonating_user? %>
  You (<%= true_user.name %>) are signed in as <%= current_user.name %>
  <%= button_to "Back to admin", stop_impersonating_users_path, data: {turbo: false} %>
<% end %>
```

## Audits

If you keep audit logs with a library like [Audited](https://github.com/collectiveidea/audited), make sure it uses the **true user**.

```ruby
Audited.current_user_method = :true_user
```

## Action Cable

And add this to your `ApplicationCable::Connection`:

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user, :true_user
    impersonates :user

    def connect
      self.current_user = find_verified_user
      reject_unauthorized_connection unless current_user
    end

    private

    def find_verified_user
      env["warden"].user # for Devise
    end
  end
end
```

The `current_user` method now returns the impersonated user in channels.

## Configuration

Pretendest is super flexible. You can change the names of methods and even impersonate multiple roles at the same time. Here’s the default configuration.

```ruby
impersonates :user,
             method: :current_user,
             with: ->(id) { User.find_by(id: id) }
```

Mold it to fit your application.

```ruby
impersonates :account,
             method: :authenticated_account,
             with: ->(id) { EnterpriseAccount.find_by(id: id) }
```

This creates five methods:

```ruby
true_account
impersonate_account
stop_impersonating_account
impersonating_account? # Pretendest addition
account_impersonator   # Pretendest addition
```

### Impersonating from another role (Pretendest addition)

You can impersonate a role from another role. Consider for instance when Employees are allowed to impersonate Clients:

```ruby
impersonates :client,
             impersonator: :employee
```

In that case, when impersonating:
- `true_client` returns `nil` (because there is no "true" client),
- `client_impersonator` returns the `current_employee`. 

For this to work, the class needs to provide both `current_client` and `current_employee` methods.

If your code base is using Devise for authentication, you can simply configure Devise to allow either Clients or Employees to access authenticated pages:

```ruby
  # Allow authenticating as either:
  # - a client (usual case)
  # - an employee (when impersonating a client)
  devise_group :allowee, contains: [:client, :employee]

  before_action :authenticate_allowee!
  before_action :reject_employee_without_impersonation!

  # Only employees currently impersonating a client are allowed to get through.
  def reject_employee_without_impersonation!
    authenticate_client! unless impersonating_client?
  end
```

## History

View the [changelog](https://github.com/enercoop/pretendest/blob/master/CHANGELOG.md)

## Contributing

Everyone is encouraged to help improve this project. Here are a few ways you can help:

- [Report bugs](https://github.com/enercoop/pretendest/issues)
- Fix bugs and [submit pull requests](https://github.com/enercoop/pretendest/pulls)
- Write, clarify, or fix documentation
- Suggest or add new features

To get started with development:

```sh
git clone https://github.com/enercoop/pretendest.git
cd pretendest
bundle install
bundle exec rake test
```
