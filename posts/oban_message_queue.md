# Using Oban as a Message Queue

# Introduction

A little over a year ago [I wrote](https://mikeybower93.github.io/posts/loosely_coupled_phoenix_contexts) about loosely coupling Phoenix Contexts using a message quque (in this case Kafka). I received some [comments](https://twitter.com/elixirstatus/status/1518195416814260225?cxt=HHwWgsC47YKs25EqAAAA) on that blog asking whether I considered using [Oban](https://hexdocs.pm/oban/Oban.html). 

Despite being a big fan of Oban, at the time I suggested that it wouldn't be good for this use case, as its not based on a publish/subscribe model. Instead you queue specific background tasks to be executed, and those tasks are explicitly declared and executed as part of the same business logic code.

However recently I was thinking about how Oban could be used as a message queue specifically as a publish/subscribe system to decouple Phoenix Contexts.

# Example

To demonstrate how this could be done I created a basic Phoenix project called [Brokker](https://github.com/MikeyBower93/brokker). The logic is similar to the [initial post](https://mikeybower93.github.io/posts/loosely_coupled_phoenix_contexts) centered around company expenses. Whereby we decouple the payments Context from the expenses Context and AML (anti money laundering), the expenses and AML Contexts both care about payments, but the payment Context should not couple to these Contexts. The Oban architecture I created to demonstrate this is as follows
![](/images/oban_message_queue_1.png)

## Producer Logic

When payments are created in the payments Context it creates a [Ecto Multi](https://hexdocs.pm/ecto/Ecto.Multi.html) transaction, the logic is as follows
```elixir
def create_payment() do
    Ecto.Multi.new()
    # Imagine some transaction for payment logic here
    |> MessageBroker.queue_publication(:payment_created, %{payment_id: 1, merchant: "Ocado"})
    |> Repo.transaction()
  end
```
as we can see this calls `MessageBroker.queue_publication`, the logic for this is the following
```elixir
def queue_publication(multi, event, payload) do
  consumers = Application.get_env(:brokker, :message_broker_consumers)[event]

  Oban.insert_all(
    multi,
    :jobs,
    Enum.map(consumers, fn consumer -> consumer.new(payload) end)
  )
end
```
this finds all consumer workers that want to be notified about the particular event, and creates an Oban job insert for each one of those workers.

## Consumer Logic

Each consumer is defined within the context, for example
```elixir
defmodule Brokker.Aml.PaymentMadeConsumerWorker do
  use Oban.Worker
  require Logger

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"payment_id" => id}}) do
    Logger.warn("Creating aml flag for payment id '#{id}'")
    :ok
  end
end
```
given that the expenses Context also care about this event, they have a similar consumer. You can create a worker for any number of interested consumers.

These consumers are then registered in the elixir config as follows
```elixir
config :brokker, :message_broker_consumers,
  payment_created: [
    Brokker.Expenses.PaymentMadeConsumerWorker,
    Brokker.Aml.PaymentMadeConsumerWorker
  ]
```

Therefore when the `MessageBroker.queue_publication` needs to insert the workers, it does a lookup for workers in the config for the given event ID.

This means whenever the producer commits the transaction, the consumer workers will execute. This acts as a multi consumer to one producer architecture.

# Conclusions

We can see from this post that it is possible to emulate a publish/subscribe pattern (similar to a message queue like Kafka) just using Oban. There are pros and cons to this approach.

## Pros
- Simplicity, this just sits on top of Postgres, no additional infrastructure, monitoring, dependencies.
- Reliability
  - Oban is persistant, so ay restarts/crashes will be continued when the app starts again.
  - Oban deals well with failures creating backoffs etc, interestingly this is easier than say using Kafka or RabbitMQ where you have to manually declare a [Dead Letter Queue](https://en.wikipedia.org/wiki/Dead_letter_queue).
  - Ability to apply max queue sizes and max jobs processed per a particular timeframe via the wonderful configuration Oban provides.
- Performance, whilst Oban cannot compete with the likes of RabbitMQ and Kafka, its [no slouch](https://getoban.pro/articles/one-million-jobs-a-minute-with-oban).
- Its runs on Postgres, and therefore the queuing of the tasks is part of the same producer transaction that applies logic. This is know as the [Dual Write Problem](https://www.cockroachlabs.com/blog/message-queuing-database-kafka/), which requires additional work to solve with an external broker like Kafka, with Oban this is solved naturally.

## Cons
- Dependant on the scale of your system, this could apply too much pressure on Postgres. As seen in the "Pros" section, Oban is [no slouch](https://getoban.pro/articles/one-million-jobs-a-minute-with-oban), so we would be talking about a lot of events to hit this limit, but its worth considering your scale.
- Cannot partition/replicate like distributed message brokers, which is how those systems achieve such a availability and throughput.
- Doesn't support real time ordering like Kafka.
- Constrained to one system, so if you had external systems that wanted to consume those events, they would be out of luck.

# Final Words

We can see that we can achieve a reliable decoupled architecture with  nothing more than Elixir, Postgres and Oban.

I want to give a few shoutouts to posts that inspired me to write this
- Firstly the original suggestions in the [Twitter post](https://twitter.com/elixirstatus/status/1518195416814260225?cxt=HHwWgsC47YKs25EqAAAA).
- [This](https://mkaszubowski.com/2021/10/15/safe-reliable-side-effects-outbox-pattern.html) post which describes using Oban as an [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html) implementation.
- [This](https://blog.appsignal.com/2022/05/10/a-guide-to-event-driven-architecture-in-elixir.html) post which talks about event driven architecture using Elixir.

Happy coding!