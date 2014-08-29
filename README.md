Canner
======

Canner is an authorization gem heavily modeled after Pundit.  

Canner's intention is to provide you a framework for authorization that has little to no magic.  
Your canner policies can be as simple or as complicated as your app requires.

Who needs another auth gem?  There's a bunch of very good ones out there.  
Pundit, cancan, cancancan and Declarative Authorization to name a few alternatives.  

Unfortunately for me, none of those solutions had built in support for a requirement I have on the
project [Kennel Captain](http://www.kennelcaptain.com)  

We need to authorize a user at a given store location, and that's a feature canner will support.

For details see the wiki page [Authorize with Branches ( Store Locations )](https://github.com/jacklin10/canner/wiki/Authorize-with-Branches)

Also note that canner works fine if you don't need this particular feature, its just there if you do.

## Installation

I've only tested Canner on rails 4 but it should work find in rails 3.2 apps.

To install Canner just put the following in your gem file.
``` ruby
gem "canner"
```

Then run

``` ruby
bundle install
```

Now include canner in your application_controller.rb

``` ruby
include Canner
```

You'll then need to create a Policy class for the models you'd like to authorize.

``` ruby
rails g canner:policy user
```

If your app gets roles from a user in a way other than @current_user.roles then you'll
need to override the fetch_roles policy method.

```ruby
rails g canner:fetch_roles
```

More details are available in the wiki: 
[Overriding the Fetching of Roles](https://github.com/jacklin10/canner/wiki/Feed-Roles)

## Policies

As mentioned Canner is strongly influenced by Pundit and is also based on Policy objects.
Your policy objects should be named using the following pattern:   
UserPolicy, CustomerPolicy, AppPolicy.

Use the generator to save you some time: 
``` rails g canner:policy <model name> ```

Your policy models need to implement 2 methods:
``` ruby
def canner_scope
end

def can?
end
```

### canner_scope

You'll want to implement this method for each of your model policies that extend from base_policy.rb.

The canner_scope method is used to scope the authorized models consistently in your app.

For example in my app the Customers controller uses the canner_scope to
ensure only Users from the current_company are displayed.

``` ruby
class CustomersController < ApplicationController
  respond_to :html, :json
  before_action :authenticate_user!

  def index
    @customers = canner_scope(:index, :customer)

    can?(:index, :customer)
  end
end
```

and the policy is:

``` ruby
class CustomerPolicy < BasePolicy

  def canner_scope
    case @method
    when :index
      User.where(company_id: @current_branch.company.id)
    else
      User.none
    end
  end

  def can?
    case @method
    when :new, :index, :create, :update, :edit
      has_role?(:admin)
    else
      false
    end
  end

end
```

Now you don't really need to think about the auth logic when fetching a list of customers.
Just make sure you use the policy and you'll only show the users what is intended.

Also if your policy changes at some point its a one place fix.

### can?

You use the can method to determine if the current_user is able to access an action or resource.

The example above uses a straightforward case statement to determine if the current_user can
access the current action or resource.

The symbols in the when portion of the case match your typical actions in the example but they
can be whatever you want really.

``` ruby
case @method
when :something_random
  has_role?(:admin)
else
  false
end
```
# Then in controller do:
```can?(:something_random, :customer)```

In english the can method is saying:

Can the currently signed in user access the something_random action?  Oh, and by the way please
use the CustomerPolicy's can? method to do the checking.

`can?(:something_random, :user)` would use the ... you guessed it UserPolicy's can? method.

If you want to deny access by default across all model policies you could do something as simple as:

``` ruby
def can?
  false
end
```

in your base_policy's `can?` method

### Force the Use of Policies

Also like Pundit you can force your app to use policies.
I recommend you do this so you don't forget to wrap authorization about some of your resources.

To make sure your controller actions are using the can? method add this near the top of your
application_controller.rb

``` ruby
after_action :ensure_auth
```

And to make sure you are using the canner_scope do the following:
``` ruby
after_action :ensure_scope, only: :index
```

Note the use of only here.  You usually won't need the canner_scope on anything except
for the index to be strictly enforced.

If you would like to skip for a particular controller just add
``` ruby
skip_filter :ensure_scope
```
And / Or
``` ruby
skip_filter :ensure_auth
```

### Handle Canner Authorization Failures

When a user does stumble onto something they don't have access to you'll want to politely
tell them about it and direct the app flow as you see fit.

To accomplish this in your application_controller.rb add

``` ruby
rescue_from Canner::NotAuthorizedError, with: :user_not_authorized
```

You can name your method whatever you want.  Mine is user_not_authorized and looks like this:

``` ruby
private

def user_not_authorized(exception)
  flash[:error] = exception.message
  redirect_to(request.referrer || root_path)
end
```

### Using can? in views

You'll likely want to show or hide on screen items based on a users' role.
This is done in canner like this:

``` ruby
  = link_to 'Create Customer', new_customer_path if canner_policy(:new, :customer).can?
```

This will look in the CustomerPolicy's can? method implemention and follow whatever rules
you have for the :new symbol.

So assuming the CustomerPolicy can? method provided below the currently signed in user
would only be able to see the create customer link if they had an admin role.

``` ruby
  def can?
    case @method
    when :new
      has_role?(:admin)
    else
      false
    end
  end
```

## Testing

See the wiki for some testing tips
[Testing](https://github.com/jacklin10/canner/wiki/Testing)
