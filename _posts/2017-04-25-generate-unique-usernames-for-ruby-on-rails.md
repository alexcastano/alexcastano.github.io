---
title: Username generation in Ruby on Rails
tags: [ruby on rails]
header:
  overlay_image: /assets/images/name_label.jpg
  caption: 'Photo credit: [**Ryan McGuire**](http://gratisography.com/){:target="_blank", rel="nofollow"}'
  overlay_filter: 0.5
last_modified_at: 04/05/2017

---

Today we are going to create a unique username automatically
using the user's full name in a typical Ruby on Rails application, i.e.,

```
"Alex Castaño" => "alex_castano"
```

It can be very useful when creating a sign-up form,
as we want it to be as simple as possible,
like our [Deverify register form](https://deverify.com/register).

We don't want to bother our potential user with another field.
Furthermore, it has to be unique in our database;
it could be a pain for our user to find an available one.
If the username is very visible in the application,
we could give the user the ability to change it if he doesn't like it.

{% include figure image_path="/assets/images/name_label.jpg" alt="names" %}


## The username conventions

Our username can only contain lower letters and underscores: `/[a-z_]+/`.
We will use the full name given by the user to generate this username.
However, the full name can contain capital letters, spaces, accents, etc.
So our first step is to turn our full name in a valid username,
keeping it as similar as possible:


```ruby
  def generate_username(fullname)
    ActiveSupport::Inflector.transliterate(fullname) # change ñ => n
      .downcase              # only lower case
      .strip                 # remove spaces around the string
      .gsub(/[^a-z]/, '_')   # any character that is not a letter or a number will be _
      .gsub(/\A_+/, '')      # remove underscores at the beginning
      .gsub(/_+\Z/, '')      # remove underscores at the end
      .gsub(/_+/, '_')       # maximum an underscore in a row
  end
```

In addition to the rules above we added:

  * Avoid username which starts with or ends with an underscore.
  * We don't want two underscores together; it can be confusing to type.

As Julien says in the comments, we can simply use:

```ruby
fullname.parameterize(separator: '_')
```

And we'll get the same result with less code!

## Avoid username duplications in the database


The next step is to prevent duplications in the database.
The best way to do this is **adding a unique index in the username column**,
so we should do it if we haven't already done it in a previous migration.
It will also make our queries faster,
and we need it to avoid concurrency problems that are explained later.


```ruby
class AddIndexToUsernameInUsers < ActiveRecord::Migration[5.0]
  def change
    add_index :users, :username, unique: true
  end
end
```

So what should it happen if the generated username,
`john_doe`,
is already in the database?
With our solution, it should generate `john_doe_2`.
But if this username is already taken too?
Well, as you can imagine, the function should return `john_doe_3`.

The code would be something like this:

```ruby
  def find_unique_username(username)
    taken_usernames = User
      .where("username LIKE ?", "#{username}%")
      .pluck(:username)

    # username if it's free
    return username if ! taken_usernames.include?(username)

    count = 2
    while true
      # username_2, username_3...
      new_username = "#{username}_#{count}"
      return new_username if ! taken_usernames.include?(new_username)
      count += 1
    end
  end
```

The given username variable is the one we generated previously with generate_username.
We will imagine that the value is `'john_doe'`.

Here, it is important thing is to minimize the number of queries we make to our database.
Instead of doing a query for each username,
we will do only one query for all the current usernames that start with `'john_doe'`.
The regexp is `/^john_doe.*/`, but in SQL, it is `john_doe%`.
We use the [ActiveRecord method pluck](http://api.rubyonrails.org/classes/ActiveRecord/Calculations.html#method-i-pluck)
which is ideal for this situation.

First, try to check if `john_doe` is taken.
If it is not, we'll return it.
Otherwise, we iterate generating the usernames:

1. `john_doe_2`
1. `john_doe_3`
1. `john_doe_4`

Until one is not taken.

## Avoid concurrency problems with unique validation in Rails

Finally, we have to deal with the concurrency problem.
If two separate requests reach our servers, at the same time,
with both the same name,
one of them will fail (if we added the unique index).
This problem is more common that you can think at first,
and it should not be ignored!
Especially as it is so easy to fix.

There are a lot of pages [talking about the problem of uniqueness validation](https://robots.thoughtbot.com/the-perils-of-uniqueness-validations),
so we know we should not use it.
The SQL unique index is a more appropriate tool
however Rails does not deal with it nicely.
If the row already exists,
Rails launches an exception we would have to capture.

Fortunately, rescue unique constraint gem
solves exactly this problem.
Personally,
I'd like a solution better integrated like [Phoenix/Ecto with `#check_constraint/3`](https://hexdocs.pm/ecto/Ecto.Changeset.html#check_constraint/3),
but Rails has its own personality.

Anyway, it is a small gem with no dependencies,
so you can add it directly
or you can to "copy & paste & modify" its code in your project.

Another difficulty is to differentiate between the same error in different fields:

* `email` is taken: we should notify the user, probably he is already registered
* `username` is taken: we have to generate a new username again because a concurrent thread was faster

For example:

```ruby
  def create_user(params)
    username = generate_username(params.fetch(:name))
    username = find_unique_username(username)
    user = User.create(params.merge(username: username))
    if user.errors.details[:username].any? { |h| h[:error] == :taken }
      # a faster thread took our username, try again!
      create_user(params)
    else
      user
    end
  end
```

## Conclusion

The code is not very long, and it does what is expected.
We could integrate it in almost any Rails project.
I recommend having all this user creation code in a [service object](http://blog.arkency.com/2016/10/the-esthetics-of-a-ruby-service-object/),
or much better,
considerate creating a [form object](https://robots.thoughtbot.com/activemodel-form-objects).

However you do it,
the most important is to simplify the life of our users.
And this is our job, isn't it?
