---
sidebar_position: 1
---

# Routing

Hanami provides a fast, simple router for handling http requests.

To add a route to your application, define it in your `Routes` class in the `config/routes.rb` file.

If you ran `hanami new bookshelf`, your `config/routes.rb` file will look like this:

```ruby title="config/routes.rb"
# frozen_string_literal: true

module Bookshelf
  class Routes < Hanami::Routes
    root { "Hello from Hanami" }
  end
end
```

## Composing a route

In the Hanami router, each route is comprised of:
- a HTTP method (i.e. `get`, `post`, `put`, `patch`, `delete`, `options` or `trace`)
- a path
- an endpoint to be invoked.

Endpoints are usually actions within your application, but they can also be a block, a rack application, or anything that responds to `#call`.

```ruby title="Example routes"
get "/authors", to: "authors.index"
get "/authors/:id", to: "authors.show"
post "/authors", to: "authors.create"
put "/authors/:id", to: "authors.update"
get "/rack-app", to: RackApp.new
```

A root method defines a root route for handling GET requests to "/". Above, the root path calls a block which returns "Hello from Hanami". You can also invoke an action for root requests by specifying `root to: "my_action"`. For example to invoke a `"home"` action:

```ruby title="config/routes.rb"
# frozen_string_literal: true

module Bookshelf
  class Routes < Hanami::Routes
    root to: "home"
  end
end
```

Let's add three routes to our bookshelf application: one for listing an index of books, one for showing a particular book, and one for creating a new book.

[Actually, let's add full set of CRUD here to show that off]

```ruby title="config/routes.rb"
# frozen_string_literal: true

module Bookshelf
  class Routes < Hanami::Routes
    root to: "home"

    get "/books", to: "books.index"
    get "/books/:id", to: "books.show"
    post "/books", to: "books.create"
  end
end
```

Hanami provides a `hanami routes` command to inspect your application's routes. Let's run `bundle exec hanami routes` on the command line after adding our new routes:

```shell title="bundle exec hanami routes"
GET     /                             home                          as :root
GET     /books                        books.index
GET     /books/:id                    books.show
POST    /books                        books.create
```

TODO: the rest of routing :)
