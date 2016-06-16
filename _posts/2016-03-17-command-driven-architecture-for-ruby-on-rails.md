---
layout: post
title:  "Command-driven architecture for Ruby on Rails"
date:   2016-03-17 11:00:55
banner_image: command.jpg
meta_description: "Ruby on Rails is a great framework for a quick start, but when project gets much bigger, tens of files in controllers, models and views directories can become a huge headache for a developer. The architecture described in this article extends a common MVC approach with adding few new primitives to an application. It doesn't break the standart RoR approach, but only extends it."
comments: true
---

Ruby on Rails is a great framework for a quick start, but when project gets much bigger, tens of files in controllers, models and views directories can become a huge headache for a developer (especially if the developer made few bad decisions at the beginning of working on the project). Each edit affecting more than few files becomes a real problem. The best way to solve this problems is to use a good architecture from very start of developing the project.

The architecture described in this article extends the Controller part of MVC approach with adding few new primitives to an application. It doesn't break the standart RoR approach, but only extends it.

## Why should you read it?

You should read it, because using such approach, you can achieve greater flexibility and ease of maintenance, when your project become bigger.

## Let's begin

There are commands, controllers and filters in this architecture.

Commands are simple classes with only one certain purpose. Controllers used to initialize commands and run them through a filters chain. Filter is a class with a method to do something and call a next filter. You can combine and rearrange filters as you like. The basic example is *Validator* -> *Executor* -> *Renderer*. Want to return JSON or write response to a file? Add custom logger before or after command execution? Or run commands asynchronously? Just add a filter!

If you want to see an example - [welcome](https://github.com/korolvs/thatsaboy/tree/cda-ddd)!

So let’s take a closer look.

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
end
{% endhighlight %}

It includes `ActiveModel::Validations` module to do validation and a token attribute that is very common and needed for authorization. Also there are two methods:

- initialize - fills a command with given attributes and makes it very easy to use commands with different input parameters. Also here should be initialized all services and models used in the command
- execute - dummy method which must be overridden in every child

Here is an example of command:

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
    @goal_repository = Goal::Repository::GoalRepository.new
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

### Validation

In the original RoR architecture there is only a model validation, so why do we need to use an action validation? With this approach you can easily check all input parameters and return to a user what is exactly wrong before running the main code of a command . After this you don’t need to catch any validation errors in the code and it becomes much more clear. Also you can add a model validation.

In commands you can use the same validators like in models and write your own. For instance, I wrote five common validators:

- Exists
- Unique
- Content Type
- Owner
- Uri

Exists and unique validators have as a parameter a lamba function to use the result and check it.

{% highlight ruby %}
#showme!
  validates :id, presence: true,
                 'Core::Validator::Exists' => ->(x) { x.goal_repository.find_not_deleted(x.id) }
{% endhighlight %}

## Controller

`Core::Controller` has two methods:

- filters_list - to get the list of needed filters
- run - to run a command through this filters

{% highlight ruby %}
# Common controller methods
class Core::Controller < ActionController::Base
  # Runs the action
  # @param [Core::Command] command
  # @see Core::CommandBus
  def run(command)
    Core::CommandBus.new(filters_list).run(command)
  end

  # Returns a list of filters needed to process commands
  # @return [Array]
  # @see Core::Filter
  # @see Core::Filter::ErrorRenderer
  # @see Core::Filter::Renderer
  # @see Core::Filter::AuthorizationChecker
  # @see Core::Filter::ValidationChecker
  # @see Core::Filter::Executor
  def filters_list
    [
        Core::Filter::ErrorRenderer.new(self),
        Core::Filter::Renderer.new(self),
        Core::Filter::AuthorizationChecker.new,
        Core::Filter::ValidationChecker.new,
        Core::Filter::Executor.new
    ]
  end
end
{% endhighlight %}

As all business logic is in commands(and services), in controllers you need just to initialize command and run it. This is how it looks like:

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

## Filters

Filter is a class with a `call` method, which is responsible for making all needed actions with a command (or without it). Also there is a protected `next` method to call the next filter in a chain. Method `next` can be called after, before or even between main actions in the `call` method and must returns the command and the result of its actions.

The default list of filters in a controller is:

- Core::Filter::ErrorRenderer
- Core::Filter::Renderer
- Core::Filter::AuthorizationChecker
- Core::Filter::ValidationChecker
- Core::Filter::Executor
 
Let's look on some examples:

### ErrorRenderer

{% highlight ruby %}
# Error handler that render errors
class Core::Filter::ErrorRenderer < Core::Filter
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
    controller.render json: { error: e.message, backtrace: e.backtrace }, status: 500
  end
end
{% endhighlight %}

This filter calls the next filter and catches all errors from the chain. It needs a controller to render errors. You can write your own error handler which will do some other actions with errors.

### AuthorizationChecker

This filter checks that the right token is given. There are different types of tokens for different commands. `AuthorizationChecker` reads `authorization_rules.yml` file and makes the decision.

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
    ....
{% endhighlight %}

### Renderer

{% highlight ruby %}
# Renderer
class Core::Filter::Renderer < Core::Filter
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
class Core::Filter::ValidationChecker < Core::Filter
  # Checks that command is valid
  # @return [[Core::Command], [Object]]
  # @raise Core::Errors::ValidationError
  def call
    raise(Core::Errors::ValidationError, command) if command.invalid?
    self.next
  end
end
{% endhighlight %}

### Executor

{% highlight ruby %}
# Default executor for commands
class Core::Filter::Executor < Core::Filter
  # Runs a command
  # @return [[Core::Command], [Object]]
  def call
    command, command.execute
  end
end
{% endhighlight %}



# Conslusion

As you can see, this approach makes code clear and flexible. Each command contains 5-10 lines of code for one certain purpose. These commands can be executed in different parts of an application such as a controller or some rake task. Moreover, filter is a simple unit responsible for one action with a command. Just by adding, removing or rearranging filters you can run your commands in many different ways.

Check the example [here](https://github.com/korolvs/thatsaboy/tree/cda-ddd).

And read about [the domain layer]({% post_url 2016-05-08-domain-driven-design-for-rails %}).
