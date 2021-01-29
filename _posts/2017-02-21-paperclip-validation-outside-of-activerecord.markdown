---
layout: post
title:  "Paperclip Validation outside of ActiveRecord"
date:   2017-02-21 09:00:00 +0100
author: Bishwa Hang Rai
categories: programming
---

In this post we will try to give an example to perform the active record validation outside the ActiveRecord models.
Why would someone even want to do that ?
At [Freeletics](https://freeletics.com){:target="_blank"}, we have moved past the principle of **Fat Model, Thin Controller** and follow **Thin Model, Thin Controller**, supported by **Interactors, Services, and Presenters**.
To give a quick overview, we have different services such as:
  * payment (payment providers such as Apple, Google, Stripe)
  * users (everything related to our Users)
  * bodyweight (one of our first product that makes you do push ups, pull ups, and **BURPEES**)

These (micro)services communicate with each other and the Clients, using JSON API.
Our approach is to make Models and Controllers as thin as possible. Most of the required Business Logic is done by the **"Interactors"**, along with the support from Service Objects, whenever needed, to perform the tasks. The Controller action will simply call the responsible Interactor and expect `true` or `false` in return. Based on the result, the Controller will render appropriate response with the help of **Presenter**.

This makes our app modular, less coupled and easy to extend/update.

If this simplified overview has intrigued you, we will give a detailed explanation about our architecture in our future posts.

Let's come back to the main idea of this post. As we mentioned earlier, we make interactors do all the tasks including _ActiveRecord_ validations; the Model class will only have definition of model structure and its associations.
In this example, we start with the basic Paperclip procedure to define an attached file in our User model.
```ruby
class User < ActiveRecord::Base
  has_attached_file :image,
    styles: {
      small:  {geometry: "375x150",  format: :jpg},
      medium: {geometry: "750x300",  format: :jpg},
      large:  {geometry: "1125x450", format: :jpg}
    },
    convert_options: {
      small:   "-strip",
      medium:  "-strip",
      orginal: "-strip"
    }

  do_not_validate_attachment_file_type :image
```
We tell explicitly that we do not want paperclip to validate the attachment file type - `Paperclip` throws an error if you don't validate the attachment - as we do that in the Interactor later.

Now we define an Interactor that is responsible to operate upon the User model.

```ruby
class SaveUserInteractor
  include ActiveModel::Model

  ATTRIBUTES = %i(first_name last_name image)

  attr_accessor(*ATTRIBUTES)

  validates :first_name, presence: true
  validates :last_name, presence: true

  # >>> all this is needed for paperclip validation outside of AR
  extend ActiveModel::Callbacks
  include Paperclip::Glue

  define_model_callbacks :save, only: [:after]
  define_model_callbacks :destroy, only: [:before, :after]
  define_model_callbacks :commit, only: [:after]

  has_attached_file :image
  validates_attachment :image, content_type: {content_type: %r{\Aimage/.*\Z}}, size: {in: 0..10.megabytes}
  attr_accessor :image_file_size, :image_file_name, :image_content_type, :id
  # <<< end of paperclip validation

  def initialize(user, attributes = {})
    super(attributes)
  end

  def call
    valid? && persist
  end

  private

  def persist
    # save/update the model
  end
```

The `ActiveModel::Model` module inclusion is self explanatory, for validations and initialization of User attributes.
Paperclip validation is done between the comment block section. `Paperclip::Glue` is included and also Model callbacks are defined to support Paperclip with its internal validation implementation.
That's it, now we are good to create a Controller that uses this Interactor to talk with User model.

The code for Controller could be something like this:

```ruby
class UserController < Admin::ApplicationController
  def create
    interactor = SaveUserInteractor.new(
      User.new,
      user_params
    )
    if interactor.call
      # return success response;
      # Presenter could be called to modify the response
    else
      # return failure response
    end
  end

  private

  def user_params
    params.require(:user).permit(:first_name, :last_name, :image)
  end
end
```

This is all you need to do to make the Paperclip attachment validation outside of ActiveRecord Model.

Things to remember in the *Interactor* class are:
 * Keep Paperclip validation after all other validations, i.e, at last, to avoid ```:flush_errors```. [Issue here](https://github.com/thoughtbot/paperclip/issues/1368#issuecomment-42587052){:target="_blank"}
 * The ```attr_accessor: :id``` should be there, else it throws `NoMethodError`, when you try to update the attachment. If you want to dig deeper, exactly [here](https://github.com/thoughtbot/paperclip/blob/master/lib/paperclip/interpolations.rb#L149-L151){:target="_blank"}.

With this pattern, our Controller should only know which Interactor to call. The interactor call returns either `true` or `false` based on the operation being succesfull or not. And the Model class remains as pure Database entity. There is no coupling between **Controller** and **Model**, and the Business Logic can be distributed between **Interactor** and **Service** classes.

Tested on `Paperclip >= 5.0.0` and `rails >= 4.2.5`.
