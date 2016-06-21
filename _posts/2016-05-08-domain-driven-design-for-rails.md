---
layout: post
title:  "Domain-driven design for Ruby on Rails"
date:   2016-05-08 11:00:55
banner_image: ddd.jpg
meta_description: "Ordinary models are great for small projects, but when you have a large and frequently changing code, it is very important to have the well-organized architecture to know exactly, where you need to find, add and use needed features."
comments: true
---

Ordinary models are great for small projects, but when you have a large and frequently changing code, it is very important to have the well-organized architecture to know exactly, where you need to find, add and use needed features.

Also, it is important to keep all logic applies to models in the domain layer, because if you need to change some model behavior, you will change it only in one place.

So main parts are:

- Services
- Repositories
- Presenters
- Entities

## Entities

Entities are plain old ruby classes with attributes, setters, getters and methods apply only to this entities. All logic applies to models might be placed in this classes(and services for methods that use several models). They have no need to know anything about how they will be saved in a database. I use ActiveRecord models as entities. It's actually not right, but it's very easy to change them to plain classes if it would be needed.

Here it is an example:

{% highlight ruby %}
#showme!
# Goal class
# Fields:
#  ...
class Goal::Goal < ActiveRecord::Base
  extend Core::Deletable

  belongs_to :user, inverse_of: :goals, class_name: 'User::User'
  belongs_to :child, inverse_of: :goals, class_name: 'Family::Child'
  has_many   :actions, class_name: 'Goal::Action'

  validates :name, presence: true, allow_blank: true, length: { maximum: 50 }
  ...

  # Creates a goal
  # @param [User::User]    user
  # @param [Family::Child] child
  # @param [Integer]       target
  # @param [String]        name
  def initialize(user, child, target, name, photo_url)
    super()
    self.user = user
    self.child = child
    self.target = target
    self.name = name
    self.photo_url = photo_url
    self.current = 0
  end

  # Adds or removes points
  # @param [int] diff
  # @return [int]
  def change_points(diff)
    real_diff = current
    self.current += diff.to_i
    self.current = target if self.current > target
    self.current = 0 if self.current < 0
    real_diff = self.current - real_diff
    real_diff
  end
end
{% endhighlight %}

## Repositories

Repositories act as mediators between the domain and data mapping layers. In the best case, there are should be data access objects(DAO) to link repositories to DB, but if you use ActiveRecord and unlikely to change it, you can use your models as DAO.

Each repository contains common methods included in `Core::Repository`: `find`, `find_not_deleted`, `find_actual_by_user`, `save!` and `delete`. Also a repository can has its own methods to find entities like `find_by_name`, `find_triggered`, etc.

{% highlight ruby %}
#showme!
# Contains methods to work with adults and children entities
class Family::Repository::PersonRepository < Core::Repository
  include Core::Deletable
  # Sets all variables
  # @see Family::Child
  # @see Family::Adult
  def initialize(person_model)
    @model = person_model
  end

  # Find adult owned by user and not deleted
  # @param [int] id
  # @param [User::User] user
  # @return [Family::Adult]
  def find_actual_by_id_and_user(id, user)
    @model.where(id: id, user_id: user.id).not_deleted.take
  end
end
{% endhighlight %}

## Services

If you need to do something with several entities, you can use services. Also, if you just don't know where to place a method - put it in a service. Services might contain methods with similar purposes. Try to avoid replacing methods from services, because it is hard to debug. It is better from the very beginning to create several services than dividing one later.

{% highlight ruby %}
#showme!
# Contains methods to work with goals
class Goal::Service::GoalService
  # Sets all variables
  # @see Goal::Goal
  # @see Family::Child
  # @see Goal::Action
  # @see Goal::Factory::GoalFactory
  def initialize
    @model = Goal::Goal
    @child_model = Family::Child
    @action_model = Goal::Action
    @goal_factory = Goal::Factory::GoalFactory.new
  end

  # Gets child goals
  # @param [Family::Child] child
  # @param [Boolean] completed
  # @return [Goal::Goal][] goals
  def get_goals_by_child(child, completed)
    goals = child.goals.not_deleted
    return goals if completed.nil?
    return goals.where('current >= target') if completed
    goals.where('current < target')
  end

  # Adds or removes points
  # @param [Goal::Goal] goal
  # @param [Family::Adult] adult
  # @param [int] diff
  # @return [int]
  def change_points(goal, adult, diff)
    real_diff = goal.change_points(diff)
    @goal_factory.create_action_and_save(goal.user, goal, adult, real_diff)
    goal
  end
end
{% endhighlight %}

## Presenters

Presenters prepare entities for showing to users. It is nothing more to say.

{% highlight ruby %}
#showme!
# Contains methods to show adults and children
class Family::Presenter::PersonPresenter
  # Creates a presenter
  # @param [ActiveRecord::Base] person
  def initialize(person)
    @person = person
  end

  # Converts attributes to hash
  # @return [Hash]
  def person_to_hash
    {
      id: @person.id,
      name: @person.name,
      photo_url: @person.photo_url
    }
  end

  # Prepares goals to show to the user
  # @param [Boolean] is_completed
  # @return [Array]
  def child_goals_to_hash(is_completed)
    child_goals = Goal::Repository::GoalRepository.new.find_goals_by_child(@person, is_completed)
    return [] if child_goals.nil?
    child_goals.inject([]) do |goals, goal|
      goals.push Goal::Presenter::GoalPresenter.new(goal).goal_to_hash
    end
  end
end
{% endhighlight %}

## In addition

You can also use **Factories**, **Value Objects** and **Aggregates**.

Sometimes objects can be created in many different ways. In this case **Factories** can keep methods for creating entities.

A **Value Object** is an object that describes some characteristic or attribute but carries no concept of identity. The best example is location, composed of a country and a city. Location may be an attribute of a place and be presented by multiple fields in your database, but in your app, it will be one object.

An **Aggregate** is a cluster of domain objects that can be treated as a single unit. Any references from outside the aggregate should only go to the aggregate root. So you don't need to create factories and services for each domain object but only for the aggregate root.

# Conclusion

Using this approach you will create more classes and code - it's bad, but you will never get your code messy or convoluted. In the end, it will be very easy to find where needed methods are stored and where to add new.

Check the example [here](https://github.com/korolvs/thatsaboy).