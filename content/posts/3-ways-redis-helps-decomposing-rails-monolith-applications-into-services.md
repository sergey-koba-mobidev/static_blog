---
title: "3 Ways Redis Helps Decomposing Rails Monolith Applications Into Services"
date: 2017-08-21T16:06:18+03:00
tags: ["ruby", "redis", "rails", "microservices"]
disqus_identifier: "3-ways-redis-helps-decomposing-rails-monolith-applications-into-services"
draft: false
---
Once in a while every developer thinks about decomposing their monster monolith Rails application into services. Very ambitious and brave thought! Why? We all know the pros of service-based architecture, but the cons make us think twice or even give up on decomposition. Here are the big fears of decomposition: single authentication, performance and complexity. Luckily we have Redis to make us brave enough again and help resolve at least some problems easily.

![](https://res.cloudinary.com/skoba/image/upload/v1503349923/ruby_jcvhp3.png)

<!--more-->

Illustration by [Anna Koba](https://dribbble.com/anna_koba).

### Shared session

Let's say you've created several services for a Rails based social network. They are:

![](https://res.cloudinary.com/skoba/image/upload/v1500661678/blog_services_otr7q7.png)

It is obvious you want users to sign in just once to access all the features of the network. This is the moment when you realize that it's time to find out how Rails stores user sessions. Usually it stores an encrypted cookie, containing all the session info. You can check **config/initializers/session_store.rb**. It has something like this in most cases:

```ruby
Rails.application.config.session_store :cookie_store, key: '_your_app_session_key'
```

So in order to make our services share the same session info of a logged in user, we should tell it to share the session cookie. To do that we should name our key the same for all services and add **domain: all** option to session_store config.

```ruby
Rails.application.config.session_store :cookie_store, key: '_your_app_shared_session_key', domain: :all
```

**Domain: :all** option tells Rails to create one session cookie for all subdomains under your domain. This way you can place services on separate subdomains, but they all will share the same session cookie. 

If you tried this approach you probably noticed nothing is working... Not a surprise! Remember I said "encrypted cookie" in the beginning of the article? It turns out Rails encrypts the cookie with a **secret_key_base** in  **config/secrets.yml**. So if this key is different across each service, then one Rails app will not be able to decrypt the cookie, encrypted by another service. The solution is to make **secret_key_base** the same across all the services...

**Well... this becomes too much to take care about :)**

The pros of the encrypted session cookie are: 

*   No additional software complexity.
*   Works out of the box.

Cons:

*   Each service should have the same **secret_key_base**. If you ever decide to change it, you'll have to update all the services at once.
*   Did you know that the cookie size is limited?
*   Session info, although encrypted, is stored on client's side.

These cons are enough for me to try something new. Let's bring up a Redis container with [Docker](http://1devblog.org/article/docker-for-rails) or use a 3rd party Redis service. Add 

```ruby
gem 'redis-session-store'
```

to your **Gemfile** and run **bundle**.

The next step is to modify your  **config/initializers/session_store.rb**

```ruby
Rails.application.config.session_store :redis_session_store, {
      key: '_your_app_shared_session_key',
      domain: :all,
      httponly: false,
      serializer: :hybrid, #transparently migrate existing Marshal cookie values to JSON
      redis: {
          key_prefix: 'myapp:session:',
          url: 'redis://URL_TO_YOUR_REDIS',
      }
}
```

So now all your services will share the cookie with a session-unique ID in Redis, **key_prefix** option says Redis to store session keys with the specified prefix, url option speaks for itself :)

Pros of the Redis-stored session:

*   No data is stored on the client side.
*   You can access session info not only from the browser, but directly accessing Redis from any service.
*   No size limit.
*   No need for secret_key_base.

Cons: now your app depends on Redis :D

### Shared cache

Let's say your News Feed service needs to display a list of 20 recent friend updates. Next to each update it displays username and avatar, but detailed user info is stored in another service and we have only unique user identifiers (uuid) in News Feed service.

So what are we usually doing? Right, we are requesting user info from another service. The dirty way to do this can look like this:

```ruby
require 'open-uri'
require 'json'

uuid = '558fe613938306f4a6791b7e72ed0b1bc585a836927a41adf9a9b2e4d15ac3b9'
user_data = JSON.parse(open("https://my-users-service.com/users/#{uuid}").read)
```

So if we have 20 records in our news feed we will make 20 additional https requests while displaying user-related data. Sounds like a **preformance trap**!!!

I can think of adding an API endpoint to request multiple user data at once... In batches... But again  **this becomes too much to take care about :)** You may start thinking I am a lazy person. Well... it's true, but is it bad for a developer?

Anyway, let Redis help us and take care of performance. Let's cache our time consuming https requests with **Rails.cache** configured to use Redis. Add the following gems and run a bundle:

```ruby
gem 'readthis'
gem 'hiredis'
```

Configure Rails.cache to use Redis

```ruby
# config/initializers/cache_store.rb
Rails.application.config.cache_store = :readthis_store, {
        namespace: 'cache',
        redis: {url: 'redis://URL_TO_YOUR_REDIS', driver: :hiredis}
}
```

Now we can wrap your API calls with **Rails.cache.fetch**

```ruby
user_data = Rails.cache.fetch("user_data_#{uuid}") do
  JSON.parse(open("https://my-users-service.com/users/#{uuid}").read)
end
```

Now each time you request user_data Rails checks if there is a cached version in Redis. If there is one you will quickly get it back. If there is none Rails will perform https request and cache the answer inside Redis with  **"user_data_#{uuid}"** key.

Great! Our service-based app is fast enough again. But wait... how come my friend has changed his avatar and our news feed shows the old one? That's because it shows the cached version of user data. The cache should be invalidated each time the user data changes.

But how will News Feed service know when user data is invalid? It shouldn't. Because we have shared the Redis cache it could be invalidated from another service, which takes care of user profile updates. It's easy, just call:

```ruby
uuid = '558fe613938306f4a6791b7e72ed0b1bc585a836927a41adf9a9b2e4d15ac3b9'
Rails.cache.delete("user_data_#{uuid}")
```

In other words you should delete cached data identified by  **"user_data_#{uuid}"** key each time User model is updated. I like DDD and [Trailblazer](http://trailblazer.to/) so I am not saying you should do this in User model after_save callback ;)

Pros of Redis-stored cache:

*   Improves app performance by caching http(s) requests.
*   It is shared across services and can be accessed from any service (even not Rails based).
*   Easy Rails integration.

Cons as usual: now your app depends on Redis :D

### Shared PubSub

Remember we "created" Notifications service for our virtual social network? We did it on purpose! We want our service to receive events from other services and send notifications to users.

We can create an API for that, or in other words, webhook endpoint, which will be called when a service wants to send an event to the Notifications service. Something like this:

![](https://res.cloudinary.com/skoba/image/upload/v1500662524/webhook_rxhzzb.png)

Imagine we have several services that need to notify each other...

![](https://res.cloudinary.com/skoba/image/upload/v1500662703/webhooks_many_vlqurj.png)

Imagine we have dozens or hundreds of services! An they all need to know where to send webhooks!

Well... **This becomes too much to take care about :)**

The design pattern I like, which solves the problem, is called **publisher/subscriber** and the appropriate architectural concept for that is a **message broker**.

![](https://res.cloudinary.com/skoba/image/upload/v1500663105/services_message_broker_l7xjyt.png)

When a service wants to share the event, it sends this event to the message broker (publish). When another service wants to receive a certain type of event, it subscribes for updates from the message broker (subscribe). And all services should know only where the message broker is located. Now it's a centralized and shared place to exchange events and other information.

Guess what? Redis can help us and act as a message broker too!

Let's say you want to share an event from the News Feed service with other services. We already installed all required gems, so here is the code:

```ruby
message = {
  event: 'new_photo',
  url: 'http://my-social-network.com/photos/1.jpg'
}
redis = Redis.new(url: 'redis://URL_TO_YOUR_REDIS')
redis.publish 'news_feed_channel', message.to_json
```

That's it? Yes, we call publish with 2 params: 1 - the channel name where to publish message, 2 - the actual message.

Let's write a service rake task, which subscribes to Redis messages.

```ruby
namespace :redis do
  task :subscribe => :environment do
    redis = Redis.new(url: 'redis://URL_TO_YOUR_REDIS')
    redis.subscribe('news_feed_channel') do |on|
         on.message do |channel, message|
           message_data = JSON.parse(message)
           # Send notifications
           # ...
         end
    end
  end
end
```

**redis.subscribe** is an infinite loop listening for updates from Redis. You should run this rake task as a separate process/daemon. Here is a **Heroku Procfile** snippet:

```
pubsub: bundle exec rake redis:subscribe
```

Probably you'll want to add retries and logging to this rake task in the future.

### Instead of Conclusion

Hope after this article you will reconsider decomposing your monster monolith Rails application :) It's not so hard with the help of Redis!