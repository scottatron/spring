# Spring

[![Build Status](https://travis-ci.org/jonleighton/spring.png?branch=master)](https://travis-ci.org/jonleighton/spring)

Spring is a Rails application preloader. It speeds up development by
keeping your application running in the background so you don't need to
boot it every time you run a test, rake task or migration.

## Features

* Totally automatic; no need to explicitly start and stop the background process
* Reloads your application code on each run
* Restarts your application when configs / initializers / gem
  dependencies are changed

## Compatibility

* Ruby versions: MRI 1.9.3, MRI 2.0.0
* Rails versions: 3.2, 4.0

Spring makes extensive use of `Process#fork`, so won't be able to
provide a speed up on platforms which don't support forking (Windows, JRuby).

## Walkthrough

Either `gem install spring` or add it to your Gemfile:

``` ruby
group :development do
  gem "spring"
end
```

Spring is designed to be used *without* bundle exec, so use `spring
[command]` rather than `bundle exec spring [command]`.

For this walkthrough I've generated a new Rails application, and run
`rails generate scaffold posts name:string`.

Let's run a test:

```
$ time spring rake test test/functional/posts_controller_test.rb
Run options:

# Running tests:

.......

Finished tests in 0.127245s, 55.0121 tests/s, 78.5887 assertions/s.

7 tests, 10 assertions, 0 failures, 0 errors, 0 skips

real    0m2.165s
user    0m0.281s
sys     0m0.066s
```

That wasn't particularly fast because it was the first run, so spring
had to boot the application. It's now running:

```
$ spring status
Spring is running:

26150 spring server | spring-demo-app | started 3 secs ago
26155 spring app    | spring-demo-app | started 3 secs ago | test mode
```

The next run is faster:

```
$ time spring rake test test/functional/posts_controller_test.rb
Run options:

# Running tests:

.......

Finished tests in 0.176896s, 39.5714 tests/s, 56.5305 assertions/s.

7 tests, 10 assertions, 0 failures, 0 errors, 0 skips

real    0m0.610s
user    0m0.276s
sys     0m0.059s
```

Writing `spring` before every command gets a bit tedious. Spring binstubs solve this:

```
$ spring binstub rake rails
```

This will generate `bin/rake` and `bin/rails`. They
replace any binstubs that you might already have in your `bin/`
directory. Check them in to source control.

If you don't want to prefix every command you type with `bin/`, you
can [use direnv](https://github.com/zimbatm/direnv) to automatically add
`./bin` to your `PATH` when you `cd` into your application.

If we edit any of the application files, or test files, the changes will
be picked up on the next run without the background process having to
restart. This works in exactly the same way as the code reloading
which allows you to refresh your browser and instantly see changes during
development.

But if we edit any of the files which were used to start the application
(configs, initializers, your gemfile), the application needs to be fully
restarted. This happens automatically.

Let's "edit" `config/application.rb`:

```
$ touch config/application.rb
$ spring status
Spring is running:

26150 spring server | spring-demo-app | started 36 secs ago
26556 spring app    | spring-demo-app | started 1 sec ago | test mode
```

The application detected that `config/application.rb` changed and
automatically restarted itself.

If we run a command that uses a different environment, then that
environment gets booted up:

```
$ bin/rake routes
    posts GET    /posts(.:format)          posts#index
          POST   /posts(.:format)          posts#create
 new_post GET    /posts/new(.:format)      posts#new
edit_post GET    /posts/:id/edit(.:format) posts#edit
     post GET    /posts/:id(.:format)      posts#show
          PUT    /posts/:id(.:format)      posts#update
          DELETE /posts/:id(.:format)      posts#destroy

$ spring status
Spring is running:

26150 spring server | spring-demo-app | started 1 min ago
26556 spring app    | spring-demo-app | started 42 secs ago | test mode
26707 spring app    | spring-demo-app | started 2 secs ago | development mode
```

There's no need to "shut down" spring. This will happen automatically
when you close your terminal. However if you do want to do a manual shut
down, use the `stop` command:

```
$ spring stop
Spring stopped.
```

## Commands

The following commands are shipped by default.

Custom commands can be specified in the Spring config file. See
[`lib/spring/commands/`](https://github.com/jonleighton/spring/blob/master/lib/spring/commands/)
for examples.

You can add the following gems to your Gemfile for additional commands:

* [spring-commands-rspec](https://github.com/jonleighton/spring-commands-rspec)
* [spring-commands-cucumber](https://github.com/jonleighton/spring-commands-cucumber)
* [spring-commands-testunit](https://github.com/jonleighton/spring-commands-testunit) - useful for
  running `Test::Unit` tests on Rails 3, since only Rails 4 allows you
  to use `rake test path/to/test` to run a particular test/directory.

### `rake`

Runs a rake task. Rake tasks run in the `development` environment by
default. You can change this on the fly by using the `RAILS_ENV`
environment variable. The environment is also configurable with the
`Spring::Commands::Rake.environment_matchers` hash. This has sensible
defaults, but if you need to match a specific task to a specific
environment, you'd do it like this:

``` ruby
Spring::Commands::Rake.environment_matchers["perf_test"] = "test"
Spring::Commands::Rake.environment_matchers[/^perf/]     = "test"

# To change the environment when you run `rake` with no arguments
Spring::Commands::Rake.environment_matchers[:default] = "development"
```

### `rails console`, `rails generate`, `rails runner`

These execute the rails command you already know and love. If you run
a different sub command (e.g. `rails server`) then spring will automatically
pass it through to the underlying `rails` executable (without the
speed-up).

## Configuration

Spring will read `~/.spring.rb` and `config/spring.rb` for custom
settings. Note that `~/.spring.rb` is loaded *before* bundler, but
`config/spring.rb` is loaded *after* bundler. So if you have any
`spring-commands-*` gems installed that you want to be available in all
projects without having to be added to the project's Gemfile, require
them in your `~/.spring.rb`.

### Application root

Spring must know how to find your Rails application. If you have a
normal app everything works out of the box. If you are working on a
project with a special setup (an engine for example), you must tell
Spring where your app is located:

```ruby
Spring.application_root = './test/dummy'
```

### Running code before forking

There is no `Spring.before_fork` callback. To run something before the
fork, you can place it in `~/.spring.rb` or `config/spring.rb` or in any of the files
which get run when your application initializers, such as
`config/application.rb`, `config/environments/*.rb` or
`config/initializers/*.rb`.

For example, if loading your test helper is slow, you might like to
preload it to speed up your test runs. To do this you could put a
`require Rails.root.join("test/helper")` in
`config/environments/test.rb`.

### Running code after forking

You might want to run code after Spring forked off the process but
before the actual command is run. You might want to use an
`after_fork` callback if you have to connect to an external service,
do some general cleanup or set up dynamic configuration.

```ruby
Spring.after_fork do
  # run arbitrary code
end
```

If you want to register multiple callbacks you can simply call
`Spring.after_fork` multiple times with different blocks.

### Watching files and directories

Spring will automatically detect file changes to any file loaded when the server
boots. Changes will cause the affected environments to be restarted.

If there are additional files or directories which should trigger an
application restart, you can specify them with `Spring.watch`:

```ruby
Spring.watch "spec/factories"
```

By default Spring polls the filesystem for changes once every 0.2 seconds. This
method requires zero configuration, but if you find that it's using too
much CPU, then you can turn on event-based file system listening:

```ruby
Spring.watch_method = :listen
```

You may need to add the [`listen` gem](https://github.com/guard/listen) to your `Gemfile`.

## Troubleshooting

If you want to get more information about what spring is doing, you can
specify a log file with the `SPRING_LOG` environment variable:

```
spring stop # if spring is already running
export SPRING_LOG=/tmp/spring.log
spring rake -T
```
