---
sidebar_position: 3
---

# Slices

In addition to the `/app` directory, Hanami also supports organising your application code into **slices**.

You can think of slices as distinct modules of your application. A typical case is to use slices to separate your business domains (for example billing, accounting or admin) or by feature concern (api or search).

Slices live in the `/slices` directory.

To create a slice, you can either create a new directory in `/slices`:

```ruby
mkdir -p slices/admin

bundle exec hanami console
Admin::Slice
=> Admin::Slice
```

Or run `bundle exec hanami generate slice api`, which has the added benefit of adding some slice-specific classes, like actions:

```shell
bundle exec hanami generate slice api

slices
└── api
    ├── action.rb
    └── actions
```


TODO - the rest of slices :)
