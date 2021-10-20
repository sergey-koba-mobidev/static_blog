---
title: "How to survive without a framework in Ruby (developing a Rack Based MVC Framework)"
date: 2017-06-06T16:55:07+03:00
tags: ["ruby", "rack", "hanami", "sequel", "cells"]
disqus_identifier: "rack-mvc-framework"
draft: false
---
You are developing with Ruby on Rails, but always dreamed to create your own framework with blackjack and other things? Don't know where to start? **You should start here!**

Building a ruby based framework using standalone gems and some tape :) Although the result is quite awesome! Anyway you are reading this blog, and it is using self-developed framework like we are going to create.

![](http://res.cloudinary.com/skoba/image/upload/v1496702152/ruby_doodle_800x600px_fwd5j4.png)

<!--more-->

### Introduction

This story begins with a conversation with my good friend and colleague [Ievgen Kuzminov](http://stdout.in/), who noticed an interesting difference between Ruby and PHP developers.

Almost every PHP developer starts to write his first web sites on pure PHP, later he finds out there are some frameworks to make his life easier. We can say he evolved. This way a PHP developer first learns the basics and later more advanced things like frameworks. A Ruby developer usually takes the opposite way, first thing he learns is a Ruby on Rails framework. And only after some period of time he becomes interested (let's be optimistic) in internal things. How does it work? What is inside Rails? I hope that if you try to build your own framework at least once you will be much more familiar with any framework internals and of course Rails. We will try to build a framework with minimum magic inside.

### Rack

Rack is a foundation of almost every ruby web framework. It is very possible the framework you use has Rack inside, of course Ruby on Rails has it too. Rack is an interface, which provides API for creating ruby web applications. It is responsible for receiving and processing HTTP request, parsing parameters, forms, cookies and etc. No more words, let's just start with it :)

A Rack based application should have a **call** method, which takes Environment object as a param and returns a Rack response object. For example:

```ruby
class MyBlog  
   def call (env)  
      [200, {"Content-Type" => "text/html; charset=utf-8"}, ["Hello World"]]
   end  
end
```

A **call** method should return an array of 3 elements: HTTP answer code, HTTP headers and HTTP body. To run an application you will need 3 files:

```
my_blog/  
 ├── Gemfile  
 ├── config.ru  
 └── my_blog.rb
```

```ruby
# Gemfile
resource "https://rubygems.org"  
gem 'rack'
```

```ruby
# config.ru
require relative 'my_blog'  
run MyBlog.new
```

**config.ru** \-  is a configuration file for **rackup** command. We are going to use it to launch Rack based application. In our case it does the only thing - points to **MyBlog class**, which has a **call** method. Let's run the application

**bundle exec rackup --port 3000 --host 0.0.0.0**

Following this link [http://localhost:3000](http://localhost:3000) you should see our awesome **Hello World**. Congratulations!!! You wrote your first ruby web application without Rails or any other framework ;) PS: there are more details about Rack [here](https://launchschool.com/blog/growing-your-own-web-framework-with-rack-part-1)

### Routes

So what our next steps are? Right, let's add more pages to our blog, for example:

```ruby
# my_blog.rb

class MyBlog  
  def call(env)  
    req = Rack::Request.new(env)  
    case req.path_info  
    when /posts/  
      [200, {"Content-Type" => "text/html"}, ["<h1>Posts</h1>"]]  
    when /about/    
      [500, {"Content-Type" => "text/html"}, ["<h1>About</h1>"]]  
    else  
      [404, {"Content-Type" => "text/html"}, ["No one here..."]]  
    end  
  end  
end
```

Current code is quite simple, we analyze url part in **path_info** property of **Rack::Request** class. Using case operator we return different answers to browser.

What do you think looking at this code? As for me I wish we had 3 things:

1.  I want to configure url paths in a more flexible and convenient way, rather than case operator. I am talking about Routing. 
2.  We need a class(es), which receive the request and executes some business logic - Controller.
3.  Store the generated html code in a separate files, not in strings. I mean Views.

How to write your own router from the scratch you can learn from this [awesome article](https://mkdev.me/en/posts/how-to-write-an-mvc-framework-in-ruby). I decided to use a lightweight and fast [Hanami router](https://github.com/hanami/router).

```ruby
# Gemfile  
source "https://rubygems.org"  
gem 'hanami-router'
```

Don't forget to run bundle...

```ruby
# my_blog.rb  
require 'hanami/router'  
class MyBlog  
  def self.router  
    Hanami::Router.new do  
      get '/',        to: ->(env) { [200, {}, ['Hello from My Blog']] }  
      get '/about',   to: ->(env) { [200, {}, ['<h1>About</h1>']] }  
      get 'post/:id', to: ->(env) { [200, {}, ["Post #{env['router.params'][:id]}"]] }  
    end  
  end  
end
```

Class **Hanami::Router** has call method, required for Rack, so we can use it as entry point for our blog

```ruby
# config.ru  

require_relative 'my_blog'  
run MyBlog.router
```

To check the result let's run **rackup --port 3000** command and go to [http://loclahost:3000/post/32](http://loclahost:3000/post/32). You should see "Post 32" at the screen.

### Controllers

Investigating Hanami I also noticed [controller micro library](https://github.com/hanami/controller). I see Single Responsibility principle realization there, because each Action is a separate class in Hanami controllers, for example:

```ruby
# app/controllers/post/show.rbrequire 'hanami/controller'  
module Post  
  class Show  
    include ::Hanami::Action  
    def call(params)  
      self.body = "Post #{params[:id]}"  
    end  
  end  
end
```

In Ruby on Rails we would put all actions for posts in one controller. And this is good from grouping point of view, so all posts actions are in one place. I solved this task by putting all actions files for posts in one folder controllers/post.

```
my_blog/ 
 ├── app      
     ├── controllers          
     ├── post
         ├── show.rb  
 ├── Gemfile  
 ├── config.ru  
 └── my_blog.rb
```

There is no autoloading (Hooray!), therefore we require controllers folder explicitly

```ruby
# config.ru  
require_relative 'my_blog'  
```

```ruby
# Load controllers  
Dir[File.join(File.dirname(__FILE__), 'app/controllers', '\*\*', '\*.rb')].sort.each {|file| require file }  
run MyBlog.router
```

of course you should install a gem

```ruby
# Gemfile  
source "https://rubygems.org"  
gem 'hanami-router'  
gem 'hanami-controller'
```

And... Surprise! Hanami controllers are easily integrated with Hanami routers (it's amazing!!!)

```ruby
# my_blog.rb
#...
get 'post/:id', to: Post::Show
#...
```

Restart your server and go to [http://loclahost:3000/post/32](http://loclahost:3000/post/32.), to make sure "Post 32" is still there.

### Database

Every realworld web application works with a data stored in database. Searching for a suitable solution I followed 3 rules (magical number): migration support, models, not ActiveRecord ;). I choosed [sequel](https://github.com/jeremyevans/sequel) gem. It has a large number of features and plugins, you should definitely take a look at it. Let's start right now...

Not inventing the wheel I created configuration file **config/database.yml** with the next contents:
```
adapter: postgres  
host: db  
database: blog  
user: postgres  
password: development
```

Next you should install **sequel** and **pg** gems and initialize DB connection in config.ru

```ruby
require 'yaml'  
require 'sequel'  
# Init Db  
db_config_file = File.join(File.dirname(__FILE__), 'config', 'database.yml')  
if File.exist?(db_config_file)  
  config = YAML.load(File.read(db_config_file))  
  DB = Sequel.connect(config)  
  Sequel.extension :migration  
end
```

As you can see the active database connection will be stored in **DB** global variable. Also I added a migration extension for Sequel (don't forget to actually create a database)!

Next create a migration and **Post** model. For this we will need first to create 2 folders **migrations** and **app/models**

```ruby
# migrations/001_create_table_posts.rb  
class CreateTablePosts < Sequel::Migration  
  def up  
    create_table :posts do  
      primary_key :id  
      column :title, :text  
      column :content, :text  
      column :created_at, :timestamp  
      column :updated_at, :timestamp  
    end  
  end  
  def down  
    drop_table :posts  
  end  
end
```

And a **Post** model

```ruby
# app/models/post.rb  
class Post < Sequel::Model(DB)  
end
```

Nothing special, let's load models exactly the same way as controllers previously

```ruby
# config.ru  
# ...  
# Load models  
Dir[File.join(File.dirname(__FILE__), 'app/models', '\*\*', '\*.rb')].sort.each {|file| require file }# ...
```

To make things really simple we will be checking and running migrations right from **config.ru** file

```ruby
# config.ru# ...# If there is a database connection, run all the migrations  
if DB  
  Sequel::Migrator.run(DB, File.join(File.dirname(__FILE__), 'migrations'))  
end
```

If you tried to run the application, you probably noticed the **Post is not a module (TypeError),** which occurs because previously we defined Post as a module in controller (now it's a model class). Let's fix it

```ruby
# app/controllers/post/show.rbrequire 'hanami/controller'  
class Post < Sequel::Model(DB)  
  class Show
  #...
```

It would be awesome right now to run a console and try creating post or two. We have a gem for that too, it's called [rack-console](https://github.com/davidcelis/rack-console). Add it to **Gemfile** and run **bundle**. You can run a console via **bundle exec rack-console** command. To create a test post execute the following code in console:

```
Post.create(title: 'I can live without Rails', content: 'Or can\'t I?', created_at: Time.now)
```

Now we can output post data from database in **Show** action

```ruby
# app/controllers/post/show.rb  
require 'hanami/controller'  
class Post < Sequel::Model(DB)  
  class Show  
    include ::Hanami::Action  
    def call(params)  
      post = Post[params[:id]]  
      self.body = "<h1>#{post.title}</h1> <p>#{post.content}</p>"  
    end  
  end  
end
```

If you are ready to be amazed than jump to [http://localhost:3000/post/1](http://localhost:3000/post/1) and say it loudly "I am no more framework addicted!" :)

### Cells

Remember at the beginning of the article I wished 3 things to come true? Well, 2 of them we already did. It's time to separate page HTML code to files (Views). To do this I propose to use [cells](https://github.com/trailblazer/cells) gem, which implements View Model pattern. In other words it allows to create a special components to render you view or a part of it.

```ruby
# Gemfile  
#...  
gem 'cells'  
gem 'cells-erb'  
```

Gem **cells** can work with many template engines, I used the standard **cells-erb**. Cells are going to be stored in **app/cells** folder. And again - no magical autoloading

```ruby
# config.ru  
#...  
# Load cells  
Dir[File.join(File.dirname(__FILE__), 'app/cells', '\*\*', '\*.rb')].sort.each {|file| require file }
```

Let's create 2 cells. First one for page general html layout and second to render a post. Let's start with layout

```ruby
# app/cells/layout_cell.rbclass LayoutCell < Cell::ViewModel  
  include ::Cell::Erb  
    def show(&block)  
    render(&block)  
  end  
end
```

I overrided **show** method for **LayoutCell** to make it accept block and render its content. A view for cell should be stored in a folder with the same name

```html
<!-- app/cells/layout/show.erb --><html>  
<head>  
  <title>My Blog</title>  
</head>  
<body>  
  <%= yield %>  
</body>
```

Pay attention, here we render content of **show**'s block using **yield**. Now use the layout in a controller action

```ruby
# controllers/post/show.rb  
require 'hanami/controller'  
class Post < Sequel::Model(DB)  
  class Show  
    include ::Hanami::Action  
    def call(params)  
      post = Post[params[:id]]  
      render_layout "<h1>#{post.title}</h1> <p>#{post.content}</p>"  
    end  
    def render_layout(content = '')  
      self.body = LayoutCell.new(nil).() { content }  
    end  
  end  
end
```

I created a method **render_layout**, which we can move to some **BaseAction** in the future and reuse. Each cell accepts an object (model), which it will use as a datasource. In our case its **nil,** because there is no model for html layout :)

One small piece of work left. Create a post cell 

```ruby
# app/cells/post_cell.rbclass PostCell < Cell::ViewModel  
  include ::Cell::Erb  
  property :title  
  property :content  
end
```

and a view

```html
<!-- app/cells/post/show.erb -->
<h1><%= title %></h1>  
<p><%= content %></p>
```

Hooray, it's time to call a Cell in a controller ( **PostCell.new(post)** )

```ruby
# controllers/post/show.rb  
require 'hanami/controller'  
class Post < Sequel::Model(DB)  
  class Show  
    include ::Hanami::Action  
    def call(params)  
      post = Post[params[:id]]  
      render_layout PostCell.new(post)  
    end  
    def render_layout(content = '')  
      self.body = LayoutCell.new(nil).() { content }  
    end  
  end  
end
```

### Conclusion

The full code of the application we developed you can find [here](https://github.com/sergey-koba-mobidev/blog/tree/blog-article).

There are a lot of cool tips and tricks I left behind in this article. But I'll give you several directions:

1.  Automatically rerun you application each time you change a code using [rerun](https://github.com/alexch/rerun/) gem.
2.  Create your own rake using ruby command line utility gem called [thor](https://github.com/erikhuda/thor).
3.  And at last but not at least use awesome advanced API for Service Objects called [Trailblazer](http://trailblazer.to/).

If you have any qestions or advices, please write me or take a look at [**this blog code, where I developed all listed features**](https://github.com/sergey-koba-mobidev/blog).

And special thanks to [Anna](https://www.linkedin.com/in/anna-koba-288263120/) for the nice doodle :D

