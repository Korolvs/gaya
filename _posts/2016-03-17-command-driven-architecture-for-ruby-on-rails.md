---
layout: post
title:  "Command-driven architecture for Ruby on Rails"
date:   2016-03-17 11:00:55
banner_image: iloveruby.jpg
meta_description: "Ruby on Rails is a great framework for a quick start, but when project gets much bigger, tens of files in controllers, models and views directories can become a huge headache for a developer. The architecture described in this article extends a common MVC approach with adding few new primitives to an application. It doesn't break the standart RoR approach, but only extends it."
comments: true
---

Ruby on Rails is a great framework for a quick start, but when project gets much bigger, tens of files in controllers, models and views directories can become a huge headache for a developer (especially if the developer made few bad decisions at the beginning of working on the project). Each edit affecting more than few files becomes a real problem. The best way to solve this problems is to use a good architecture from very start of developing the project.

The architecture described in this article extends a common MVC approach with adding few new primitives to an application. It doesn't break the standart RoR approach, but only extends it.

So main primitives are:

- Commands
- Middleware
- Controllers
- Models
- Views
 
Models and views are the same as in the original RoR architecture, so I am not touching them in this article. Commands are simple actions that don`t know where they will be implemented and where the results go. Controllers used to initialize commands and run them through a middleware chain. And middleware (not to be confused with rake middleware) is where the magic begins. Middleware is a class with a method to do something and call a next middleware. You can do combining and rearranging middleware with any commands you want. The basic example is *Validator* -> *Executor* -> *Renderer*. Want to return JSON or write response to a file? Add custom logger before or after command execution? Or run commands asynchronously? Just add a middleware!

If you want to see an example - [welcome](https://github.com/korolvs/thatsaboy)!

I’m using a project without views as an example and views aren’t described below but you can simply use this architecture for this type of projects as well. So let’s take a closer look.

## Modules

First of all, to avoid making a mess in the project it is needed to divide all files to modules. Each module has a one declaration file  and two own folders - one in controllers and one in models. Module contains all the files related to it. The best way is to make modules highly separated from each over.

## Command

The basic primitive in this architecture is a command. Command is a separated action, for instance creating an object or deleting it. Each command contains a validation part and main part that makes all needed actions. Execute method contains business logic and can return a hash or nothing if it doesn’t needed.

Each command must be inherited from the `Core::Command` class(or its child, of course). Let's take a look at this class.

{% highlight ruby %}
#showme!
# Contains common methods for commands
class Core::Command
  include ActiveModel::Validations

  attr_accessor :token

  # Fils a command with attributes
  # @param [Hash] attributes
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

  # Gets the model to validate
  # @return [Class]
  # @raise Core::Errors::NoModelToValidateError
  def model_to_validate
    fail Core::Errors::NoModelToValidateError
  end
end
{% endhighlight %}

It includes `ActiveModel::Validations` module to do validation and a token attribute that is very common and needed for authorization. Also there are three methods:

- initialize - fills a command with given attributes and makes it very easy to use commands with different input parameters.
- execute - dummy method which must be overridden in every child
- model_to_validate - returns a model needed to some validation like exists validator and unique validator

Here is an example of command:

{% highlight ruby %}
#showme!
# Deletes a goal command
class Goal::Command::GoalDeleteCommand < Core::Command
  attr_accessor :id

  validates :id, presence: true,
                 'Core::Validator::Exists' => ->(x) { { id: x.id, deleted_at: nil } }

  # Runs command
  # @return [Hash]
  def execute
    goal = Goal::Goal.where(id: id).first
    goal.deleted_at = DateTime.now.utc
    goal.save
    nil
  end

  # Gets the model to validate
  # @return [Class]
  def model_to_validate
    Goal::Goal
  end
end
{% endhighlight %}

### Validation

In the original RoR architecture there is only a model validation, so why do we need to use an action validation? With this approach you can easily check all input parameters and return to a user what is exactly wrong before running the main code of a command . After this you don’t need to catch any validation errors in the code and it becomes much more clear. Also you can add a model validation.

In commands you can use the same validators like in models and write your own. For instance, I wrote four common validators:

- Exists
- Unique
- Content Type
- Uri

Exists and unique validators use `model_to_validate` method and have as a parameter a lamba function to use the result in the query

{% highlight ruby %}
#showme!
  validates :id, presence: true,
                 'Core::Validator::Exists' => ->(x) { { id: x.id, deleted_at: nil } }
{% endhighlight %}

## Controller

`Core::Controller` has two methods:

- middleware_list - to get the list of needed middleware
- run - to run a command through this middleware

{% highlight ruby %}
# Common controller methods
class Core::Controller < ActionController::Base
  # Runs the action
  # @param [Core::Command] command
  # @see Core::CommandBus
  def run(command)
    Core::CommandBus.new(middleware_list).run(command)
  end

  # Returns a list of middleware needed to process commands
  # @return [Array]
  # @see Core::Middleware
  # @see Core::Middleware::ErrorRenderer
  # @see Core::Middleware::Renderer
  # @see Core::Middleware::AuthorizationChecker
  # @see Core::Middleware::ValidationChecker
  # @see Core::Middleware::OwnerChecker
  # @see Core::Middleware::Executor
  def middleware_list
    [
        Core::Middleware::ErrorRenderer.new(self),
        Core::Middleware::Renderer.new(self),
        Core::Middleware::AuthorizationChecker.new,
        Core::Middleware::ValidationChecker.new,
        Core::Middleware::OwnerChecker.new,
        Core::Middleware::Executor.new
    ]
  end
end
{% endhighlight %}

As all logic is in commands, in controllers you need just to initialize command and run it. This is how it looks like:

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

## Middleware

Middleware is a class with a `call` method, which is responsible for making all needed actions with a command (or without it). Also there is a protected `next` method to call the next middleware in a chain. Method `next` can be called after, before or even between main actions in the `call` method and must returns the command and the result of its actions. 

The default list of middleware in a controller is:

- Core::Middleware::ErrorRenderer
- Core::Middleware::Renderer
- Core::Middleware::AuthorizationChecker
- Core::Middleware::ValidationChecker
- Core::Middleware::OwnerChecker
- Core::Middleware::Executor
 
Let's look on some examples:

### ErrorRenderer

{% highlight ruby %}
# Error handler that render errors
class Core::Middleware::ErrorRenderer < Core::Middleware
  attr_accessor :controller

  # Sets a controller to render
  # @param [Core::Controller] controller
  def initialize(controller)
    self.controller = controller
  end

  # Handle errors and process them
  # @return [[Core::Command], [Object]]
  def call
    return self.next
  rescue Core::Errors::UnauthorizedError
    controller.render json: { error: 'Unauthorized' }, status: 401
  rescue Core::Errors::ForbiddenError
    controller.render json: { error: 'Forbidden' }, status: 403
  rescue Core::Errors::ValidationError => e
    controller.render json: e.command.errors, status: 422
  rescue StandardError => e
    controller.render json: { error: e.message }, status: 500
  end
end
{% endhighlight %}

This middleware calls the next middleware and catches all errors from the chain. It needs a controller to render errors. You can write your own error handler which will do some other actions with errors.

### AuthorizationChecker

This middleware checks that the right token is given. There are different types of tokens for different commands. `AuthorizationChecker` reads `authorization_rules.yml` file and makes the decision.

{% highlight yaml %}
#showme!
test:
  # User
  - action: User::Command::LogoutCommand
  - action: User::Command::ConfirmCommand
    token: :confirmation
  - action: User::Command::RecoveryCommand
    token: :recovery
  - action: User::Command::PinSetCommand
  - action: User::Command::PinCheckCommand
  - action: User::Command::ViewCommand
  # File
  - action: Uploaded::Command::PhotoCreateCommand
  - action: Uploaded::Command::PhotoIndexCommand
  - action: Uploaded::Command::PhotoDeleteCommand
    owner: true
    ....
{% endhighlight %}

The `owner` flag in this file is used as a signal for `OwnerChecker` to show that it is needed. With small changes this authorization system can become a full role-based access control system.

### Renderer

{% highlight ruby %}
# Renderer
class Core::Middleware::Renderer < Core::Middleware
  attr_accessor :controller

  # Sets a controller to render
  # @param [Core::Controller] controller
  def initialize(controller)
    self.controller = controller
  end

  # Renders response
  # @return [[Core::Command], [Object]]
  def call
    command, result = self.next
    if result.nil?
      controller.render json: nil, status: 204
    else
      controller.render json: result, status: 200
    end
    return command, result
  end
end
{% endhighlight %}

### ValidationChecker

{% highlight ruby %}
# Default validation checker
class Core::Middleware::ValidationChecker < Core::Middleware
  # Checks that command is valid
  # @return [[Core::Command], [Object]]
  # @raise Core::Errors::ValidationError
  def call
    fail(Core::Errors::ValidationError, command) if command.invalid?
    self.next
  end
end
{% endhighlight %}

### OwnerChecker

{% highlight ruby %}
# Class that checks that user is the owner
class Core::Middleware::OwnerChecker < Core::Middleware
  include Core::Authorization

  # Checks that user is the owner
  # @return [[Core::Command], [Object]]
  def call
    owner?
    self.next
  end

  private

  # Checks all tests
  # @raise Core::Errors::ForbiddenError
  def owner?
    rule = get_rule command
    return if rule.blank?
    return unless get_owner_param rule
    token = get_token command
    user = User::User.get_user_by_token_code(token.code, token.token_type)
    model = command.model_to_validate
    fail Core::Errors::ForbiddenError if model.where(user_id: user.id).first.nil?
  end
end
{% endhighlight %}

### Executor

{% highlight ruby %}
# Default executor for commands
class Core::Middleware::Executor < Core::Middleware
  # Runs a command
  # @return [[Core::Command], [Object]]
  def call
    return command, command.execute
  end
end
{% endhighlight %}



# Conslusion

As you can see, this approach makes code clear and flexible. Each command contains 5-10 lines of code for one certain purpose. These commands can be executed in different parts of an application such as a controller or some rake task. Moreover, middleware is a simple unit responsible for one action with a command. Just by adding, removing or rearranging middleware you can run your commands in many different ways.

Check the example [here](https://github.com/korolvs/thatsaboy).
