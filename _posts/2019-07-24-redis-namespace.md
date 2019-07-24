---
layout: post
title: "redis-namespace: when one sidekiq process is not enough"
date: 2019-07-24T19:54:27+02:00
keywords: "sidekiq, redis, ruby, rails"
---
Few months ago I was working in a project in which we had two applications communicating with each other. 
Each of them had to has its own sidekiq process, but they were deployed to one server. After some time, I realized that after deploying one app, the second one has problems with invoking its workers. The problem was that sidekiq was restarting and loading workers only from one (last deployed) app. So after each deploy I had many errors saying that `NameError: uninitialized constant AppOneWorker` and another time `NameError: uninitialized constant AppTwoWorker`. With rescue came gem called `redis-namespace`.

### [redis-namespace](https://github.com/resque/redis-namespace)

As pointed in it's repo description, `redis-namespace`:
```
...adds a Redis::Namespace class which can be used to namespace Redis keys. 
```
What does it mean? It means that we can have plenty of apps using one Redis server, with many sidekiq processes not conflicting with each other.

All you need to do is to add `redis-namespace` to your Gemfile and configure it in `app/config/initializers/sidekiq.rb` like this:

```
# frozen_string_literal: true

require 'sidekiq'

Sidekiq.configure_server do |config|
  config.redis = {
    url: 'redis://127.0.0.1:6379',
    namespace: 'my_first_app_name'
  }
end

Sidekiq.configure_client do |config|
  config.redis = {
    url: 'redis://127.0.0.1:6379',
    namespace: 'my_first_app_name'
  }
end
```
And do the same for the second app (with its proper name, of course).  

After that, when you visit your app's `/sidekiq` route (if you mounted it in `routes.rb`), you can see that sidekiq process has namespace added at the beginning of it's name.
Other way to check if everything is OK is to launch `redis-cli` in command line, and look up for keys containing your namespace.
```
redis 127.0.0.1:6379> keys *app_name*
 1) "my_first_app_name:queues"
 2) "my_second_app_name:queues"
```

### Credits
When I had that problem, I wouldn't resolve it without [Maciej Mensfeld's post](https://mensfeld.pl/2014/07/multiple-sidekiq-processes-for-multiple-railssinatra-applications-namespacing/) about redis-namespace. Thanks!  
You can even find my enthusiastic comment there, I left back then, haha.