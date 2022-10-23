---
sidebar_position: 3
---

# Slices

In addition to the `app` directory, Hanami also supports organising your application code into **slices**.

You can think of slices as distinct modules of your application. A typical case is to use slices to separate your business domains (for example billing, accounting or admin) or to have separate modules for a particular feature (API) or technical concern (search).

Slices exist in the `slices` directory.
## Creating a slice

Hanami provides a slice generator. To create an API slice, run `bundle exec hanami generate slice api`.

This creates a directory in `slices`, adding some slice-specific classes like actions:

```shell
bundle exec hanami generate slice api

slices
└── api
    ├── action.rb
    └── actions
```

Simply creating a new directory in `slices` will also create a slice:

```shell
mkdir -p slices/admin

slices
└── admin
```

## Features of a slice

Slices offer much of the same behaviour and features as Hanami's `app` folder.

A Hanami slice:

- has its own container
- can have its own providers (e.g. `slices/api/providers/my_provider.rb`)
- can include actions, routable from the application's router
- can import and export components from other slices
- can be prepared and booted independently of other slices
- can have its own slice-specific settings (e.g. `slices/api/config/settings.rb`)

## Slice containers

Like Hanami's `app` folder, components added to a Hanami slice are automatically organised into the slice's container.

For example, suppose our Bookshelf application, which catalogues international books, needs an API to return the name, flag, and currency of a given country. We can create a countries show action in our API slice (by running `bundle exec hanami generate action countries.show --slice api` or by adding the file manually) that looks like:

```ruby title="slices/api/actions/countries/show.rb"
# frozen_string_literal: true

require "countries"

module API
  module Actions
    module Countries
      class Show < API::Action
        include Deps[
          query: "queries.countries.show"
        ]

        params do
          required(:country_code).value(included_in?: ISO3166::Country.codes)
        end

        def handle(request, response)
          response.format = format(:json)

          halt 422, {error: "Unprocessable country code"}.to_json unless request.params.valid?

          result = query.call(
            request.params[:country_code]
          )

          response.body = result.to_json
        end
      end
    end
  end
end
```

This action uses the countries gem to check that the provided country code (`request.params[:country_code]`) is a valid ISO3166 code and returns a 422 response if it isn't.

If the code is valid, the action calls the countries show query (aliased here as `query` for readability). That class might look like:

```ruby title="slices/api/queries/countries/show.rb"
# frozen_string_literal: true

require "countries"

module API
  module Queries
    module Countries
      class Show
        def call(country_code)
          country = ISO3166::Country[country_code]

          {
            name: country.iso_short_name,
            flag: country.emoji_flag,
            currency: country.currency_code
          }
        end
      end
    end
  end
end
```

As an exercise, as with `Hanami.app` and its app container, we can boot the `API::Slice` to see what its container contains:

```ruby
bundle exec hanami console

bookshelf[development]> API::Slice.boot
=> API::Slice
bookshelf[development]> API::Slice.keys
=> ["settings",
 "actions.countries.show",
 "queries.countries.show",
 "inflector",
 "logger",
 "notifications",
 "rack.monitor",
 "routes"]
```

We can call the query with a country code:

```
bookshelf[development]> API::Slice["queries.countries.show"].call("UA")
=> {:name=>"Ukraine", :flag=>"🇺🇦", :currency=>"UAH"}
```

To add a route for our action, we can add the below to our application's `config/routes.rb` file. This is done for you if you used the action generator.

```ruby title="config/routes.rb"
# frozen_string_literal: true

module Bookshelf
  class Routes < Hanami::Routes
    root { "Hello from Hanami" }

    slice :api, at: "/api" do
      get "/countries/:country_code", to: "countries.show"
    end
  end
end
```

`slice :api` tells the router it can find actions for the routes within the block in the API slice. `at: "/api"` specifies an optional mount point, such the routes in the block each be mounted at `/api`.

After running `bundle exec hanami server`, the endpoint can be hit via a `GET` request to `/api/countries/:country_code`:

```shell
curl http://localhost:2300/api/countries/AU
{"name":"Australia","flag":"🇦🇺","currency":"AUD"}
```


# Slice imports and exports

Suppose that our bookshelf application uses a content delivery network (CDN) to serve book covers. While this makes these images fast to download, it does mean that book covers need to be purged from the CDN when they change, in order for freshly updated images to take their place.

Images can be updated in one of two ways: the publisher of the book can sign in and upload a new image, or a Bookshelf staff member can use an admin interface to update an image on the publisher's behalf.

In our bookshelf app, an `Admin` slice supports the latter functionality, and a `Publisher` slice the former. Both these slices want to trigger a CDN purge when a book cover is updated, but neither slice needs to know exactly how that's achieved. Instead, a `CDN` slice can manage this operation.

```ruby title="slices/cdn/book_covers/purge.rb"
module CDN
  module BookCovers
    class Purge
      def call(book_cover_path)
        # "Purging logic here!"
      end
    end
  end
end
```

Slices can be configured by creating a file at `config/slices/slice_name.rb`.

To configure the `Admin` slice to import components from the CDN container (including the purge component above), we can create a `config/slices/admin.rb` file with the following configuration:

```ruby title="config/slices/admin.rb"
module Admin
  class Slice < Hanami::Slice
    import from: :cdn
  end
end
```

Let's see this import in action in the console, where we can see that the `Admin` slices's container now has a `"cdn.book_covers.purge"` component:

```ruby
bundle exec hanami console

bookshelf[development]> Admin::Slice.boot.keys
=> ["settings",
 "cdn.book_covers.purge",
 "inflector",
 "logger",
 "notifications",
 "rack.monitor",
 "routes"]
```

Using the purge operation from the `CDN` slice within the `Admin` slice component below is now as simple as using the `Deps` mixin:

```ruby title="slices/admin/books/operations/update.rb"
module Admin
  module Books
    module Operations
      class Update
        include Deps[
          "repositories.book_repo",
          "cdn.book_covers.purge"
        ]

        def call(id, params)
          # ... update the book using the book repository ...

          # If the update is successful, purge the book cover from the CDN
          purge.call(book.cover_path)
        end
      end
    end
  end
end
```

It's also possible to import only specific components from another slice. Here for example, the `Publisher` slice imports strictly the purge operation, while also - for reasons of its own choosing - using the suffix `content_network` instead of `cdn`:

```ruby title="config/slices/publisher.rb"
module Publisher
  class Slice < Hanami::Slice
    import keys: ["book_covers.purge"], from: :cdn, as: :content_network
  end
end
```

In action in the console:

```ruby
bundle exec hanami console

bookshelf[development]> Publisher::Slice.boot.keys
=> ["settings",
 "content_network.book_covers.purge",
 "inflector",
 "logger",
 "notifications",
 "rack.monitor",
 "routes"]
```

Slices can also limit what they make available for export to other slices.

Here, we configure the CDN slice to export only its purge component:

```ruby title="config/slices/cdn.rb"
module CDN
  class Slice < Hanami::Slice
    export ["book_covers.purge"]
  end
end
```

TODO: the rest of slices :)
