<p align="center">
  <a href="http://moove-it.github.io/sidekiq-scheduler/">
    <img src="https://moove-it.github.io/sidekiq-scheduler/images/logo.png" alt="Sidekiq Scheduler" />
  </a>
</p>

<p align="center">
  <a href="https://badge.fury.io/rb/sidekiq-scheduler">
    <img src="https://badge.fury.io/rb/sidekiq-scheduler.png" alt="Gem Version">
  </a>
  <a href="https://codeclimate.com/github/moove-it/sidekiq-scheduler">
    <img src="https://codeclimate.com/github/moove-it/sidekiq-scheduler/badges/gpa.svg" alt="Code Climate">
  </a>
  <a href="https://travis-ci.org/moove-it/sidekiq-scheduler">
    <img src="https://api.travis-ci.org/moove-it/sidekiq-scheduler.svg?branch=master" alt="Build Status">
  </a>
  <a href="https://coveralls.io/github/moove-it/sidekiq-scheduler?branch=master">
    <img src="https://coveralls.io/repos/moove-it/sidekiq-scheduler/badge.svg?branch=master&service=github" alt="Coverage Status">
  </a>
  <a href="https://inch-ci.org/github/moove-it/sidekiq-scheduler">
    <img src="https://inch-ci.org/github/moove-it/sidekiq-scheduler.svg?branch=master" alt="Documentation Coverage">
  </a>
  <a href="http://www.rubydoc.info/github/moove-it/sidekiq-scheduler">
    <img src="https://img.shields.io/badge/yard-docs-blue.svg" alt="Documentation">
  </a>
</p>

sidekiq-scheduler is an extension to [Sidekiq](http://github.com/mperham/sidekiq) that adds support
for running scheduled jobs, which are like cron jobs, recurring on a regular basis.

## Installation

Add this to your Gemfile:

```ruby
gem 'sidekiq-scheduler', '~> 2.0'
  ```

If you are using Rails you are set.

If you are not using Rails create a file with this content:

```ruby
require 'sidekiq-scheduler'
  ```

and then execute:

```sh
sidekiq -r created_file_path.rb
  ```

Look at [Loading the schedule](https://github.com/moove-it/sidekiq-scheduler/#loading-the-schedule)
for information on how to load your schedule.

You can add sidekiq-scheduler configuration options to `sidekiq.yml` config file.
Available options are:

``` yaml
:schedule: <the schedule to be run>
:dynamic: <if true the schedule can be modified in runtime [false by default]>
:enabled: <enables scheduler if true [true by default]>
:scheduler:
  :listened_queues_only: <push jobs whose queue is being listened by sidekiq [false by default]>
```

## Manage tasks from Unicorn/Rails server

If you want start sidekiq-scheduler only from Unicorn/Rails, but not from Sidekiq you can have
something like this in an initializer:

```ruby
# config/initializers/sidekiq_scheduler.rb
require 'sidekiq/scheduler'

puts "Sidekiq.server? is #{Sidekiq.server?.inspect}"
puts "defined?(Rails::Server) is #{defined?(Rails::Server).inspect}"
puts "defined?(Unicorn) is #{defined?(Unicorn).inspect}"

if Rails.env == 'production' && (defined?(Rails::Server) || defined?(Unicorn))
  Sidekiq.configure_server do |config|

    config.on(:startup) do
      Sidekiq.schedule = YAML
        .load_file(File.expand_path('../../../config/scheduler.yml', __FILE__))
      Sidekiq::Scheduler.reload_schedule!
    end
  end
else
  Sidekiq::Scheduler.enabled = false
  puts "Sidekiq::Scheduler.enabled is #{Sidekiq::Scheduler.enabled.inspect}"
end
```

## Scheduled Jobs (Recurring Jobs)

Scheduled (or recurring) jobs are logically no different than a standard cron
job. They are jobs that run based on a fixed schedule which is set at
startup.

The schedule is a list of Sidekiq worker classes with arguments and a
schedule frequency (in crontab syntax). The schedule is just a Hash, being most likely
stored in a YAML like so:

``` yaml
schedule:
  CancelAbandonedOrders:
    cron: "*/5 * * * *"

  queue_documents_for_indexing:
    cron: "0 0 * * *"
    # you can use rufus-scheduler "every" syntax in place of cron if you prefer
    # every: 1h

    # By default the job name (Hash key) will be taken as worker class name.
    # If you want to have a different job name and class name, provide the 'class' option
    class: QueueDocuments

    queue: high
    args:
    description: "This job queues all content for indexing in solr"

  clear_leaderboards_contributors:
    cron: "30 6 * * 1"
    class: ClearLeaderboards
    queue: low
    args: contributors
    description: "This job resets the weekly leaderboard for contributions"
```

You can provide options to `every` or `cron` via an Array:

``` yaml
schedule:
  clear_leaderboards_moderator:
    every: ["30s", :first_in => '120s']
    class: CheckDaemon
    queue: daemons
    description: "This job will check Daemon every 30 seconds after 120 seconds after start"
```


NOTE: Six parameter cron's are also supported (as they supported by
rufus-scheduler which powers the sidekiq-scheduler process).  This allows you
to schedule jobs per second (ie: `30 * * * * *` would fire a job every 30
seconds past the minute).

A big shout out to [rufus-scheduler](http://github.com/jmettraux/rufus-scheduler)
for handling the heavy lifting of the actual scheduling engine.


### Loading the schedule

Let's assume your scheduled jobs are defined in a file called `config/scheduler.yml` under your Rails project,
you could create a Rails initializer called `config/initializers/scheduler.rb` which would load the job definitions:

```ruby
require 'sidekiq/scheduler'

Sidekiq.configure_server do |config|
  config.on(:startup) do
    Sidekiq.schedule = YAML.load_file(File.expand_path("../../../config/scheduler.yml",__FILE__))
    Sidekiq::Scheduler.reload_schedule!
  end
end
```

If you are running a non Rails project you should add code to load the workers classes before loading the schedule.

```ruby
require 'sidekiq/scheduler'
Dir[File.expand_path('../lib/workers/*.rb',__FILE__)].each do |file| load file; end

Sidekiq.configure_server do |config|
  config.on(:startup) do
    Sidekiq.schedule = YAML.load_file(File.expand_path("../../../config/scheduler.yml",__FILE__))
    Sidekiq::Scheduler.reload_schedule!
  end
end
```

You can also put your schedule information inside sidekiq.yml and load it with:

```sh
sidekiq -C ./config/sidekiq.yml
```

#### The Spring preloader and Testing your initializer via Rails console

If you're pulling in your schedule from a YML file via an initializer as shown, be aware that the Spring application preloader included with Rails will interefere with testing via the Rails console.

**Spring will not reload initializers** unless the initializer is changed.  Therefore, if you're making a change to your YML schedule file and reloading Rails console to see the change, Spring will make it seem like your modified schedule is not being reloaded.

To see your updated schedule, be sure to reload Spring by stopping it prior to booting the Rails console.

Run `spring stop` to stop Spring.

For more information, see [this issue](https://github.com/Moove-it/sidekiq-scheduler/issues/35#issuecomment-48067183) and [Spring's README](https://github.com/rails/spring/blob/master/README.md).

### Reloading the schedules

Schedules are stored in Redis. To add / update an schedule, you have to:

```ruby
Sidekiq.set_schedule('heartbeat', { 'every' => ['1m'], 'class' => 'HeartbeatWorker' })
```

When `:dynamic` flag is set to `true`, schedule changes are loaded every 5 seconds.

You can set that flag in the following ways.

- YAML configuration:

```
:dynamic: true
```

- Initializer configuration:

```ruby
Sidekiq.configure_server do |config|
  # ...

  config.on(:startup) do
    # ...
    Sidekiq::Scheduler.dynamic = true
  end
end
```

If `:dynamic` flag is set to false, you have to reload the schedule manually in sidekiq
side:

```ruby
Sidekiq::Scheduler.reload_schedule!
```

If the schedule did not exist it will we created, if it existed it will be updated.

### Testing

In your tests you can check that a schedule change has been set you have to:

```ruby
require 'sidekiq'
require 'sidekiq-scheduler'
require 'sidekiq-scheduler/test'

Sidekiq.set_schedule('some_name', { 'every' => ['1m'], 'class' => 'HardWorker' })

Sidekiq::Scheduler.schedules
#  => { 'every' => ['1m'], 'class' => 'HardWorker' }

Sidekiq::Scheduler.schedules_changed
#  => ['every']
```


### Time zones

Note that if you use the cron syntax, this will be interpreted as in the server time zone
rather than the `config.time_zone` specified in Rails.

You can explicitly specify the time zone that rufus-scheduler will use:

``` yaml
cron: "30 6 * * 1 Europe/Stockholm"
```

Also note that `config.time_zone` in Rails allows for a shorthand (e.g. "Stockholm")
that rufus-scheduler does not accept. If you write code to set the scheduler time zone
from the `config.time_zone` value, make sure it's the right format, e.g. with:

``` ruby
ActiveSupport::TimeZone.find_tzinfo(Rails.configuration.time_zone).name
```

A future version of sidekiq-scheduler may do this for you.

### Sidekiq Web Integration

SidekiqScheduler provides an extension to the Sidekiq web interface that adds a Recurring Jobs page.

To use it, set up the Sidekiq web interface according to the Sidekiq documentation and then add the `sidekiq-scheduler/web` require:

``` ruby
require 'sidekiq/web'
require 'sidekiq-scheduler/web'
```

## Note on Patches / Pull Requests

* Fork the project.
* Make your feature addition or bug fix.
* Add tests for it. This is important so it won't break it in a future version unintentionally.
* Commit, do not mess with version, or history.
  (if you want to have your own version, that is fine but bump version in a commit by itself so it can be ignored when merging)
* Send a pull request. Bonus points for topic branches.

## Credits

This work is a partial port of [resque-scheduler](https://github.com/bvandenbos/resque-scheduler) by Ben VandenBos.
Modified to work with the Sidekiq queueing library by Morton Jonuschat.
Scheduling of recurring jobs has been added to v0.4.0, thanks to [Adrian Gomez](https://github.com/adrian-gomez).

## License

MIT License

## Copyright

Copyright 2013 - 2016 Moove-IT
Copyright 2012 Morton Jonuschat
Some parts copyright 2010 Ben VandenBos
