unicorn-heroku-config
=====================

Unofficial Unicorn web server configurations, optimized for running on Heroku.

This is a living document and notes, addendums and clarifications are all welcome.



Required: add and configure unicorn
-----------------------------------

You need to add unicorn to your Gemfile, `bundle install`, and add these files:

* Procfile
* config/unicorn.rb


Required: Rails logging fix
---------------------------

Also allows setting LOG_LEVEL config var to tune on the fly

**config/environments/production.rb**:

    # We're on Heroku, just output straight to STDOUT
    # This is required because we're using Unicorn too: https://github.com/ryanb/cancan/issues/511#issuecomment-3643266
    config.logger = Logger.new(STDOUT)
    config.logger.level = Logger.const_get(ENV['LOG_LEVEL'] ? ENV['LOG_LEVEL'].upcase : 'INFO')



Optional: timeout tuning (UNICORN_TIMEOUT)
-------------------------

Heroku times out at 30s but you can set this lower to
(a) keep slow-running requests from bogging up your app. Twitter, for example, uses a 5s unicorn timeout
(b) prefer unicorns timeout handling to Heroku router's (of which I don't know many details)


Optional: worker count tuning (UNICORN_WORKERS)
------------------------------

Typically one Heroku instance can run 2-4 unicorn worker processes per-dyno, easily multiplying your concurrency.

Heroku currently allow up to 500MB in RAM usage per-dyno before it starts swapping.

A typical medium-sized Rails app uses around 100MB at launch and often grows to 200MB+ after serving a number of requests, so
start at 4, monitor for [H14 Memory Alerts](), and tune down as required.

Sinatra and Rack apps can run significantly more workers without issue. 6+ is not uncommon

Optional: backlog tuning (UNICORN_BACKLOG)
-------------------------

Per Unicorn docs:

  This is the backlog of the listen() syscall...
  If you are running unicorn on multiple machines, lowering this number can help your load balancer detect when a machine is overloaded and give requests to a different machine.
  Default: 1024

In other words, each unicorn master process is locally queueing up requests before starting to serve up 502 Bad Gateway errors. Setting this much lower allows the unicorn process
to indicate to Heroku that this dyno is overloaded and the request should go elsewhere (although specifics of Heroku load balancing behavior [are a contentious subject](http://rapgenius.com/James-somers-herokus-ugly-secret-lyrics))


Optional: preload_app (requires manual configuration)
-------------------------------------------

Per the [Unicorn docs](http://unicorn.bogomips.org/Unicorn/Configurator.html#method-i-preload_app):

  Allows memory savings when using a copy-on-write-friendly GC but can cause bad things to happen when resources like sockets are opened at

  This allows worker processes to start up a lot faster, but requires manually opening and reopening all connections (postgres, redis, newrel
  Not doing so will cause a lot of unexpected issues.

Pros:
* faster worker starts - e.g. on initial load, request timeout, or USR2 rolling restarts (which can't used on Heroku)
* better memory usage?

Cons:
* easy to screw up connection handling and bad things happen
* most apps don't really restart workers much so this has no benefit

In our default unicorn.rb configuration this is disabled to avoid issues. If you are using a fast timeout you might want to look into this.


