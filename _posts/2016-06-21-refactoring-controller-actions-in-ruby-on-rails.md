---
layout: post
title:  "Refactoring controller actions in Ruby on Rails"
date:   2016-06-21 11:00:55
banner_image: controller_refactoring.jpg
meta_description: "Thin controllers, fat models - one of the base principles of MVC in Ruby on Rails, but time goes by, your project is getting bigger, the number of methods is growing. It's becoming hard to find needed part of the code and add new features. Thin controllers are not really so thin anymore. Let's clean up this mess!"
comments: true
---

Thin controllers, fat models - one of the base principles of MVC in Ruby on Rails, but time goes by, your project is getting bigger, the number of methods is growing. It's becoming hard to find needed part of code and add new features. Thin controllers are not really so thin anymore. Let's clean up this mess!

## Controllers

First of all, we don't want to have controllers with hundreds of lines of the code. The best way to solve it is to create more controllers with difficult responsibilities. Also, I prefer to create an individual class for each action. This classes called **Commands**. It makes each controller looks like this:

{% highlight ruby %}
#showme!
# Goal controller
class Goal::GoalController < Core::Controller
  # Shows a goal
  # @see Goal::Command::GoalViewCommand
  def view
    command = Goal::Command::GoalViewCommand.new(params)
    run(command)
  end

  # Updates a goal
  # @see Goal::Command::GoalUpdateCommand
  def update
    command = Goal::Command::GoalUpdateCommand.new(params)
    run(command)
  end

  # Deletes a goal
  # @see Goal::Command::GoalDeleteCommand
  def delete
    command = Goal::Command::GoalDeleteCommand.new(params)
    run(command)
  end

  # Adds or removes points
  # @see Goal::Command::GoalPointsUpdateCommand
  def points_update
    command = Goal::Command::GoalPointsUpdateCommand.new(params)
    run(command)
  end
end
{% endhighlight %}

## Command structure

Now let's take a look at common parts of each action(all of them are optional):

 - Authorization checks
 - Validation input params
 - Main logic
 - Render
 - Redirecting or rendering some messages in case of errors

We need to separate this parts in our commands to make them easy and clean. But before I'll show you the *run* method of **Core::Controller**:

{% highlight ruby %}
#showme!
# Common controller methods
class Core::Controller < ActionController::Base
  around_action Core::Filter::ErrorRenderer

  # Runs the action
  # @param [Core::Command] command
  # @see Core::FilterChain
  def run(command)
    command.check_authorization
    command.check_validation
    result = command.execute
    if result.nil?
      render json: nil, status: 204
    else
      render json: result, status: 200
    end
  end
end
{% endhighlight %}

This method handle all described parts of an action.

At first, it checks authorization and validation with *command.check_authorization*, then runs the command with *command.execute* and renders the result(json in this case).
Finally, action is wrapped with *around_action* class **Core::Filter::ErrorRenderer**:

{% highlight ruby %}
# Error handler that render errors
class Core::Filter::ErrorRenderer

  class << self
    # Handle errors and process them
    # @param [Core::Controller] controller
    def around(controller)
      @controller = controller
      yield
    rescue Core::Errors::UnauthorizedError
      self.render 401, JSON.generate(:error => 'Unauthorized')
    rescue Core::Errors::ForbiddenError
      self.render 403, JSON.generate(:error => 'Forbidden')
    rescue Core::Errors::ValidationError => e
      self.render 422, e.command.errors.to_json
    rescue StandardError => e
      self.render 500, JSON.generate({ error: e.message, backtrace: e.backtrace })
    end

    # Handle errors and process them
    # @param [Integer] status
    # @param [Hash] json
    def render(status, json)
      @controller.response.status = status
      @controller.response.body = json
    end
  end
end
{% endhighlight %}

## Command class

Thus, we need to create *check_authorization*, *check_validation* and *execute* methods in **Core::Command** class.

### Authorization

You can use your favorite gem or write your own code for this. In projects, where authorization logic is simple, I prefer not to use gems for this.

Let's create a method with rules:

{% highlight ruby %}
#showme!
# Rules for authorization
# @return [Hash]
def authorization_rules
  { token_type: :login }
end
{% endhighlight %}

and check, that everything is alright:

{% highlight ruby %}
#showme!
# Checks that command can be executed by the user
def check_authorization
  User::Service::AuthorizationService.get.get_token_by_command self
end
{% endhighlight %}

### Validation

In the original RoR architecture there is only a model validation, so why do we need to use an action validation? With this approach, you can easily check all input parameters and return to a user what is exactly wrong before running the main code of a command . After this, you don’t need to catch any validation errors in the code and it becomes much clearer. Also, you can add a model validation.

Include `ActiveModel::Validations` module to do validation. You can use the same validators like in models and write your own. For instance, I wrote five common validators:

- Exists
- Unique
- Content Type
- Owner
- Uri

Exists, unique and owner validators have a lambda function as a parameter to use the result and check it.

{% highlight ruby %}
#showme!
  validates :id, presence: true,
                 'Core::Validator::Exists' => ->(x) { x.goal_repository.find_not_deleted(x.id) }
{% endhighlight %}

### Execute

Each command must have *execute* method, where the logic should be placed.

Finally, we have:

{% highlight ruby %}
#showme!
# Contains common methods for commands
class Core::Command
  include ActiveModel::Validations

  attr_accessor :token

  # Fills a command with attributes
  # @param [Hash] attributes
  # @raise Core::Errors::ValidationError
  def initialize(attributes = {})
    attributes.each do |name, value|
      if methods.include? "#{name}=".to_sym
        method("#{name}=".to_sym).call(value)
      end
    end
  end

  # Runs command
  def execute
  end

  # Rules for authorization
  # @return [Hash]
  def authorization_rules
    { token_type: :login }
  end

  # Checks that command can be executed by the user
  def check_authorization
    User::Service::AuthorizationService.get.get_token_by_command self
  end

  # Checks that all params are correct
  def check_validation
    raise(Core::Errors::ValidationError, self) if self.invalid?
  end
end
{% endhighlight %}

## Examples

Now, when we know how commands should be written, let's look at some examples

{% highlight ruby %}
#showme!
# Update family command
class Family::Command::FamilyUpdateCommand < Core::Command
  attr_accessor :name, :photo_url

  validates :name,      length: { maximum: 50 }
  validates :photo_url, length: { maximum: 100 }
  validates :photo_url, 'Core::Validator::Uri' => true

  # Sets all variables
  # @param [Object] params
  # @see User::Service::AuthorizationService
  # @see Family::Repository::FamilyRepository
  def initialize(params)
    super(params)
    @authorization_service = User::Service::AuthorizationService.get
    @family_repository = Family::Repository::FamilyRepository.get
  end

  # Runs command
  def execute
    user = @authorization_service.get_user_by_token_code(token)
    family = user.family
    family.name = name unless name.nil?
    family.photo_url = photo_url unless photo_url.nil?
    @family_repository.save!(family)
    nil
  end
end
{% endhighlight %}

{% highlight ruby %}
#showme!
# Delete a goal command
class Goal::Command::GoalDeleteCommand < Core::Command
  attr_accessor :id
  attr_accessor :goal_repository

  validates :id, presence: true,
                 'Core::Validator::Exists' => ->(x) { x.goal_repository.find_not_deleted(x.id) }
  validates :id, 'Core::Validator::Owner' => ->(x) { x.goal_repository.find(x.id) }

  # Sets all variables
  # @param [Object] params
  # @see Goal::Repository::GoalRepository
  def initialize(params)
    super(params)
    @goal_repository = Goal::Repository::GoalRepository.get
  end

  # Runs command
  # @return [Hash]
  def execute
    goal = @goal_repository.find(id)
    @goal_repository.delete(goal)
    nil
  end
end
{% endhighlight %}

{% highlight ruby %}
#showme!
# View a family command
class Family::Command::FamilyViewCommand < Core::Command
  # Sets all variables
  # @param [Object] params
  # @see User::Service::AuthorizationService
  # @see Family::Presenter::FamilyPresenter
  def initialize(params)
    super(params)
    @authorization_service = User::Service::AuthorizationService.get
    @family_presenter = Family::Presenter::FamilyPresenter.get
  end

  # Runs command
  # @return [Hash]
  def execute
    user = @authorization_service.get_user_by_token_code(token)
    family = user.family
    @family_presenter.family_to_hash(family)
  end
end
{% endhighlight %}

## Conclusion

As you can see, this approach makes code clear and flexible. Each command contains 5-10 lines of code for one certain purpose. These commands can be executed in different parts of an application such as a controller or some rake task. Your controllers are simple and clean.

Check the example [here](https://github.com/korolvs/thatsaboy).

Also, you can check [different architecture for a rails application]({% post_url 2016-03-17-command-driven-architecture-for-ruby-on-rails %}).

And read about [the domain layer]({% post_url 2016-05-08-domain-driven-design-for-rails %}).

<a href="http://www.freepik.com/free-photos-vectors/background">Background vector designed by Freepik</a>
