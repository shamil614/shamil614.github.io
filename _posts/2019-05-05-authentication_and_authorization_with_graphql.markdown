---
layout: post
title:  "Authentication and Authorization with GraphQL, Elixir and Absinthe"
date:   2019-5-05 12:00:54 -0500
categories: programming
tags: elixir graphql absinthe authorization authentication phoenix
---

For a while now GraphQL has been a regular topic at programming conferences. While I am a big fan of REST APIs, the touted benefits of GraphQL are appealing. I won't go into the REST vs GraphQL debate on this post because that's a post all of it's own. 

In short, I have worked on very big REST API with many endpoints that added to a lot of developer overhead. Since GraphQL promises a single endpoint for the client to request the desired payload, I could see how it can lead to efficiencies for computing resources and developers.

A little while ago our developer team kicked around the idea of adopting a GraphQL api. During that time I started to research more about GraphQL. While there are many posts covering GraphQL, I didn't find a lot on Authentication, and Authorization patterns. In this post I will cover what I learned about Authentication / Authorization with Absinthe and GraphQL.


## Overview

In order to implement a GraphQL api, the spec must be followed. The easiest way to build out a GraphQL api in Elixir is to use the well featured [Absinthe library](https://github.com/absinthe-graphql/absinthe). While we could use Absinthe alone, it's best to use [Absinthe Plug](https://github.com/absinthe-graphql/absinthe_plug) since it allows for the integration of GraphQL into a http api pipeline via `Plug`, and `absinthe_plug` requires Absinthe as a dependency. I won't be able to cover everything. Otherwise this post would have too much information. [View the full code](https://github.com/shamil614/graphql_blog/tree/auth) to see all the details.

## Authentication

As far as I can tell the GraphQL spec doesn't outline any requirements on how to authenticate a user, which I appreciate. While the GraphQL spec is much more specific then REST, there are certain areas like Authentication where the spec allows for the developer to decide what to implement. In this post I will implement a simple token authentication strategy via HTTP Authentication header. Here's how I went about adding Authentication:

Add `absinthe_plug` to your deps

{% highlight elixir %}
  defp deps do
    [
      {:absinthe_plug, "~> 1.4"},
      {:phoenix, "~> 1.4.0"},
      ....
    ]
  end
{% endhighlight %}

Setup the router to handle api requests and setup the GraphiQL utility.

{% highlight elixir %}
# lib/blog_web/router.ex

defmodule BlogWeb.Router do
  use BlogWeb, :router

  pipeline :api do
    plug :accepts, ["json"]
    plug BlogWeb.Plugs.Context
  end

  scope "/api" do
    pipe_through :api

    forward "/graphiql", Absinthe.Plug.GraphiQL,
      schema: BlogWeb.Schema,
      json_codec: Phoenix.json_library()

    forward "/", Absinthe.Plug, schema: BlogWeb.Schema
  end
end
{% endhighlight %}

Make sure to install the new deps `mix deps.get`.

Now a Plug can be created to read the HTTP Authentication header, look up the User, and pass the `current_user` to `Absinthe.run` via `context` in the `options` ([see docs](https://hexdocs.pm/absinthe/Absinthe.html#run/3-options)).


{% highlight elixir %}
# lib/blog_web/plugs/contex.ex
defmodule BlogWeb.Plugs.Context do
  @behaviour Plug

  import Plug.Conn
  import Ecto.Query, only: [where: 2]

  alias Blog.Repo
  alias Blog.Accounts.User

  def init(opts), do: opts

  def call(conn, _) do
    context = build_context(conn)
    # Absinthe.Plug calls Absinthe.run() with the options added to the `conn`.
    Absinthe.Plug.put_options(conn, context: context)
  end

  @doc """
  Return the current user context based on the authorization header
  """
  def build_context(conn) do
    with ["Bearer " <> token] <- get_req_header(conn, "authorization"),
         {:ok, current_user} <- authorize(token) do
      %{current_user: current_user}
    else
      _ -> %{}
    end
  end

  # Note this is a simple token auth strategy. This is should not be used in production.
  defp authorize(token) do
    User
    |> where(token: ^token)
    |> Repo.one()
    |> case do
      nil -> {:error, "invalid authorization token"}
      user -> {:ok, user}
    end
  end
end
{% endhighlight %}

Since `Absinthe.run` passes along the `context`, the `resolvers` can handle an authenticated / unauthenticated request. The `schema.ex` file defines the `query` for `posts`, and defines a `resolver` for the request.

{% highlight elixir %}
# lib/blog_web/schema
query do
  @desc "Get all posts"
  field :posts, list_of(:post) do
    # the third argument passed is the `context` set via `plug`
    resolve(&Resolvers.Content.list_posts/3)
  end
  
  ....
end
{% endhighlight %}

{% highlight elixir %}
# lib/blog_web/resolvers/content.ex

@doc """
Resolve Posts associated to a User, posts independent of a User and Unauthorized Users
"""
def list_posts(%User{} = author, args, %{context: %{current_user: current_user}}) do
  {:ok, Content.list_posts_by_author(args, author, current_user)}
end

def list_posts(_parent, args, %{context: %{current_user: current_user}}) do
  {:ok, Content.list_posts(args, current_user)}
end

def list_posts(_, _, _) do
  {:error, "Access denied"}
end
{% endhighlight %}

By matching on the context map with a `current_user` key, the authenticated user is accounted for. The last head matches for unauthenticated users.

Now with graphiql the Authorization Header can easily be passed in the api call. Navigate to `localhost:4000/api/graphiql` then under the `Headers` section click `Add`. There you can enter a fake token to correspond to a particular user with a value of `Bearer <Some-Token-Matching-A-User>`.

![Add Header via graphiql](/assets/images/graphql_auth/graphiql_add_header.png)

Here's an example query with a valid Authorization token.

![Valid Token request](/assets/images/graphql_auth/graphiql_auth.png)

Remove the header and make the same request.

![Invalid Token request](/assets/images/graphql_auth/graphiql_unauth.png)

Notice that no posts are returned, and the payload included an `errors` JSON object.

## Authorization

Authorization is not explicitly covered in the GraphQL spec, it is covered in the [Learn](https://graphql.github.io/learn/authorization/) section of the guides. The guide strongly advises the developer to put the Authorization logic in the business logic layer. Basically, this means put the logic in a module outside the GraphQL specific schema, content types, resolvers, etc.

In my sample code the User schema has a `role` attribute. The code has seeds to create a `admin` and a `consumer` User. The Post schema has an attribute called `private_notes` that only Users with a role of `admin` can view. 

The hook for authorizing a specific field can be implemented via the `post` `content type`.

{% highlight elixir %}
# lib/blog_web/schema/content_types.ex

defmodule BlogWeb.Schema.ContentTypes do
  use Absinthe.Schema.Notation

  alias BlogWeb.Resolvers

  object :post do
    field :id, :id
    field :title, :string
    field :body, :string
    field :author, :user
    field :published_at, :naive_datetime
    field :private_notes, :string do
      resolve(fn(post, _, context) ->
        Resolvers.Content.authorize_post_field(post, :private_notes, context)
      end)
    end
  end
end
{% endhighlight %}

The resolver then delegates to the business layer

{% highlight elixir %}
defmodule BlogWeb.Resolvers.Content do
  alias Blog.Accounts.User
  alias Blog.Content
  alias Blog.Content.Post

  def authorize_post_field(post = %Post{}, field, %{context: %{current_user: current_user}}) do
    Content.authorize_post_field(post, field, current_user)
  end

  def authorize_post_field(_post, _field, _) do
    {:error, "Not authorized for field"}
  end
  
  .....
end
{% endhighlight %}

The `Blog.Content` is where the business layer logic processes the authorization logic.

{% highlight elixir %}
# lib/blog/content.ex
def authorize_post_field(post, :private_notes, _current_user = %User{role: "admin"}) do
  {:ok, Map.get(post, :private_notes)}
end

def authorize_post_field(_post, :private_notes, _current_user = %User{}) do
  {:error, "Field not authorized"}
end
{% endhighlight %}

The first function matches on the `admin` User role which then returns a `{:ok, <val>}` for `admin` users. The second function matches for any other user role which returns a `{:error, <message>}` tuple preventing the access of the `private_notes` field.

Here's an example query via GraphiQL.

First set a token that belongs to a Admin User.
![Admin token set](/assets/images/graphql_auth/graphiql_admin_token.png)

Below is an example query requesting private notes.
![Admin requests private notes](/assets/images/graphql_auth/graphiql_admin_private_notes.png)

Notice that private notes are returned in the response.

Now let's test what happens for a `consumer`.

Set a token belonging to a Consumer User.
![Consumer token set](/assets/images/graphql_auth/graphiql_consumer_token.png)

Below is the request for `private_notes` by a `consumer`.
![Consumer requests private notes](/assets/images/graphql_auth/graphiql_consumer_private_notes.png)

The result shows the proper error message that was defined in the business layer module.

## Conclusion

Exploring GraphQL and Absinthe provided great insights into the spec, and ways to implement a GraphQL api. I'm very impressed with how both GraphQL and Absinthe define and implement the means to handle Authentication and Authorization in an Elixir application. I can see why so many developers are leveraging GraphQL to tackle large APIs.
