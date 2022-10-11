---
sidebar_position: 2
---

# Providers

Providers are a way to register components with your containers, outside of the automatic registration mechanism detailed in [containers and dependencies](/docs/application-architecture/containers).

Providers are useful when:

- you need to set up a dependency that requires non-trivial configuration (often a third party library, or some library-like code in your `lib` directory)
- you want to take advantage of provider lifecycle methods (prepare, start and stop)
- you want to share a component across both your app container and the containers of all your [slices](/docs/application-architecture/slices).

App-level providers should be placed in the `config/providers` directory. Slices can have their own providers also, placed in `slices/my_slice/providers`.

Here's an example provider for that registers an email client in the app container, using an imagined third-party Acme Email service.

```ruby title="config/providers/email_client.rb"
# frozen_string_literal: true

Hanami.app.register_provider(:email_client) do
  prepare do
    require "acme_email/client"
  end

  start do
    client = AcmeEmail::Client.new(
      api_key: target["settings"].acme_api_key,
      default_from: "no-reply@bookshelf.example.com"
    )

    register "email_client", client
  end
end
```

The above provider initializes an instance of Acme's email client, providing an api key from the application's setting as well as a default from address, then registers the client in the app container with the key `"email_client"`.

The registered dependency can now be used in app components, via `include Deps["email_client"]`:

```ruby title="app/emails/welcome/operations/send.rb"
# frozen_string_literal: true

module NotificationsService
  module Emails
    module Welcome
      module Operations
        class Send
          include Deps["email_client", "settings"]

          def call(name:, email_address:)
            return unless settings.email_sending_enabled

            email_client.deliver(
              to: email_address,
              subject: "Welcome!",
              text_body: "Welcome to Bookshelf #{name}"
            )
          end
        end
      end
    end
  end
end
```

Every provider has a name (`Hanami.app.register_provider(:my_provider_name)`) and registers _one or more_ related components with the relevant container. Registered items are not limited to objects - they can be classes too.

```ruby title="config/providers/something_provider.rb"
# frozen_string_literal: true

Hanami.app.register_provider(:something_provider) do
  start do
    register "something", Something.new
    register "another.thing", AnotherThing.new
    register "thing", Thing
  end
end
```

## Provider lifecycle

Providers offer a three-stage lifecycle: `prepare`, `start`, and `stop`. Each has a distinct purpose:

- prepare - basic setup code, here you can require 3rd party code and perform basic configuration
- start - code that needs to run for a component to be usable at runtime
- stop - code that needs to run to stop a component, perhaps to close a database connection, or purge some artifacts.

```ruby title="config/providers/database.rb"
Hanami.app.register_provider(:database) do
  prepare do
    require "3rd_party/db"

    register "database", 3rdParty::DB.configure(target["settings"].database_url)
  end

  start do
    target["database"].establish_connection
  end

  stop do
    target["database"].close_connection
  end
end
```

Lifecycle steps will not run until a provider is required by another component, is started directly, or when the container finalizes as a result of Hanami booting.

`Hanami.boot` and `Hanami.shutdown` call `start` and `stop` respectively on each of the application’s registered providers.

Lifecycle transitions can be triggered directly by using `Hanami.app.container.prepare(:provider_name)`, `Hanami.app.container.start(:provider_name)` and `Hanami.app.container.stop(:provider_name)`.

## Accessing the container via `#target`

Within a provider, the `target` method (also available as `target_container`) can be used to access the container (either the app container, or, if the provider is specific to a slice, the slice's container).

This is useful for accessing the application's settings or logger (via `target["settings]` and `target["logger"]`). It can also be used when a provider wants to ensure another provider has started before starting itself, via `target.start(:provider_name)`:

```ruby title="config/providers/uploads_bucket"
Hanami.app.register_provider(:uploads_bucket) do
  prepare do
    require "aws-sdk-s3"
  end

  start do
    target.start(:metrics)

    uploads_bucket_name = target["settings"].uploads_bucket_name

    credentials = Aws::Credentials.new(
      target["settings"].aws_access_key_id,
      target["settings"].aws_secret_access_key,
    )

    uploads_bucket = Aws::S3::Resource.new(credentials: credentials).bucket(uploads_bucket_name)

    register "uploads_bucket", uploads_bucket
  end
end
```

## Default providers

Hanami ships with several providers. TODO.
