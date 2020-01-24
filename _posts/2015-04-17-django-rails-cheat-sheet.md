---
layout: post
title:  "Django/Rails cheat sheet"
date:   2015-04-17 15:44:39 +0100
categories: python django
tags: django rails
image: django-rails-cheat-sheet.jpg
short_description: A more detailed list of for Django developers working in Rails, covering the differences in ORM, tasks, forms and more.
keywords: "python, ruby, tutorial, django, rails, cheatsheet, orm, tasks, forms"
canonical_url: https://medium.com/@yeraydiazdiaz/django-rails-cheat-sheet-50adf2441913
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">
Hey, Django dev, come here for a sec!

I heard you're on Rails now, looking for some quick answers?

Yeah, I'm your man, got them right here.

<!--more-->

<div class="note">
<a href="{{ page.canonical_url }}" target="_blank"><em>Django/Rails cheat sheet</em> was first published on Medium</a>, if you'd like to comment or give feedback please do so there.
</div>

## ORM

I’m gonna be straight with you, Django’s ORM is nicer than Rails’s.

There, I said it. Now that it’s out of my system let’s find some models:

```
# One model
Dj> Model.objects.get(id=1)
Ra> Model.find(id: 1)  # when using a PK
Ra> Model.find_by(name: "A name")  # when using any other field
```

So far so good, Rails uses [`find`](http://api.rubyonrails.org/classes/ActiveRecord/Associations/CollectionProxy.html#method-i-find) when looking up by ID, [`find_by`](http://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-find_by) when looking up by anything else.

```
# Single record filtered collection
Dj> Model.objects.filter(name="First")
Ra> Model.where("name = First")

# Multiple record filtered collection
Dj> Model.objects.filter(name__startswith="First")
Ra> Model.where("name LIKE 'First%'")
```

I can almost hear you saying “Is that (gasp) SQL?”. Yep, it is, it’s good to get back to the basics sometimes, am I right or am I right? Basically, there’s no field lookups, but [Django’s field lookup documentation’s](https://docs.djangoproject.com/en/dev/ref/models/querysets/#field-lookups) got the SQL equivalents for you.

Let’s add some ordering:

```
Dj> Model.objects.all().order_by('created_at')
Ra> Model.all.order(created_at: :desc)  # or 'created_at DESC'
```

and selecting single and multiple elements:

```
Dj> Model.objects.all()[5:10]
Ra> Model.all.offset(5).limit(5)

Dj> Model.objects.all().first()
Ra> Model.first  # and .second and .third, etc.

Dj> Model.objects.all().last()
Ra> Model.last
```

Selecting specific fields:

```
# Single array of fields
Dj> Model.objects.values_list('field', flat=True)
Ra> Model.pluck(:field)

# Multiple fields
Dj> Model.objects.values('field', 'otherfield')
Ra> Model.pluck(:field, :otherfield)  # array of arrays
Ra> Model.select(:field, :otherfield) # array of partial objects
```

When passing multiple fields Rails’s [`pluck`](http://api.rubyonrails.org/classes/ActiveRecord/Calculations.html#method-i-pluck) will return nested arrays of fields values. If you’d rather have something like an array of hashes with field-value pairs like Django’s `values` you can use [`select`](http://api.rubyonrails.org/classes/ActionView/Helpers/FormBuilder.html#method-i-select). Notice however that `select` initializes objects with only the selected fields, it’s not an actual hash.

Now let’s join stuff!

```
# Query spanning relations
Dj> Model.objects.filter(other_model__id=3)
Ra> Model.joins(:other_models).where("other_models.id = 3")
```

Notice the [`joins`](http://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-joins), the argument needs to match the relation name in the ActiveRecord, like [`belongs_to`](http://guides.rubyonrails.org/association_basics.html#the-belongs-to-association) or [`has_one`](http://guides.rubyonrails.org/association_basics.html#the-has-one-association). Also note the prefixing of the ID in the `where`, it needs to match the other model’s table, which should just be the plural of the model’s name if you follow Rail’s database conventions.

Creating is quite similar:

```
Dj> Model.objects.create(field="foo", other_field="bar")
Ra> Model.create(field: "foo", other_field: "bar")
```

As is updating:

```
Dj> Model.objects.get(5).update(field="bar")
Ra> Model.find(5).update(field: "bar")

or, less efficiently...

Dj> m = Model.objects.get(5)
Dj> m.field = "bar"
Dj> m.save()

Ra> m = Model.find(5)
Ra> m.field = "bar"
Ra> m.save
```

And finally deleting/destroying:

```
Dj> Model.objects.get(5).delete()
Ra> Model.find(5).destroy
```

Both of these have a collection-specific cousin, `update_all` and `destroy_all`.

```
Dj> Model.objects.filter(field="foo").update(field="bar")
Ra> Model.where(field: "foo").update_all(field: "bar")

Dj> Model.objects.all().delete()
Ra> Model.all.destroy_all
```

In Rails you’re bound to encounter two similarly named methods, for example `create` and [`create!`](http://api.rubyonrails.org/classes/ActiveRecord/Associations/CollectionProxy.html#method-i-create-21). This pattern is common in Ruby in general, the latter will raise an exception if it fails and the former will return `false`.

Django’s lovely [aggregation/annotation functions](https://docs.djangoproject.com/en/1.7/ref/models/querysets/#aggregation-functions) are present in Rails via the [`calculate`](http://api.rubyonrails.org/classes/ActiveRecord/Calculations.html#method-i-calculate) method, the more common ones have shortcuts methods:

```
Dj> Model.objects.count()
Ra> Model.count

Dj> Model.objects.all().aggregate(Sum("num_field"))
Ra> Model.sum(:num_field)

Dj> Model.objects.all().annotate(Max("other_model__num_field"))
Ra> Model.joins(:other_models).maximum("other_models.num_field")
```

Again, notice how in Rails you need specifically join to the other model before specifying the column in [`maximum`](http://api.rubyonrails.org/classes/ActiveRecord/Calculations.html#method-i-maximum).

## Management tasks

You’ve probably memorized a bunch of management commands for Django for several tasks and most likely created custom ones for your projects.

Rails of course has the same concept, some of them are specific to Rails, like commands to generate migrations, and others are defined using Ruby’s [`Rake`](http://docs.seattlerb.org/rake/) and defined in a `Rakefile`. Here are a few of the most common ones:

```
# Start a project
Dj> django-admin startproject myproject
Ra> rails new myproject
```

This will generate all the basic folders for you to start working on.

```
# Create a migration
Dj> django-admin makemigration
Ra> rails generate migration AddDescriptionToModel
```

As you know Django’s migration system, and in general, is based around models. I wrote an overview of the differences between the two architectures if you want more details, for now just now that Rails does not do migrations based on models. `generate migration` will generate a file in `db/migrate` with a timestamp for you to add the migration commands of your database.

```
# Execute the migrations
Dj> django-admin migrate
Ra> rake db:migrate

# Rollback
Dj> django-admin migrate 0001_initial
Ra> rails db:rollback  # the latest migration
Ra> rails db:migrate MIGRATION_ID  # usually a timestamp
```

Notice how this one is a Rake task, this will run through the migrations and apply the ones that have not been already applied.

```
# List the migrations and their status
Dj> django-admin showmigrations --list
Ra> rake db:migrate:status
```

Loading fixtures:

```
Dj> django-admin loaddata my_data.json
Ra> rake db:seed
```

Rails’s seeds are scripts that can load any type of data. The `seed` command will run through all the scripts in `db/seeds` directory and execute them.

Our trusted shell and the dev server:

```
Dj> django-admin shell
Ra> rails console

Dj> django-admin runserver
Ra> rails server
```

Adding new boilerplate code for you to work on.

Dj> django-admin startapp mynewapp
Ra> rails generate controller mynewcontroller mynewaction

These are not really equivalent, Rails does not have the concept of apps, there is only one. The generate controller command will create a controller with a single action named mynewaction.

And of course testing:

```
Dj> python manage.py test
Ra> rake test

# Specific tests
Dj> python manage.py test myapp.tests.MyTest
Ra> rake test tests/my_test_suite.rb TestTheTruth
```

Now, to create your own management commands in Rails you need to define them in `lib/tasks`. The tasks should have a `.rake` extension and you probably want to namespace them:

```
namespace :my_namespace do
  desc "Description of My Cool Task"
  task :my_cool_task => :environment do
    puts "This is cool!"
  end
end
```

If you run `rake -T` you should see your task under the namespace ready to be executed.

## Forms

Vanilla Rails does not have a specific class for forms, however it does ship with a [series of template helpers](http://guides.rubyonrails.org/form_helpers.html) to make writing forms easier, but that’s probably not enough for you if you come from Django.

Luckily we can create a similar behaviour to Django’s Forms using a combination of the [Virtus gem](https://github.com/solnic/virtus) and some elements from ActiveRecord base class. What’s Virtus? [Adam Hawking explains it best](http://hawkins.io/2014/01/form_objects_with_virtus/) but essentially it allows you to create attributes that will be coerced to the correct data type in Ruby.

What about validations? Turns out we can use that ActiveRecord’s [Validations framework](http://api.rubyonrails.org/classes/ActiveModel/Validations.html) to define them.

Remember that pattern in [Django’s ModelForms](https://docs.djangoproject.com/en/1.8/topics/forms/modelforms/#module-django.forms.models) where we simply call `save()` on the form if it’s valid? We can do a similar thing using [`ActiveRecord::Conversion`](http://api.rubyonrails.org/classes/ActiveModel/Conversion.html) which allows you to implement persistence methods.

If we wrap all into a custom BaseForm class:

{% highlight ruby %}
class BaseForm
  include Virtus.model
  include ActiveModel::Conversion
  include ActiveModel::Validations

  def persisted?
    false  # to be implemented in the subclass
  end

  def save
    if valid?
      persist!
      true
    else
      false
    end
  end

  private

  def persist!
    raise  # to be implemented in the subclass
  end

end
{% endhighlight %}

which we can subclass this BaseForm to create more specific forms with custom methods for persistance and validation:

{% highlight ruby %}
class BookForm < BaseForm

  attribute :book, Book
  attribute :title, String

  validates :title, presence: true

  def initialize(book, attributes = {})
    self.book = book
    self.title = attributes[:title] || book.title
  end

  def persisted?
    self.book.persisted?
  end

  def persist!
    book.title = title
    book.save!
   end

end
{% endhighlight %}

This will allow you to create form objects and validate the fields even in the console:

```
Ra> BookForm.new(Book.first).persisted?
=> true

# valid forms
Ra> form = BookForm.new(Book.new, {:title => "Crime and punishment"})
=> #<BookForm...>

Ra> form.valid?
=> true

Ra> form.persist!
=> true

# invalid forms
Ra> form = BookForm.new(Book.new)
=> #<BookForm...>

Ra> form.valid?
=> false
```

These forms along with the form helpers should have your form needs covered.

---

So there you go, that’s about as quick cheat sheet I can give you to get you past the first hurdles when you’re used to Django and find yourself working in Rails.

I’ll let you get back to it now. Have fun!
</div>
