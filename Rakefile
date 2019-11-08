task :default => ['hello']

desc 'Test rake task'
task :hello do
  p 'Hello from Rake!'
end

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