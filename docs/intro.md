---
sidebar_position: 1
title: Getting started
description: Getting started with Hanami 2
---

# Getting started

Hanami is a Ruby framework designed to create software that is well-architected, maintainable and a pleasure to work on.

These guides aim to introduce you to the Hanami framework and demonstrate how its components fit together to produce a coherent application.

Ideally, you already have some familiarity with web applications and the [Ruby language](https://www.ruby-lang.org/en/).

:::tip Hanami 2.0 is beta software

Hanami 2 currently beta software. If you encounter an issue with these guides, please raise an issue or contribute a correction on our guides repository. If you encounter what you think is a bug with Hanami, please raise an issue on the [Hanami repository](https://github.com/hanami/hanami).

:::

## Creating a Hanami application

### Prerequisites

To create a Hanami application, you will need Ruby 3.0 or greater. Check your ruby version:

```bash
ruby --version
```

If you need to install Ruby, follow with the instructions on [rubylang.org](https://www.ruby-lang.org/en/documentation/installation/).

### Installing the gem

In order to create a Hanami application, first install the hanami gem:

```bash
gem install hanami --pre
```

### Generating your first application

Hanami provides a `hanami new` command for generating a new application. Let's use it to create a new application called "bookshelf":

```bash
hanami new bookshelf
```

Here's what was generated for us:

```shell title="Generated files"
cd bookshelf
tree .
.
├── Gemfile
├── Gemfile.lock
├── Guardfile
├── README.md
├── Rakefile
├── app
│   ├── action.rb
│   └── actions
├── config
│   ├── app.rb
│   ├── puma.rb
│   ├── routes.rb
│   └── settings.rb
├── config.ru
├── lib
│   ├── bookshelf
│   │   └── types.rb
│   └── tasks
└── spec
    ├── requests
    │   └── root_spec.rb
    ├── spec_helper.rb
    └── support
        ├── requests.rb
        └── rspec.rb

9 directories, 16 files
```

As you can see, a new Hanami application has just 16 files in total. We'll look at each file as this guide progresses but for now let's get our new application running.

In the bookshelf directory, run:

```shell
hanami server
```

If all has gone well, you should see this output:

```
Puma starting in single mode...
* Puma version: 5.6.5
*  Min threads: 0
*  Max threads: 5
*  Environment: development
*          PID: 31634
* Listening on http://0.0.0.0:2300
Use Ctrl-C to stop
```

Visit your application in the browser at [http://localhost:2300](http://localhost:2300)

```
open http://localhost:2300
```

You should see "Hello from Hanami"!

:::tip A note on bundle install

You may have noticed that we did not run `bundle install` before starting our server. That's because the `hanami new` command runs `bundle install` for you. To opt out of this behaviour, run `hanami new --skip-bundle`. See `hanami new --help` for more options.

:::


## Anatomy of a generated application

Before we add our first functionality, let's take a brief look at some of the key files in your freshly generated application.

### The `App` class

As we'll explore in this guide, `config/app.rb` defines the `App` class - the core of your application.

```ruby title="config/app.rb"
# frozen_string_literal: true

require "hanami"

module Bookshelf
  class App < Hanami::App
  end
end
```

This class allows you to configure application-level behaviours, while also providing a way to do things like booting your application, or starting or stopping your application's [providers](/docs/application-architecture/providers).

It's also the interface some core components, such as your application's settings, via `Hanami.app["settings"]`, and its logger, via `Hanami.app["logger"]`.

Read more about [Hanami's app class](#).

### Routes

Your application's routes are defined using the `Routes` class in `config/routes.rb`.

```ruby title="config/routes.rb"
# frozen_string_literal: true

module Bookshelf
  class Routes < Hanami::Routes
    root { "Hello from Hanami" }

    get "/books/:id", to: "books.show"
  end
end
```

Routes in Hanami most commonly invoke what Hanami calls actions. For example, with the routes configuration above, a GET request to `/books/1` will call a `"books.show"` action, defined in your application's `app/actions` folder:

```ruby title="app/actions/books/show.rb"
# frozen_string_literal: true

module Bookshelf
  module Actions
    module Books
      class Show < Bookshelf::Action
        def handle(request, response)
          # Show the book specified by request.params[:id] here
        end
      end
    end
  end
end
```

Read more about routes an actions in [HTTP handling](/docs/category/http-handling).

### Settings

Hanami provides a `Settings` class where you can define the custom settings that your application needs.

```ruby title="config/settings.rb"
# frozen_string_literal: true

require "bookshelf/types"

module Bookshelf
  class Settings < Hanami::Settings
    # Define your app settings here, for example:
    #
    setting :my_flag, default: false, constructor: Types::Params::Bool
  end
end
```

Settings read from the environment (i.e. the `my_flag` setting above takes its value from the `MY_FLAG` environment variable), and are available either through `Hanami.app["settings"]` or by injecting the `"settings"` component via Hanami's dependency injection mechanism (see [Dependency injection](#)).

A **default** and a **constructor** can be optionally specified for each setting. The `Types::Params::Bool` constructor above will coerce any value found in the `MY_FLAG` environment variable into a boolean value. And because of the `default: false` argument, if the `MY_FLAG` environment variable is absent, `my_flag` will be `false`.

This means that you can trust `Hanami.app["settings"].my_flag` to always return a boolean.

Read more about [Hanami's settings](#).


## Commands

Hanami ships with useful commands for things like starting a console, inspecting routes and generating code.

To list available commands, run:

```shell
bundle exec hanami --help

Commands:
  hanami console                              # App REPL
  hanami generate [SUBCOMMAND]
  hanami install
  hanami middlewares                          # List all the registered middlewares
  hanami routes                               # Inspect application
  hanami server                               # Start Hanami server
  hanami version
```

We'll see most of these commands at play in this guide. For complete information on these commands, see the [Hanami reference](#).

## What's next?

Now that we're a little acquainted, let's examine the structure of a Hanami application in more detail through [containers, providers and slices](/docs/category/application-architecture).
