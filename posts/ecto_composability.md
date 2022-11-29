# JSON API and Ecto

# Introduction
Over the past few years I have worked on quite a few projects that provide JSON API's using Elixir/Phoenix, and have found that we always end up in a situation that requires the API to be flexible. To illustrate this, consider an example where you have a JSON API that returns information about space centers, astronauts and rockets. You might have for example, a rocket which is associated to a space center, which also has astronauts who fly in the rocket, additionally the space center might belong to a particular country. The JSON API might provide an endpoint for rockets, at which a query to that endpoint might look like `/api/v1/rockets`. However I find the requirements for that endpoint grow in the following way:
- The entity needs filtering, for example `api/v1/rockets?filter[name]=Apollo`
- This can then mutate into different filter types being required, for example you might want to match that the filter is simply contained in the property like `api/v1/rockets?filter[name][LK]=ollo`
- Then the API might go even further requiring nested properties to be filtered such as `api/v1/rockets?[space-center.country.name]=UK`

Before you know it you need to provide an API that is flexible to the client requesting the data. However a lot of implementations I have seen often end up working in an inefficent way, as it works on the request after the database query has executed. This can result in a tonne of results being returned from the database, followed by in memory filtering, or preloading of includes (`api/v1/rockets?[include]=space-center`) as its preparing the results. I wanted to work out a way that we could do this with a small amount of database requests ensuring the API stays efficent and reusable.

To that end, I ended up creating a library that would parse the request, and generate a query that you could execute based on the parameters received, the library can be seen [here](https://github.com/MikeyBower93/json_api_ecto_builder). 

This blog will describe how Ecto allowed this to be achieved quite easily. 

# Ecto Composibility
## A Simple Example
One of the things I like the most about Ecto queries is how they are composible, so for example you can create a pipeline to build up a query gradually with small functions, for example:
```elixir
Rocket
|> filter_by_name("Apollo")
|> filter_by_active()
|> Repo.all()

defp filter_by_name(query, value) do
  where(query, [rocket], rocket.name == ^value)
end
 
defp filter_by_active(query) do
  where(query, [rocket], not is_nil(rocket.deleted_at))
end
```
Notice how you can have small pure functions that adapt the query, but don't execute until you tell it to using `Repo`. This example can be taken further by reducing over some parameters to create a query that can be executed, for example imagine:
```elixir
def list_rockets(params) do
    params
    |> Map.to_list
    |> Enum.reduce(Rocket, &filter/2)
    |> Repo.all()
end

defp filter({"name", value}, query) do
  where(query, [rocket], rocket.name == ^value)
end

defp filter({"age", value}, query) do
  where(query, [rocket], rocket.age == ^value)
end

list_rockets({"name" => "Apollo", "age" => 20})
```
In this example we are passing through a map of filters, which are being broken down into a list, which is then used to generated a query using an `Enum.reduce`. 

## How the Library Uses Composibility
The ability to reduce and compose queries is the cornerstone of the JSON API builder I created, it allowed me to parse the parameters, and generate a query off them by reducing over those parameters, an example of this in the library itself would be here:
```elixir
defmodule JsonApiEctoBuilder.Applier.Filter do
  import Ecto.Query
  alias JsonApiEctoBuilder.ParamParser.Filter

  def apply(query, params, base_alias) do
    params
    |> Filter.parse(base_alias)
    |> Enum.reduce(query, &do_apply/2)
  end

  defp do_apply({field_param, :GT, value, binding}, query) do
    query
    |> where([{^binding, x}], field(x, ^field_param) > ^value)
  end

  defp do_apply({field_param, :GTE, value, binding}, query) do
    query
    |> where([{^binding, x}], field(x, ^field_param) >= ^value)
  end
  ...
end
```
This is a snippet of how the filtering works in the library, a few key points around this are:
- The `Filter.parse` takes the parameters from the request in and parses them, for example `api/v1/rockets?filter[age][GT]=18` would be broken down into, the field, the table its on and the operator (greater than).
- We then reduce over the parameters to build up the query, we can see how this is being done in the `do_apply` function, where its basically pattern matching on the operator. 
- The `do_apply` functions do have a wierd set of binding destructuring, however that will be covered in the next section. 

## How Composibility Allows for Developer Control
Inspired by how Ecto queries work, and the composible nature of them allowing us to only execute the query when the developer is ready was something that I wanted to adopt in the library. So much so that when calling the library to generate a query based on the query parameters, the developer can pass an initial query in, or even amend the query once it has been generated. For example:
```elixir
query = 
  if current_user.role == "admin" do
    (from r in Rocket, as: :rocket)
  else
    (from r in Rocket, as: :rocket
     where: not r.is_secret_rocket)
  end 

query = JsonApiEctoBuilder.build(query, Rocket, :rocket, json_api_parameters, &apply_join/2)

results = Repo.all(query)
...
```
This example should demonstrate a few things:
- If you have a scenario where the authenticated user might only be able to see particular results, you can use the composible nature of Ecto queries to your advantage, in the above example, when generating the query we set an initial condition where only admins can see secret rockets, otherwise they are hidden. This condition is within the same query that the parameters will be applied to ensure full flexibility of the builder.
- Even when the builder has done its job and created a query from the parameters, we return the query itself rather than executing it, that way you can decide how you want to use it, for example you might want to paginate; you might want to list all; you may even want to append a select to the query and run an aggregate function on it, that is your call!

# Ecto Named Bindings
## A Simple Example
One other piece that I would like to touch on is how the recent addition of Ecto named bindings made this library possible. To demonstrate this, lets imagine an example with Ecto positional bindings:
```elixir
query =
    (from r in Rocket,
    join: sc in assoc(r, :space_center))

query = filter_by_space_center_name(query, "Houston")

def filter_by_space_center_name(query, name) do
  where(query, [_rocket, space_center], space_center.name == ^value)
end 
```
In the example we have applied a join because there is a relationship, however the problem is when we call the filter function we have to selectively say which table in the query to execute the filter on, but the way you actually choose the table is by defining the position of where that join actually took place. So in the previous example I have illustrated this in the `filter_by_space_center_name` function where I have destructured the space center as the second table in the query, this might work when you are creating a very specific API, but when trying to make a resuable generic JSON API library this leaves you in a hard position, because you don't know the following:
- The position of the table.
- Does it even have the table joined in the query.
- What the table even represents.

This is something Ecto named bindings can solve, in a nutshell these work by allowing you to apply an alias to the table, rather than them existing at some position in the query, so for example you can do this:
```elixir
query =
    (from r in Rocket, as: :rocket,
    join: sc in assoc(r, :space_center), as: :space_center)

query = filter_by_space_center_name(query, "Houston")

def filter_by_space_center_name(query, name) do
  where(query, [space_center: space_center], space_center.name == ^value)
end 
```
This essentially does the following:
- When I generate the query, I state that each table has an alias (`:rocket`, `:space_center` etc)
- Then when I go to run a filter on the query, I don't need to know the position of that table, I can simply destructure it with `[space_center: space_center]`.

## How the Library Uses Named Bindings
As you can probably see, the problem created by positional bindings would have made the API difficult to develop, as it wouldn't be able to know which table to apply the filter to. Initially how the library works is it checks the filters to see what tables need joining in the query, an example of this can bee seen here: 
```elixir
def apply(query, params, apply_join_callback) do
  params
  |> Join.parse
  |> Enum.reduce(query, fn join, query ->
      case has_named_binding?(query, join) do
      true ->
          query
      false ->
          apply_join_callback.(join, query)
      end
  end)
end
```
This is doing the following:
- Taking the parameters and getting a unique list of the tables that need joining.
- Going through those parsed tables and checking if they already have been set in the query using `has_named_binding?`, this is great as if the developer has already added some joins to the table for a pre query (like the permission example earlier), then we aren't duplicating the join, allowing us to keep the query efficent.
- If we notice that the binding isn't there, we know it needs adding, we then essentially do a callback to ask the code using the library to do the join for us. Unfortuantly, I couldn't find a way that I could apply the join in the library and provide the correct named bindings, as you cannot do `join: x in assoc(x, joining_table), as: ^alias_variable`, as the `as` property is required to be there at compile time. 

Once the query has the joins applied and it gets to the filtering the code can destructure the binding, for example in a previous example of the library you will have seen this:
```elixir
defmodule JsonApiEctoBuilder.Applier.Filter do
  import Ecto.Query
  alias JsonApiEctoBuilder.ParamParser.Filter
 
  ...
  defp do_apply({field_param, :GTE, value, binding}, query) do
    query
    |> where([{^binding, x}], field(x, ^field_param) >= ^value)
  end
  ...
end
```
This is where the filters get applied to the query, the crucial point here is that this can be executed for any of the joins and the base `from` table. When the filter function is executed, the binding is passed in as an atom, so for example that could be `:astronaut` or `:space_center` etc. Then when the `where` is added to the query, we request the named binding in the query. The only difference here is that its destructured differently, as what we want for example might be `[space_center: space_center]`, however as the first argument is a variable to the function, we need to destructure it as a tuple, which is how the bindings work underneath the hood. 

# Conclusions

Although there are additional things the library can do, such as applying sorts and includes, this is the key part I wanted to demostrate, as it shows how Ecto allows us to build efficent queries, without compromising on reusability. 

Although I am happy with the library, and it has been useful on a few projects there are some considerations on how it could have been done differently, or where it might not fit in peoples projects which include:
- Marcos could have been used to stop the developer having configure the library for usage. For example instead of the developer having to pass in a callback function to apply the join, to solve the named bindings being required at runtime, I could have potentially used macros to generate this at compile time. 
- Though JSON API is quite a nice standard for requesting data and can be very flexible, it certainly seems to be a less adopted way to approach API's. Perhaps a lot of projects would be better served either looking to REST API's or GraphQL.
- Some deliberation would need to be made in a team as to whether having your web requests being intrinsically linked to your database layer is a good idea, this could be a bad choice if you have complex buisness rules or trying to follow domain driven design utilitizing contexts.

All considerations aside, this works nicely for CRUD applications, and can remove the need for duplicate code, which in my experience always ends up doing the same thing.

Happy coding!
