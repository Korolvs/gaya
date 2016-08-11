---
layout: post
title:  "Zero Downtime Deployment With Docker"
date:   2016-08-11 11:00:55
banner_image: zerodocker.jpg
meta_description: "There are parts of your application that need updates from time to time. You are using docker and you want to have no downtime during this updates? Let's talk about it!"
comments: true
---

There are parts of your application that need updates from time to time. You are using docker and you want to have no downtime during this updates? Let's talk about it!

## What we need to update?

There are three parts of application, which we will be updating:

 - Backend code
 - Frontend code
 - DB structure

We will try to update them separately and then we will talk about how we can update them together.

This is how our architecture might look like:

![Architecture]({{ site.url }}/assets/images/zero-downtime/arch-before.svg)

## Updating frontend

Assume, we have a server to serve static files, e.g. NGINX, and we want to update some html files. So we need another container to build(concatenate, minify, uglify, etc) assets and place them to a volume with static files.

![Updating frontend]({{ site.url }}/assets/images/zero-downtime/frupdate.svg)

The problem is that browsers can cache old assets and user will have a mix of old and new assets. Also, if you dynamically load some files, users can have old index.html and try to load new versions of other files.

The solution is versioning. Each new build must have a unique id, and you need to keep old versions.

![Versioning with number]({{ site.url }}/assets/images/zero-downtime/versioning-num.svg)
![Versioning with hash]({{ site.url }}/assets/images/zero-downtime/versioning-hash.svg)

## Updating backend

For backend updates we will use [`dockercloud/haproxy`](https://github.com/docker/dockercloud-haproxy) image which is just brilliant. Why brilliant? When you run `docker-compose scale`, it automatically connects new containers to HAProxy and removes old ones. It can work with Docker Swarm and doesn't need Councul or other tools.

So we have architecture like this:

![Updating backend before]({{ site.url }}/assets/images/zero-downtime/backbefore.svg)

### Step 1

Run `dockec-compose build api` to build new API.

### Step 2

Run `docker-compose scale api=4` to add new containers. HAProxy will add them to the configuration and will send requests to them.

![Updating backend step 2]({{ site.url }}/assets/images/zero-downtime/backmiddle.svg)

By default, HAProxy adds new containers immediately, but we need to give them time to prepare. Therefore I added some delay before linking new containers to the script that works with docker events. You can find it [here](https://github.com/korolvs/thatsaboy/blob/master/docker/prod/proxy/eventhandler.py)

### Step 3

Now we can remove old containers. But if we run `docker-compose scale api=2`, docker will remove new ones and keep old. So we need to remove them with `docker stop` and `docker rm`.

![Updating backend step 3]({{ site.url }}/assets/images/zero-downtime/backafter.svg)

I wrote a [bash script](https://github.com/korolvs/thatsaboy/blob/master/run_update_api.sh) that runs these steps, and now you can update API with one command.

## Updating database

It is the easiest and the hardest part at the same time. If migrations will not break your current backend, you just need to add another container, which will connect to the database and run migrations.

![Updating database]({{ site.url }}/assets/images/zero-downtime/dbupdate.svg)

## Updating database and code

For example, you want to change column type. You also want to be able to rollback, of course.

To achieve it, you need to keep your API and DB compatible with each other after every migration. It might be hard to do.

![Updating database]({{ site.url }}/assets/images/zero-downtime/changecolumn.svg)

You can find some other examples [here](https://www.rainforestqa.com/blog/2014-06-27-zero-downtime-database-migrations/)

## Conclusion

As you can see, zero downtime with docker is possible and seems rather easy. But you should be careful when you are updating different parts of your application at once.

Check the example [here](https://github.com/korolvs/thatsaboy).

Also, you can check [Refactoring controller actions in Ruby on Rails]({% post_url 2016-06-21-refactoring-controller-actions-in-ruby-on-rails %}).

And read about [Domain-driven design for Ruby on Rails]({% post_url 2016-05-08-domain-driven-design-for-rails %}).
