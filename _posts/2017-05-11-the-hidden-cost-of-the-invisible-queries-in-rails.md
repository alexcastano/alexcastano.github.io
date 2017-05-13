---
title: The hidden cost of the invisible queries in Rails
description: Rails queries are awesome, but there are some drawbacks we should avoid to get the best performance
tags: [ruby on rails]
header:
  image: /assets/images/hidden_cat.jpg
  overlay_image: /assets/images/hidden_cat.jpg
  caption: 'Photo credit: [**mn-que**](http://es.freeimages.com/photo/cat-1378752){:target="_blank", rel="nofollow"}'
  overlay_filter: 0.5
last_modified_at: 13/05/2017

---

Ruby on Rails is awesome,
*it is really awesome*.
It allows us to create web applications without knowing every involved technology.
After all,
[Rails is optimized for programmers happiness](http://rubyonrails.org/doctrine/#optimize-for-programmer-happiness)
and it does a terrific job.

One of those technologies where Rails is really helpful is SQL.
It removes the complexity of creating manual SQL queries using ActiveRecord.
However, it is not only for our own good.
Today we will see some of the drawbacks
and how they affect the Rails application development.


## What the heck am I doing?

Yes, we know we are creating a Ruby on Rails application, but...

> Do you know exactly what are you doing on each line of your code?

This is not a trap question,
it is a very important one,
and it should be asked continuously while we are developing.
We are better coders
when we learn new implications of our code.

![image-center](/assets/images/i_have_no_idea_im_doing.jpg){: .align-center}

Rails helps us to write nice code,
although the less experimented developers forget that
behind a beautiful method call there are
queries to the database,
email sending,
calls to remote API
or messages to Redis. For example:

```ruby
# We are doing a query to the table movies
# where actor_id == actor.id
actor.movies.each do |movie|
  ...
end

# Sends an email
UserMailer.welcome_email(@user).deliver_now

# Here we are doing 3 different API calls to Stripe
customer = Stripe::Customer.retrieve("cus_AcnQorgF0BIeBt")
card = customer.sources.retrieve("card_1AHXpr2eZvKYlo2CtTq8IsO5")
card.name = "Noah Moore"
card.save

# We are sending a message to a Redis server
# with the needed data to perform a background task.
# Later a Sidekiq server will read it and it will be executed
MyWorker.perform_async(1, 2, 3)
```

All those operations have something in common:
*they have to communicate with external processes*.
Maybe with a process on the same computer, so they are faster,
or maybe on a remote server.

> Communication with an external process is slow

Anyway, the point is those communications take time,
much more than `meaning_of_life = 21 * 2`.
So we have to be careful when using them.
With a deeper knowledge about the framework we are using,
we start to write better code and
do fewer beginners mistakes.

Let's see some of those mistakes that I personally did
or I found in candidates’ code [Deverify submissions](https://deverify.com)


## The super infamous 1+N

The most common beginner mistake in Ruby on Rails:

```ruby
# This is a query
actor = Actor.find(actor_id)

actor.movies.each do |movie|
  # for each movie we are doing another query to get the director
  # so it is not very efficient
  puts movie.director.name
end
```

A good solution is to make only two queries:

1. Get all the movies from the actors.
2. Get all the directors using the already loaded movies.

This can be done in a convenient way with the `includes` method:

```ruby
# This is a query
actor = Actor.find(actor_id)
# This makes two queries, one for the movies
# and another for all the directors of the movies
actor.movies.includes(:director).each do |movie|
  # No query is performed here
  puts movie.director.name
end
```

After the `includes(:director)`,
all the movies have the associated director model loaded in memory.
Another way to get the same result is:

```ruby
# It makes the three queries here: actor, movies and directors
actor = Actor.find(actor_id).includes(movies: :director)

actor.movies.each do |movie|
  # No query is performed here
  puts movie.director.name
end
```

We have also the fantastic [gem Bullet](https://github.com/flyerhzm/bullet){:rel="nofollow"}
that warns us if it detects a 1+N query.
Take a look.
It is a very good one.

Finally,
we have this [fantastic Arkency post](http://blog.arkency.com/2013/12/rails4-preloading/)
to learn more about preloading associations in Rails.


## How far do you plan to count?

Another poor performance statement using ActiveRecord is:

```ruby
if Actor.count == 0
  puts "No actors"
end
```

This code performs a count query on the SQL server.
However, we don't need to count *all* the actors on the table.
The only thing we want is to know if there is at least one.

{% include figure image_path="/assets/images/counting_fast.jpg" alt="Counting fast"
caption="Counting fast by [Loic Djim](https://unsplash.com/search/count?photo=ft0-Xu4nTvA){:rel='nofollow'}" %}

The [SQL count query](https://wiki.postgresql.org/wiki/Slow_Counting)
can be slower than we think.
It possibly makes a sequential scan of the table.
This means the database will go through each row,
until the very last one,
just to return the number of actors.
A better approach is:

```ruby
if Actor.limit(1).count == 0
  puts "No actors"
end
```

Here we are telling to the database
to stop the query after finding the first one
and return the result immediately.
It's much more performant this way!

> Count only until you need

It's also applicable to compare the count with other numbers:

```ruby
if Actor.limit(2).count > 1
  puts "Many actors"
end

if Actor.limit(101).count > 100
  puts "More than 10 pages of 10 actors"
end
```

### Some Rails query methods seem not optimized

When I was writing this post,
I was testing alternatives methods in Rails to do the same.
It surprised me that the common methods like `any?`,
`many?` and `empty?` don't use the simple optimization I recommended before.
Maybe there is a reason for this behavior, but I don't know.

Let me show some examples:

```ruby
pry(main)> Actor.count
   (3.4ms)  SELECT COUNT(*) FROM "actors"
=> 9211
pry(main)> Actor.all.empty?
   (3.6ms)  SELECT COUNT(*) FROM "actors"
=> false
pry(main)> Actor.any?
   (3.3ms)  SELECT COUNT(*) FROM "actors"
=> true
pry(main)> Actor.many?
   (3.1ms)  SELECT COUNT(*) FROM "actors"
=> true
pry(main)> Actor.all.limit(1).count
   (0.8ms)  SELECT COUNT(count_column) FROM (SELECT  1 AS count_column FROM "actors" LIMIT $1) subquery_for_count  [["LIMIT", 1]]
=> 1
pry(main)> Actor.all.limit(2).count
   (0.9ms)  SELECT COUNT(count_column) FROM (SELECT  1 AS count_column FROM "actors" LIMIT $1) subquery_for_count  [["LIMIT", 2]]
=> 2

```

And a simple benchmark:

```ruby
Benchmark.ips do |x|
  x.report("any?") { Actor.any? }
  x.report("limit(1)") { Actor.limit(1).count }
  x.compare!
end

Comparison:
  limit(1):     1213.5 i/s
      any?:      597.1 i/s - 2.03x  slower
```

As we can see,
only with a table of 9.000 rows the difference is noticeable.
With 100.000.000 rows the difference will be huge.

If someone knows the reason,
please leave a comment about this.
Thank you.


#### Exists? method is good (Updated)

KD made a comment to add that the method `exists?` works as expected.

```ruby
2.2.4 :007 >User.exists?
User Exists (3.0ms) SELECT 1 AS one FROM "users" LIMIT 1
=> true
```

It uses the `limit 1`, so it is efficient.
Nevertheless, this method won't help us
when we want to know if there are 2, 3, 4 or more rows in the database.

## From wave-particle duality to relation-array duality on Rails

A `has_many` association in Ruby on Rails is not an array:

```ruby
pry(main)> actor.movies.class
=> Movies::ActiveRecord_Associations_CollectionProxy
```

But it can behave like one:

```ruby
pry(main)> Array.wrap(actor.movies)
=> [#<Movie:0x00000004a77660...
```

The `ActiveRecord_Associations_CollectionProxy` is a subclass of `ActiveRecord::Relation`
([see the code on Github](https://github.com/rails/rails/blob/b73e1c49e170f255daf0150b12234300725e782f/activerecord/lib/active_record/associations/collection_proxy.rb#L30){:rel="nofollow"}).

`ActiveRecord::Relation` is like a query proxy object
and it allows us to create pretty queries:

```ruby
User.where(surname: 'Stallman') # this returns an ActiveRecord::Relation
User.where(surname: 'Stallman').where(name: 'Richard') # and this one too
```

And the `ActiveRecord::Relation` class includes the module `Enumerable`
([see the code on Github](https://github.com/rails/rails/blob/ff0395721d79ebff18ce4c3c23dade9196dfaa87/activerecord/lib/active_record/relation.rb#L15){:rel="nofollow"}).

> It's an array, it's a relation, it's RAILS!

So there are methods in the `ActiveRecord_Associations_CollectionProxy`
that work like a SQL query
and others work like an array.
It is really important to see the difference
and take advantage of it when it is possible.

Let's see some examples.

### When once is better than twice

Imagine we want to print two movie lists,
one with the published movies and another with the unpublished movies.
If we do:

```ruby
puts "Published movies"
# First query
director.movies.where(published: true).each do |movie|
  puts movie.name
end

puts "Unpublished movies"
# Second query
director.movies.where(published: false).each do |movie|
  puts movie.name
end
```

We are doing two queries,
but we are loading all the movies for this director anyway.
It has better performance if we just load all movies at once
and then we use them like an array:

```ruby
# This is an array now
movies = director.movies.to_a

puts "Published movies"
movies.select { |m| m.published }.each do |movie|
  puts movie.name
end

puts "Unpublished movies"
movies.reject { |m| m.published }.each do |movie|
  puts movie.name
end
```

There are several situations
where doing a _select_ in plain ruby
is faster than doing it in the SQL server.
This isn't a general rule,
we should considerate each case independently.
For example,
if we are not showing all the models,
we are paginating them,
to do the _select_ in plain ruby is slower
because we are requesting and instantiating unused models.


### Where is my movie??

Does this code surprise you?

```ruby
pry(main)> director.movies.build(name: "Gone with the Rails")
=> #<Movie:0x00000007be57b0
 id: nil,
 name: "Gone with the Rails">

pry(main)> director.movies.size
=> 1

pry(main)> director.movies.count
   (1.4ms)  SELECT COUNT(*) FROM "movies" WHERE "movies"."director_id" = $1  [["director_id", 1]]
=> 0
```

It is the array-relation duality in action again!
First of all, we create a movie associated with the director;
but this model is not saved in the database
because we used the `build` method instead of the `create` method.

The method `ActiveRecord::Relation#size` is the _"array method"_
and the `ActiveRecord::Relation#count` is the _"relation/SQL method"_.
The `count` method will always perform a query.
Our new movie is not in the database so it is impossible to be counted.

On the other hand,
`size` is taking care of the models in memory (in the array),
and it will make the query only if it is really needed.

This is not the only case, though,
it can also happen in a very different situation:

```ruby
pry(main)> director.movies.build(name: "Gone with the Rails")
=> #<Movie:0x00000007be57b0
 id: nil,
 name: "Gone with the Rails">

pry(main)> director.movies.where(name: "Gone with the Rails").first
  Movie Load (1.1ms)  SELECT  "movies".* FROM "movies" WHERE "movies"."director_id" = $1 AND "movies"."name" = $2 ORDER BY "movies"."id" ASC LIMIT $3  [["director_id", 1], ["name", "Gone with the Rails"], ["LIMIT", 1]]
=> nil

pry(main)> director.movies.select { |m| m.name == "Gone with the Rails" }.first
  Movie Load (1.0ms)  SELECT "movies".* FROM "movies" WHERE "movies"."director_id" = $1  [["director_id", 1]]
=> #<Movie:0x00000007be57b0
 id: nil,
 name: "Gone with the Rails">
```

Again, the `where` method just performs the query
and the `select` method uses in memory models as well.

## Conclusion

{% include figure image_path="/assets/images/hidden_cat.jpg" alt="Hidden cat"
caption="So nice as the invisible queries in Rails" %}

At the end, this is not a critic to Ruby on Rails.
It is an awesome technology that I love,
and beginners have the opportunity to learn it fast.
As we learn we have to take into account this kind of peculiarities, though.

Specially if your application,
with thousands of active users,
is down because a query is really really slow;
and the problem is this harmless line:

```ruby
if notes.count > 0
```

Oh my ...! Believe me! You learn faster! **True story.**


So be aware of those issues and use it to your advantages to improve as programmer!
And RTFM! :wink:

What about you? What unexpected behaviour did you find developing your app?
Do you find bad performance issues due to invisible queries?
