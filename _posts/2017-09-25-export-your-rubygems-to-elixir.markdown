---
layout: post
title:  "Export Your RubyGems to Elixir"
date:   2017-09-24 11:00:54 -0500
categories: programming
tags: ruby rubygems elixir export erlport
---

Ruby in Elixir?! Yes, that's right you can have the best of both worlds. The Elixir ecosystem is growing by leaps and bounds, but there are some RubyGems that are well developed and don't have a comparable Hex package. If you find yourself needing some functionality from a gem, but can't find a comparable Hex package or don't have the time write the functionality in Elixr, then [Export](https://github.com/fazibear/export) can help.

Export is just a handy wrapper around the Erlang [ErlPort](http://erlport.org/) package. I don't want to go into too much detail around `ErlPort` since there's better resources on the topic. Basically `ErlPort` starts a Ruby (or Python) process which can send and receive messages via Erlang Ports.

Alright, I can see how this might be controversial, but there's a time and a place for everything. In my case I was working on porting over a major API endpoint over to Elixir, but coudln't find a good subsitution for the `IceCube` gem. 

My first approach was to try to use Ruby to do the scheduling logic; which was already well tested in Ruby, then send the IceCube data to the Elixir microservice (OTP Application). Long story short, I decided against that approach because it would require a good deal of rework on the Ruby side, and made it harder to port the existing logic to Elixir. Also, I discovered `Export` after stumbling across these blogs:

* [Elixir, Ruby, don’t fight. Talk… with Export/Erlport](https://medium.com/@Stephanbv/ruby-code-in-elixir-project-97614a9543d)
* [Ruby code in Elixir project](https://medium.com/@Stephanbv/ruby-code-in-elixir-project-97614a9543d)
 
While those blogs are great resources to get started, there really wasn't much there to help with how to actually use a Rubygem, bundler, Gemfile, etc. Bundler is pretty much an essential tool of Rubygem management. Almost every host/deployment strategy uses bundler, and my situation was no exception. In the rest of this post, I'll walk through an example of how I got `export` to use bundler via `bundle exec`.

Create a new Elixir application

{% highlight shell %}
$ mix new export_ruby
$ cd export_ruby
{% endhighlight %}


Add Export as a dependency
{% highlight elixir %}
# mix.exs

defp deps do
  [
    {:export, "~> 0.1.0"}
  ]
end
{% endhighlight %}

Fetch dependencies

{% highlight shell %}
$ mix deps.get
{% endhighlight %}

Create a directory for the ruby code to live in. `priv` is ideal since it's included in the compilation process by default.

{% highlight shell %}
$ mkdir -p priv/ruby
{% endhighlight %}

The steps above are standard affair, and resemble those other blog posts. This is the part where I start to diverge. 

Let's say you need to use the `IceCube` gem like I did. Since `priv/ruby` is where all the ruby goodness goes...let's create a `Gemfile` for `bundler`.

{% highlight shell %}
$ cd priv/ruby
$ gem install bundler
$ bundler init
{% endhighlight %}

After adding `IceCube` and `ActiveSupport` to the Gemfile bundler created it looks like what I have below. Note: I added `ActiveSupport` because of the utilities it provides in parsing Dates and Times.

{% highlight ruby %}
# export_ruby/priv/ruby/Gemfile
# frozen_string_literal: true
source "https://rubygems.org"

git_source(:github) {|repo_name| "https://github.com/#{repo_name}" }

# Needed for Date Time extensions
gem "activesupport", "4.2.9"

# handles repeated events / schedules.
gem "ice_cube", git: "https://github.com/seejohnrun/ice_cube.git", ref: "full_tz"
{% endhighlight %}

Your terminal shell should already be in `export_ruby/priv/ruby` then we need to get the gems.

{% highlight shell %}
$ bundle install
{% endhighlight %}

Note that I'm using a specific branch of `IceCube` that is fetched from Github via bundler. At the time of this writing, we needed full timezone support and the changes needed were not yet in a released version.


Now it's time to get coding. We're going to build a `IceCube::Schedule` to get series of dates and times. That means we need a ruby file to execute the `IceCube` code.

{% highlight ruby %}
# export_ruby/priv/ruby/schedule.rb

# frozen_string_literal: true

require "ice_cube"
require "active_support/time"

def daily(start_time, days)
  # time should be a string. needs to be a Time/DateTime object for IceCube.
  start_time = Time.parse(start_time)

  schedule = IceCube::Schedule.new(start_time) do |s|
    s.add_recurrence_rule(IceCube::Rule.daily.count(days))
  end

  # complex objects don't serialize well via ports. 
  # best to use simple objects (strings, integers, etc).
  schedule.all_occurrences.map(&:to_s)
end
{% endhighlight %}

While in the `export_ruby/priv/ruby` directory, use bundler to install the gem dependencies.
{% highlight shell %}
$ bundle install
Using i18n 0.8.6
Using minitest 5.10.3
Using thread_safe 0.3.6
Using bundler 1.15.4
Using ice_cube 0.16.2 from https://github.com/seejohnrun/ice_cube.git (at full_tz@10ed1e4)
Using tzinfo 1.2.3
Using activesupport 4.2.9
{% endhighlight %} 

You should see `ice_cube` installed with `bundle list`, but not when you run `gem list`. This is an important distinction. That's because bundler is managing the dependency from Github, and the gem install installed in a different path from the system gems.


Since this is an Elixir application, we need Elixir code to call the Ruby code.

{% highlight elixir %}
# export_ruby/lib/schedule.ex

defmodule ExportRuby.Schedule do
  use Export.Ruby

  # Path to ruby files relative to the project root
  @ruby_lib Application.app_dir(:export_ruby, "priv/ruby")

  def daily(start_time, days) do
    # passing simple data types
    arguments = [to_string(start_time), days]

    # Note I'm using `start_link` vs `start` as other examples show.
    # Ensures the ruby process is cleaned up after parent process stops or crashes.
    # This example is not the most efficient technique as this starts and 
    # stops a ruby process on EVERY function call, which works but starting and 
    # stopping the process each time adds overhead.
    {:ok, ruby} = Ruby.start_link(ruby_lib: @ruby_lib)

    Ruby.call(ruby, "schedule", "daily", arguments)
  end
end
{% endhighlight %}

Now that all the parts are setup, let's give it a try. Remember to pass `-S mix` to iex.

{% highlight shell %}
$ iex -S mix
{% endhighlight %}


{% highlight iex %}
iex(1)> start_time = DateTime.utc_now

iex(2)> ExportRuby.Schedule.daily(start_time, 2)
  
iex(3)> ExportRuby.Schedule.daily(DateTime.utc_now, 2)
** (ErlangError) erlang error: {:ruby, :LoadError, "cannot load such file -- ice_cube", ["-e:1:in `<main>'", "/Users/scotthamilton/.rvm/rubies/ruby-2.4.1/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb:55:in `require'", "/Users/scotthamilton/.rvm/rubies/ruby-2.4.1/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb:55:in `require'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/cli.rb:94:in `<top (required)>'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/cli.rb:41:in `main'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/erlang.rb:138:in `start'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/erlang.rb:194:in `_receive'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/erlang.rb:234:in `call_with_error_handler'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/erlang.rb:195:in `block in _receive'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/erlport/priv/ruby1.9/erlport/erlang.rb:218:in `incoming_call'", "/Users/scotthamilton/.rvm/rubies/ruby-2.4.1/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb:55:in `require'", "/Users/scotthamilton/.rvm/rubies/ruby-2.4.1/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb:55:in `require'", "/Users/scotthamilton/Projects/export_ruby/_build/dev/lib/export_ruby/priv/ruby/schedule.rb:2:in `<top (required)>'", "/Users/scotthamilton/.rvm/rubies/ruby-2.4.1/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb:55:in `require'", "/Users/scotthamilton/.rvm/rubies/ruby-2.4.1/lib/ruby/site_ruby/2.4.0/rubygems/core_ext/kernel_require.rb:55:in `require'"]}
    (erlport) /Users/scotthamilton/Projects/export_ruby/deps/erlport/src/erlport.erl:234: :erlport.call/3
{% endhighlight %}

What just happened, you ask? As I mentioned earlier, the `bundler` managed paths are not loaded when ruby is started by `export` so `require` fails to load `ice_cube`.

While pairing with a coworker of mine, [Luke Imhoff](https://twitter.com/KronicDeth), we started poking around the `export` source code and found the [ruby option](https://github.com/fazibear/export/blob/master/lib/export/ruby.ex#L51):

```
## Ruby options
    - ruby: Path to the Ruby interpreter executable
```

The first thought was to simply set the `ruby:` option to point to a bash script that used `bundle exec`. Long story short, it wasn't that simple. After digging into `erlport` (the erlang lib that actually does the heavy lifting), Luke found that `erlport` is passing a [series of flags](https://github.com/hdima/erlport/blob/master/src/ruby.erl#L187-L193) to `ruby`.


{% highlight erlang %}
  Path = lists:concat([Ruby,
      " -e 'require \"erlport/cli\"'"
      % Start of script options
      " --"
      " --packet=", Packet,
      " --", UseStdio,
      " --compressed=", Compressed,
      " --buffer_size=", BufferSize]),
{% endhighlight %}

Putting the pieces together, we can point `erlport` to a bash script that runs `bundle exec`, but how do we get the flags that `erlport` needs to `bundle exec` in the bash script? As it turns out just use `@` in the bash script after the `bundle exec`. [Here's how]( ) it works.

Time to create the bash script. Remember to make it executable.

{% highlight shell %}
$ touch priv/ruby/bundle-exec-ruby
$ chmod +x priv/ruby/bundle-exec-ruby
{% endhighlight %}

Important: you want to use quotes so the white space around the flags `erlport` is maintained `"$@"`.

{% highlight bash %}
# export_ruby/priv/ruby/bundle-exec-ruby

#!/bin/bash
# get the dir path relative to the bash script file
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# change directory and silence output so it won't error in erlport
pushd $DIR > /dev/null 2>&1

# ensure that bundler is looking for the correct Gemfile
export BUNDLE_GEMFILE=$DIR"/Gemfile"

# change PATH and Gem file vars and pass in the flags set in erlport
bundle exec ruby "$@"
{% endhighlight %}

Now that the bash script is in place let's modify the elixir code to point to the bash script.

{% highlight elixir %}
# export_ruby/lib/schedule.ex

defmodule ExportRuby.Schedule do
  use Export.Ruby

  # Path to ruby files relative to the project root
  @ruby_lib Application.app_dir(:export_ruby, "priv/ruby")
  # Path to ruby executable
  @ruby_exec Application.app_dir(:export_ruby, "priv/ruby/bundle-exec-ruby")

  def daily(start_time, days) do
    # passing simple data types
    arguments = [to_string(start_time), days]

    # Note I'm using `start_link` vs `start` as other examples show.
    # Ensures the ruby process is cleaned up after parent process stops or crashes.
    # This example is not the most efficient technique as this starts and 
    # stops a ruby process on EVERY function call, which works but starting and 
    # stopping the process each time adds overhead.
    {:ok, ruby} = Ruby.start_link(ruby: @ruby_exec, ruby_lib: @ruby_lib)

    Ruby.call(ruby, "schedule", "daily", arguments)
  end
end
{% endhighlight %}

Once those changes are in place, you can recompile the source code without restarting iex.

{% highlight iex %}
iex(4)> c "lib/schedule.ex" 
warning: redefining module ExportRuby.Schedule (current version defined in memory)
  lib/schedule.ex:1

iex(5)> ExportRuby.Schedule.daily(DateTime.utc_now, 2)
["2017-10-22 15:25:17 UTC", "2017-10-23 15:25:17 UTC"]
{% endhighlight %}

Presto...you are using ruby with bundler to build a schedule via `ice_cube` in your Elixir app! Sorta gives you that special kind of feeling like...

![Army of Darkness Groovy](/assets/ash_groovy.gif)


You can find the [ export_ruby code ](https://github.com/shamil614/export_ruby) on Github.

In closing, if you want to maximize speed you need to make sure to use a Supervisor to keep the ruby process running instead of starting and stopping on every call. I'll write another shorter post on how I setup a Supervisor and include some benchmarks. At the time of writing this post, we haven't deployed the feature to production, but I just started testing locally and everything works great. As long as you're passing simple data types, `export/erlport` should be considered when you need a ruby gem but can't find an equivalent hex package, or simply don't have the time to write it in Elixir.

I hope this post helps others and saves you the many days I spent working to get `export` running in my Elixir app. 
