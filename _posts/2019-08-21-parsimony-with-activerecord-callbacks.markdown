---
layout: post
title: Parsimony with ActiveRecord callbacks
tags: [ruby, rails, activerecord]
image: '/images/posts/callbacks.png'
---

It's not rare to see someone associating activerecord callbacks with something bad or wrong, avoiding at all cost using them and thinking in other ways to achieve the desired goal, although i believe that nothing is absolute and there are situations where they are needed.

Taking the following scenario as example

```ruby
class Purchase < ApplicationRecord
  belongs_to :user

  def confirm!(user)
    self.user = user
    self.confirmed = true
    PurchaseProcessor.create!(user: user)
    Save!
  end
end
```

We have the class `Purchase` which have a method` confirm! `Responsible for changing the state of` Purchase`, however this can lead us to a problem of consistency capable of creating a headache because the ActiveRecord automatically generates methods for the attributes and they can be changed in other fluxes

```ruby
purchase.update_attributes!(confirmed: true)
Purchase.new(confirmed: true).save
Purchase.create!(confirmed: true)
```

This is a big problem having in mind that the state of the object is inconsistent, it is confirmed but dont have user and the processor wasnt executed

If exist a worker that generates any important report of purchases with base at a condition setted on `PurchaseProcessor`, the chance to leave your base inconsistency is huge

One option would be using callbacks

```ruby
class Purchase < ApplicationRecord
  belongs_to :user

  validates_presence_of :user_id, if: :confirmed?

  after_save :execute_purchase_processor_on_confirm

  private

  def execute_purchase_processor_on_confirm
    If confirmed? && confirmed_changed?
      PurchaseProcessor.create!(user: user)
    end
  end
end
```

In that way, independently of how any attribute is setted in any flux we have the garantee of the integrity

But there are cases where callbacks became a problem by itself, eg:

```ruby
class User < ApplicationRecord
  after_create :send_confirmation_email

  def send_confirmation_email
    UserMailer.registration_confirmation(self).deliver
  end
end
```

This way maybe its necessary to create an user anywhere without calling the `send_confirmation_email`, and with this design you can just do it by using the private method `create_without_callbacks` who also ignores the validations ( not a good option )

I think it worths the question, how frequently you will have to create an user withotu sending the confirmation email? maybe it deserves to be splited in other class, eg:

```ruby
class UserRegistration
  def initialize(user)
    @user = user
  end

  def register
    @user.save
    UserMailer.registration_confirmation(self).deliver
  end
end
```

Maybe its better to split behaviors that is not expected by default with the goal to simplify the use and manteining ( you expect that creating a `User` just create it without executing other actions or fluxes )

Callbacks are not all evil, but i believe that should had some reflection before using it, thinking about the context and the scenario... but as as rule i particularly like to start thinking about the design without the use of the callbacks and analyse if it can cause some inconsistency or indesired scenario
