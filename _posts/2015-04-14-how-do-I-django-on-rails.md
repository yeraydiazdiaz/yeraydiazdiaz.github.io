---
layout: post
title:  "How to I Django on Rails?"
date:   2015-04-14 15:44:39 +0100
categories: blog
tags: django rails
image: how-do-i-django-on-rails.jpg
---

You are seasoned Django developer.

You are master of the models. Ruler of the views. King of the templates.

But fate has brought you to Rails now, and you feel clumsy, slow, googling every step of the way. You're asking yourself how do I Django in Rails?

<!--more-->

I've been there and it's not fun. The good thing is that Rails has been used and loved for a long time, there's a lot of really nice resources out there. My two favourites are the official Rails guides and API documentation.

But I've found there's a lot differences in naming and workflow coming from Django that can be a pain to research. So this is my attempt to make your journey easier.

Rails and Django are really quite similar, they are both MVC-type web frameworks so many of the concepts are the same. In a nutshell:

- Django's **models** are Rails's **ActiveRecord models**
- Django's **views** are Rails's **controller actions**
- Django's **templates** are Rails's **views**
- Django's **URLs** are Rails's **routes**
- Django is **explicit** Rails is **implicit** (Convention over configuration)

## Models

Let's start at the core. Much of Django revolves mainly around models, you define the schema in the class, then Django detects changes and takes care of database migrations. Nice.

In Rails, however, the table definition is completely separate from that of the ActiveRecord class. It is your responsibility to define the tables using migrations and then map some of these fields to the model in [ActiveRecord](http://guides.rubyonrails.org/active_record_basics.html) yourself. Remember the *Convention over configuration* I mentioned before? That's an example. Rails expects a [field naming convention](http://guides.rubyonrails.org/active_record_basics.html#convention-over-configuration-in-active-record) in the migration in order for things to work smoothly.

I know, Django takes care of that for you but you need to *start letting go*, I promise there are will be good bits.

Just keep in mind that the ActiveRecord does not hold all the truth about the model like it does on Django. If you need to look up the names of the model's fields check out the file generated after the migration, usually **schema.rb**.

In the ActiveRecord class you define the [associations](http://guides.rubyonrails.org/association_basics.html) between models (ForeignKeys and M2M definitions) for the ORM to work as intended and it's where you write any logic to do with the model like database callbacks and auxiliary methods for complex queries.

Still with me? No biggie, right? You can handle that. Let's move on.

## Controllers

Now, what Django calls *views* Rails calls *controller actions*, but in essence they're exactly the same. They both retrieve the information and pass it to a particular template for rendering. Rails's *controller* is the class that implements the *actions*, very similar to Django's class-based views. Just a couple of things to keep in mind, though:

As another example of *Convention over configuration*, Rails expects the action's template to have the same name. So much in fact that an action does **not** need to be defined in the controller if a template file exists with the same name in a folder named as the controller. So make sure your view files are up to date at all times. You can of course use whichever template name and force Rails to render it, but it's best to embrace the convention, trust me.

Similarly the action will automatically render the template and return it. You do not need to render and return explicitly.

Any parameters captured by the routing (more on that later) or passed in the request are available in the params hash.

Finally, there's no concept of context for the view to render, you create instance variables on the controller i.e. `@info` which you can then retrieve in the view.

Hey, you wanna see something cool? If you embrace the *Rails's Convention over configuration* you can have an action as simple as a single query populating an instance variable. That's it, no more code. Since the view is derived from the controller and action names and it is automatically rendered at the end of the action that's all you really need. Cool, huh? Told you there'd be good bits.

## Templates

I'll go quickly through these because they're basically the same. Django's *templates* are Rails's *views* and they behave the exact same way. Rails expects to find them in the `app/views` directory, under a folder named after the controller and with the same name as the action.

Rails, of course, implements view inheritance similarly to Django. A base view, usually a *layout* is defined and blocks of content are pulled from child views using `yield` or `content_for`. The child views define blocks of HTML using the `content_for` helper.

If you want a reusable bit of template in Django you use `include with` and pass the variables in a hash. In Rails this is called a `partial` and, yes, you guessed it, there's a naming convention for them. They need to start with an underscore, but they can live in any folder you'd like. You invoke them using the `render` helper and you can pass a hash to map the variable names.

Cool, that takes care of MVC and we're looking good. One more thing before you alt-tab to your editor.

## Routing

First off, there's a nice command you can run to get a summary of the routes, **rake routes**. As you make changes you may want to double check everything looks good.

Rail's basic routing is very similar to Django's. In it's most basic for the main difference is that you include the request method in the URL, i.e:

{% highlight ruby %}
get 'hello' => 'first_run#index', as: 'first_run'
{% endhighlight %}

(You realize this is the first bit of actual code I've pasted, right? Feeling pretty good about that…)

Anyway, `hello` is the path, `first_run` is the controller, `index` is the action and finally the last `first_run` is an alias to be used in reverse lookups. Nice and simple.

If you need to capture a variable in the request you simply add a symbol to the path:

{% highlight ruby %}
get 'hello/:name' => 'first_run#index', as: 'first_run'
{% endhighlight %}

The action's `params` will now include whatever is captured after the slash. Notice the lack of a regex typical of Django, Rails does not do that by default, it will capture just about anything. If that makes you itchy (it did for me) you can add constraints to the rule.

{% highlight ruby %}
get 'hello/:name' => 'first_run#index', as: 'first_run', constraints: { name: /[a-zA-Z]+/ }
{% endhighlight %}

Rails has a shorthand for quickly RESTful APIs that you may want to use, resource. Just type `resource :things` and Rails will create routes for GET and POST for `/things` and GET, PUT, PATCH and DELETE for `/things/:id`. Quite handy. You can also pass it a block to add more routes, like a intermediate `delete` route for a confirmation page before deleting.

You will, of course, want to resolve URLs from the action names, like Django does with the lovely `reverse` helper. Rails does not have a generic helper but rather creates a function with the suffix `_path` for each route's alias. So in our example the function would be `first_run_path`. As with Django the function will fail if the action requires parameters and you don't pass them. A typical call would be:

{% highlight ruby %}
first_run_path(name: 'yeray')  # returns "/hello/yeray"
{% endhighlight %}

And that's all we have time for today. There are still a few quick tips I've gathered for different tasks that I'll compile into a separate post. For now I hope you're better suited to Django in Rails.