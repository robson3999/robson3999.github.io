---
layout: post
title: 'How I learned rake'
date: 2019-11-08 17:22:03 +0100
keywords: "ruby, rake, automation"
---

When I was using Hugo as my blogging engine I get used to automated post generator. I just opened console, typed one command and had ready-to-extend post template. 
When I switched to Jekyll, I spent maybe like 10 seconds to search for that command and - as you may expect - didn't found any. I forgot about it and kept using Jekyll, but when I wanted to write new post today something switched in my head!
```
Man, you're programmer,
why don't you write your own ruby command to create that dummy post file for you?
```
Since I'm mainly Rails developer `Rake` came to my mind at first place.

## Rakefile
I opened my `blog/` directory and created `Rakefile`. It is needed for `rake` to load commands (or tasks, as they are called in rake). I started with classic `Hello World` example, just to check how things work!

```ruby
task :default => ['hello']

desc 'Greeting from rake task'
task :hello do
  puts 'Hello World from rake task!'
end
```
and that's it. Run `rake hello` from command line and (if you have `rake` gem installed) you'll be greeted by your ruby code!

One thing needs explanation, though. That `task :default` line. It is said, that's that good practice to always set a default task, so when `rake` is invoked without argument it will fallback to the default one. So, if we put nothing after `rake`, it will greet us, isn't it nice?

## How to use arguments in rake tasks
So yeah, that was easy one. But when I want to create new post, I would like to name new file with text provided by me, not always the same, default one. So now, I had to learn how to pass arguments to task.
As you may expect, it is easy as previous example. Let's assume that we're still expanding our `Rakefile` so I won't need to repeat myself with that `task :default` code.

```ruby
desc 'Greet someone with his name!'
task :hello_dear, [:name] do |t, args|
  args.with_defaults(name: 'Dude')
  puts "Hello, #{args.name.capitalize}!"
end
```
Now, if you run `rake hello_dear` you'll see wonderful greeting:
```
Hello, Dude!
```
But to make use of that cool method, you can now run `rake hello_dear[gizmo]` and see:
```
Hello, Gizmo!
```
Yeah, we did it! We know how to use arguments in rake tasks. Let's create some files!

## Using bash commands from ruby
It may not be suprising, but... invoking bash commands from ruby is very easy. And pleasant.
To make use of all that `cd`, `touch`, `cat`, etc. you just need to surround it with '\`' sign. Check it out!

```ruby
desc 'Touch command ruby wrapper'
task :touch, [:file_name] do |t, args|
  args.with_defaults(file_name: 'new_file')
  `touch #{args.new_file}`
  puts "Created new file with name: #{args.new_file}"
end
```

And here you go, here's your file maker. Create few files with theese commands:
```
rake touch
rake touch[thats_my_file]
```
With `backticks` you can use string interpolation same as with '\"\"'. Isn't it cool?

## The final purpose of this post
So finally we got all needed skills to generate new post template with `rake`. Let's see my code!
```ruby
desc 'Create new post with given name'
task :new_post, [:post_name] do |t, args|
  args.with_defaults(post_name: 'new_post')
  time_now = Time.new.strftime('%Y-%m-%d')
  safe_post_name = args.post_name.gsub(' ', '_')
  post_name_with_timestamp = [time_now, safe_post_name].join('-') + '.md'
  post_header = [
    "---",
    "layout: post",
    "title: #{args.post_name}",
    "date: #{Time.new}",
    'keywords: ',
    "---"
  ]

  puts "Timestamp: #{time_now}"
  puts "Creating new post with file name: #{post_name_with_timestamp}"
  `touch ./_posts/#{post_name_with_timestamp}`
  
  post_header.each do |string|
    `echo #{string} >> ./_posts/#{post_name_with_timestamp}`
  end

  puts "You can find it in ./_posts/#{post_name_with_timestamp}"
end
```
Few words about that script. At first I needed timestamp, to use it in filename. Jekyll makes use of that to order posts and display them correctly. So i used built-in `Time` module and formatted it nicely, to produce Jekyll-friendly date format.

Next line, with `safe_post_name` is 'just to make sure' that the file name won't have spaces. So with this 'feature', you can pass arguments like that `rake new_post['thats new post with spaces in title']` and it won't raise any errors nor exceptions. And, what is even more cool, in `"title"` tag it will provide name with spaces, ready to display on webpage!

Next, I combine safe post name with timestamp and `.md` file extension - Jekyll uses `markdown` files, so I need to create file with that extension. Then I create post_header, which is used by Jekyll to provide proper page layout (it may be post or page or whatever I implement in future), display title, date and make use of keywords.

Actually I don't use keywords anywhere in my blog I suppose, but maybe in future it will be useful to have it. You never know!


Then, few `puts` to inform user (me) what is going on, when I run that task, and then bash command - `touch` - to create file with given filename.

And at the end, I fill that new file with `post_header` data, to finally get that ready-to-use post template file.

At the last line I just tell myself where to find new file, in case I would forget the structure of Jekyll project (which is impossible I think, it is really very straight forward).

That's it! Now you can grab that code, improve, tune to your needs and make use of that file generator made in ruby lang! Cheers!
