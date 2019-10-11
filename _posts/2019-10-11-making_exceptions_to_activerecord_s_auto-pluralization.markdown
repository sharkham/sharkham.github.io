---
layout: post
title:      "Making exceptions to ActiveRecord’s auto-pluralization "
date:       2019-10-11 18:53:48 +0000
permalink:  making_exceptions_to_activerecord_s_auto-pluralization
---


For my first web app using Sinatra and ActiveRecord, I decided to make a simple Pokémon manager. Make an account through the site, enter the Pokémon’s species, an optional nickname field, save it, view other user’s Pokémon… et cetera. Basic functionality. It sounded simple, but when I started to actually build it out, I ran into problems. 

There is a lot of automatic pluralization in ActiveRecord, and ActiveRecord expects to encounter singular forms of words some places and plural forms in other places. It expects your table name to be the plural version of your model name, for example. If you were making a blogging platform, you would `create_table: posts` in your migration, which also needs to be named something like `20191003192830_create_posts.rb`. If you have a user class that `has_many` posts, you would access the posts belonging to this user with `user.posts`. 

All of this seems really convenient until you start trying to create a model with a name that does not pluralize like this! The plural of Pokémon is still Pokémon, so when I made a table named Pokémon and tried to use this with some other code, ActiveRecord still looked for a table named “Pokémons.” 

So I started googling. The first thing I tried was a method to manually set the table name in my model class: 

```
class Pokemon < ActiveRecord::Base
  belongs_to :user

  self.table_name = "pokemon"
end
```

And this did fix my first error! But then I still couldn’t access `user.pokemon` because the `has_many`/`belongs_to` relationship was looking for an attribute named `user.pokemons`—it wanted to pluralize too. 

I poked around some more, found a lot of answers online that seemed to be applicable only to Rails, asked some classmates and my instructor for help, and eventually cobbled an answer together. What I found was ActiveRecord’s built-in `Inflector` class. 

This is the class definition on [rubyonrails.org](https://api.rubyonrails.org/classes/ActiveSupport/Inflector.html):

> The Inflector transforms words from singular to plural, class names to table names, modularized class names to ones without, and class names to foreign keys. The default inflections for pluralization, singularization, and uncountable words are kept in inflections.rb.

`Inflector` contains the functionality to transform words from singular to plural, and also contains methods you can use to add exceptions to these rules. This does not require all of Rails to work; I was able to do this just running Sinatra and ActiveRecord! 

This is the code I added in my `environment.rb` file to fix my program: 

```
ActiveSupport::Inflector.inflections do |inflect|
  inflect.uncountable %w( pokemon )
 end
```

Telling the `Inflector` class that Pokémon was an uncountable word meant that when I named everything “pokemon”, from my table name to my model name to accessing a user’s many Pokémon through `users.pokemon`, ActiveRecord understood what I was doing. 

And after figuring this out I was able to take out the manually set table name in my Pokémon model too. It didn’t hurt anything, but it wasn’t necessary to correct the pluralization because adding an exception to the `Inflector` class fixed everything, and taking out this other code helped keep things dry. 

It’s possible to change how ActiveRecord pluralizes things in other ways with this method too, and the docs go into this in more detail. `inflect.irregular foo, fooze` would tell ActiveRecord that the plural of “foo” is “fooze”, for example. 

Doing this research helped me understand a lot more about how this part of ActiveRecord works, and was an overall satisfying example for me of how you really do learn more about a language by using it, not just studying it! I hope this can help someone else who has run into problems like I did with this project, and happy coding! 
