# Distributed Patterns in Payments with Elixir

# Introduction
Recently I was reading [this](https://t.co/T6CjcTfn7g) excellent post by Jon Chew from Airbnb about how they achieve consistency, performance and correctness in their microservice architecture when dealing with payments internally. This got me reflecting on a number things, such as my personal experiences during my career with such problems, and also considering excellent books I had read about distributed systems such as Martin Kleppmann's "Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems" and Chris Richardson's "Microservices Patterns: With Examples in Java".

I personally find that it can often be hard to piece together different information from blogs, books, small code examples etc to paint the full picture, this is unfortuantely true when trying to apply this to a language I'm passionate about (Elixir), as these examples are often in Java or Object Oriented heavy books. To that end, I wanted to create a fairly extensive example in Elixir about how you tie together these distributed patterns to achieve reliability and scalability in payments. The example is demonstrated in [this](https://github.com/MikeyBower93/ex_bank) repository called "ex_bank", this is to demonstrate a small banking application, where a customer can have an account, which has a balance, and that customer can send money from that account to someone via their sort code and account number, this in turn has to reduce the balance on our system, create a payment transaction, and notify a payment provider to send the money.

# Considerations
There are a number of considerations that need to be made in such an application, but before listing these, I want to lay out what "types" of considerations that are being made here, as any application can have a number of considerations, from scalability, to testing, to deployment etc, however we are dealing specifically with the questions around distributed systems here. To this end, I will demonstrate considerations that apply to 3 properties of a distributed system laid out in Martin Kleppmann's "Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems" which are as following
- Reliability, specifically is the application fault tolerant?, how does it respond to faults?
- Scalability, specifically can this application scale to a high load of traffic (high load often being hard to define)?
- Maintainability, specifically is the application easy to maintain through its existance, whilst we won't cover this too much, it will be covered to some extent.

## What We Will Consider
- Idempotency, specifically no matter how many time you execute you the payment, you get the same result. 
- Consistency, specifically ensuring that this payment and the many steps that occur within it (reducing the balance, creating a payment transaction, sending a request to a payment provider) happens atomically, put another way, this payment either completely happens or doesn't happen at all.
- We also need to ensure consistency in the balance, we cannot let a customer spend money that they don't have.
- Scalability, as described earlier, this is an application that can scale to a high load, but often defining this is hard, without putting some unverified number of payments per second etc here, I will simply describe scalability as the "effective" use of Postgres and Elixir, meaning that we are using those tools in the way they are intented, with the features they provide to providing good performance. To this end, we need to ensure that we ensure we avoid exhausting Postgres transactions by having them execute for longer than necessary (> ~100ms), and also avoid too much row locking to ensure we don't get any deadlocks.
- Reliability, specifically when contacting the payment provider API, we need to deal with their system being down, potentially returning error responses, and even our system crashing mid payment.

## What Isn't In Scope
I want to ensure that anyone reading the application or this blog can take inspiration from the things that have had consideration put into them, but also not be mislead with things that haven't had too much consideration (I don't want people taking the bad with the good), here are some things that haven't been considered, which you may want consider to in your own system
- PPI and GDPR, storing payment data can be critical to get correct, ensure you store this data securely and wisely.
- Tests, whilst I have creating some tests, these are acceptance tests that demonstrate an end to end flow to prove the application works as suggested, however there is a lot going on in this code, you will want to consider your tests more wisely including more unit tests to check the individual pieces.
- General code architecture, whilst in general this follows the context/domain driven design structure, it is by no means perfect, for example there is quite a lot of logic in the Oban job.
- More constraints/validation to ensure the correct formatting of data which could be achieved by Ecto, Postgres constraints, or both.
- This payment provider emulation is just an example, I can make no guarantee that a third party you integrate with offers API's like the one demonstrated, however I will say that if such a critical system is unreliable or not idempotent this could be a big issue, whilst some of these patterns demonstrated can help mitigate against things such as the third party API having downtime, it cannot solve for everything, the applications we build are only as good as the systems we use.

# Flow
The state diagram defining the flow can be seen [here](https://mermaid.ink/img/pako:eNqNVcFu3CAQ_RXEsdqVrFV6qFXlkDSRWqlSm82trioCs1m0NlgYokTR_nsBYxu8tmNfbM-894YZhuEdU8kA57jRRMM3Tp4VqbYvu0Ig-_z59Bdtt9foVoH1_iJvFQj9qIhoCNVcihblqbMY9N6iYr0HYIZy8XxDSiIoDICRYwhubUH6h3zK0d5QCk0zEC9BKRcEs69oWUsaF-BWS1YV18AmmFPLfpBlCeyG0FOO7gkvjYLVodZx01QXOYPTY-0-RHohr9Rzbl-zu-rAwXzrcnb6o8BruKEAfU0PpmzJqT-m2ITjvovKMNFp1vzbgAE2uHpTB7h7BWpcRRNMb_Wwe9D06Iqu5AtnoELUgTED8OQ9aJ3u8qO0Va9L8O30_YDqgIZX3ugGcYHqoLMuQt8TqS_RZhIaJORykBmhj9NYJ9HZ7pSSUdTE3G2L6yVXntboxhT6nGUbdLX7gmwpltihI8cnPlK6yrJoxVNwLxT9910-8Ka8Yb5x4bpzesSNfV3C4yPgnsWaLxH7AnrQoT2X_-gR6GkAJeZOLTrOtnsCxFbN2D28RrvsA3p7tia4XxNusuzLoRSvY3IwjaZfN4pCaXx4YrtduJ1GCppaigYWqUnaZJ7XxUiXNVZJvXiDK1AV4cxetX5MFVgfoYIC5_aTEXUqcCHOFkeMlvs3QXGulYENNjUbbmacH0jZWCswrqX62d7d_go__wfXA46t?type=png) (it doesn't fit on the blog screen).

An overview of the complete flow is as follows
- When a payment is initiated we create a Postgres transaction, within that transaction we reduce the customer account balance, insert an Oban job to be executed later, and then insert a payment transaction into our system.
- If any of those steps fail, the whole Postgres transaction is rolled back and therefore the payment is cancelled.
- If they succeed, then the payment is pending.
- The Oban job worker picks up the queued job and executes it.
- The first step is to check if the payment has already been created by the provider (its not impossible that the job was executing previously, successful hit the payment provider, but during this process our VM crashed, at which the job will be started again, but from the payment providers point of view it was already successful), this can lead to 2 possible outcomes.
  - It already exists in the provider, in which case we just set our payment transaction to completed and we are done (payment complete).
  - It does not exist, in which case we attempt to create it. The provider could result in different possible results.
    - If it returns a 400, we have sent it an invalid payment, in which case we mark the payment transaction as cancelled, and reinstate the balance to the customer account.
    - If it returns a 429 or 500 error (and other possible errors), we know that this is temporary so we mark the job as failed, and assuming it hasn't exceeded the failed count, it will re-attempt the job with a backoff period.
    - It returns 200, which means its successful and we can therefore mark the payment transaction as completed.
- If any of the steps fail unexpectedly within the Oban job, this will also follow the failed/retry flow Oban provides (this is omitted from the diagram to reduce bloating the diagram, and because its an Oban guarantee).
- Once the Oban job as completed, we can consider the payment complete, however if it was a 400 error its considered cancelled.

# Patterns
I will now go into the patterns that has been used to achieve scalablity and reliability in this payment process.

## ACID
Arguably the most important reliability guarantee we can make in this system, is in ensuring we only allow a payment to be processed if account has enough balance, otherwise the customer could be sending money they simply don't have. Its also important to us that if any step fails, whether that be communicating with the provider, creating the payment transaction or reducing the balance, non of these steps succeed, leaving our system in an inconsistent state. Luckily we are using Postgres which is an ACID compliant database which allows us to make guarantees on the consistency of our data (there are several resources to that can be read around ACID, [here](https://www.ibm.com/docs/en/cics-ts/5.4?topic=processing-acid-properties-transactions) is a simple one from IBM).

However whilst Postgres is ACID compliant, its very easy to fall into some potential issues, lets investigate theses one by one, and show how we can apply distributed patterns to solve them.

### No ACID at All
One possible troubled approach would be to use no ACID at all, and simply produce some code as follows
```elixir
defp do_send_money(%CreatePaymentRequest{} = create_payment_request) do
  account = Repo.get!(Account, create_payment_request.account_id)
  
  if Decimal.compare(account.balance, create_payment_request.amount) == :gte do
    account 
    |> change(balance: Decimal.add(account.balance, -create_payment_request.amount))
    |> Repo.update()
  end
end
```

This could easily lead to consistency issues as its not concurrency safe. Lets imagine a scenario, where Pete and Becca have access to the same account, and that acount has £50 in its balance.
- Pete creates a payment for £40.
- Concurrently Becca creates a payment for £20.
- Due to concurrency, its possible that both Becca and Pete access this account at the same time, and receive the account struct with £50 in its balance.
- Both do a greater than check and succeed (as £40 and £20 is above £50).
- Both subsequently update the balance and we have a negative balance which isn't allowed.

### Long Running Transactions
Given the non ACID approach isn't a solution, we can introduce Postgres transactions that provide us ACID guarantees, however there are some interesting concerns when using Postgres transactions, so its important to apply them correctly.

Firstly, always [check what transaction isolation you need](https://www.postgresql.org/docs/current/transaction-iso.html), in our case (for reasons that will become clear soon) we are fine with the default which is read committed. However its easy to end up assuming things about Postgres transactions that might not hold true, read the documentation to ensure you use the correct approach.

Secondly, consider how long a Postgres transaction is running for, when we communicate to Postgres with our code we have connection pools, which typically limit the amount of connections concurrently, we want to keep these as free as possible to handle multiple requests, moreover Postgres transactions use [MVCC](https://www.postgresql.org/docs/7.1/mvcc.html) which means it will create snapshots of data, and finally these transactions can halt other requests if they are modifying similar data. The longer these Postgres transactions exist for, the bigger chance they could provide scalability issues. Whilst that might not be a huge issue in some cases such as something like this
```elixir
defp do_send_money(%CreatePaymentRequest{} = create_payment_request) do
  Repo.transaction(fn ->
    account = Repo.get!(Account, create_payment_request.account_id)

    if Decimal.compare(account.balance, create_payment_request.amount) == :gte do
      account
      |> change(balance: Decimal.add(account.balance, -create_payment_request.amount))
      |> Repo.update()
    end
    
    # ...other stuff, like creating the payment
  end)
end
```
as the latency for the Postgres transaction to be created, followed by a select request, processing on the application server, and the performing a database update will likely be low, there are still ways we can provide even swifter performance here which will be demonstrated shortly.

However its even possible for this to become a bigger issues if we do some heavy network requests within this Postgres transaction. Whilst I want to avoid describing other patterns yet, its crucial that when we contact the payment provider, it does this as part of a distributed transaction, otherwise consistency could be a huge issue. Imagine a case where we reduce the balance, commit the transaction and try to hit the provider, which fails, we will have reduced the balance but failed in communication with the payment provider, this is known as the [dual write](https://thorben-janssen.com/dual-writes/) issue (which will will cover in more depth later). A naive way to solve this would be to communicate with the payment provider from within the Postgres transaction itself, something like this
```elixir
defp do_send_money(%CreatePaymentRequest{} = create_payment_request) do
  Repo.transaction(fn ->
    account = Repo.get!(Account, create_payment_request.account_id)

    if Decimal.compare(account.balance, create_payment_request.amount) == :gte do
      account
      |> change(balance: Decimal.add(account.balance, -create_payment_request.amount))
      |> Repo.update()
    end

    PaymentProviderClient.create_transaction(%{...data})
    # ...other stuff, like creating the payment
  end)
end
```

this would now mean we are performing network operations within the Postgres transaction which can be a big concern for scalability. You can even read about this in Jon Chew's Airbnb [post](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb). Whilst this particular problem will be covered later in more depth, its easy to see how Postgres transactions can be misused.

### Atomic Increments and Constraints
A simple solution to ensure ACID guarantees in a scalable way is use a mix of [atomic increments](https://mbukowicz.github.io/databases/2020/05/03/simple-locking-use-cases-in-postgresql.html) and [Postgres check constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

By using a check constraint in our database schema on the account we can do this (full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/priv/repo/migrations/20230202161530_create_accounts.exs#L15))
```elixir
create constraint(:accounts, :balance_must_be_positive, check: "balance >= 0")
```
which ensures that the balance stays positive whenever we run an update statement.

Coupled with an atomic increment we can ensure consistency within our Postgres transaction (full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments.ex#L136))
```elixir
Ecto.Multi.update_all(
  multi,
  :update_balance,
  from(a in Account, where: a.id == ^account_id),
  inc: [balance: -amount]
)
``` 
This performs a consistent update, which looks something like this in SQL which [safe for concurrency](https://mbukowicz.github.io/databases/2020/05/03/)
```sql
update accounts
set balance = balance - $2
where id = $1
```
This also doesn't provide inconsistency with other Postgres transactions, as we know that other Postgres transactions wanting to edit the same data will be locked until the completion of the Postgres transaction (see part 13.2.1 chapter 2 of [this](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED) documentation).

### Pairing with a Transaction
By pairing this atomic increment and constraint check with a Postgres transaction, we guarantee a nice scalable ACID operation. Whereby we safety reduce the balance, create a payment transaction and prepare the payment job within one Postgres transaction which performs swiftly. We couple this together with Ecto's wonderful [Multi library](https://hexdocs.pm/ecto/Ecto.Multi.html) which allows us to create a nice clean pipeline approach to our transaction steps, like so (full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments.ex#L107))
```elixir
payment_idempotency_key = Ecto.UUID.generate()

Ecto.Multi.new()
|> prepare_update_balance_step(create_payment_request)
|> prepare_payment_job_step(payment_idempotency_key, create_payment_request)
|> prepare_create_transaction_step(payment_idempotency_key, create_payment_request)
|> execute_masked_transaction()
|> case do
  {:ok, %{create_transaction: new_transaction}} -> {:ok, new_transaction}
  {:error, _step, %Ecto.Changeset{errors: errors}, _rest} -> {:error, errors}
  otherwise -> otherwise
end
```
**Please note** that the `execute_masked_transaction` catches any constraint errors we receive from Postgres, such as negative balance, whilst its not ideal to do exception handling in Elixir, this is an atomic increment, which means we cannot make use of [Ectos constraint check function](https://hexdocs.pm/ecto/Ecto.Changeset.html#check_constraint/3).

### ACID Conclusion
This summerises the first set of patterns that have been applied which includes
- ACID
- Atomic increments
- Check constraints

By using these approaches and patterns we have guaranteed scalability, reliability and even maintainability
- scalability as these are fast and efficent Postgres transactions
- reliability because we now know the payments are not left if an inconsistent state, specifically the balance is safely reduced and stays positive (or zero), and if the payment fails in the creation of any steps the whole payment is cancelled.
- maintainable as this doesn't require any bloated tools or libraries, its achievable within nothing more than Ecto Multis and Ecto Changesets.

## Distributed Transactions
In the previous section I made these 2 following claims
1. "Its also important to us that if any step fails, whether that be communicating with the provider, creating the payment transaction or reducing the balance, non of these steps succeed, leaving our system in an inconsistent state.".
2. "and if the payment fails in the creation of any steps the whole payment is cancelled.".

However I also made this claim "its crucial that when we contact the payment provider, it does this as part of a distributed transaction, otherwise consistency could be a huge issue. Imagine a case where we reduce the balance, commit the transaction and try to hit the provider, which fails, we will have reduced the balance but failed in communication with the payment provider", whilst describing how doing this in a Postgres transaction can lead to significent scalability issues.

At face value these two things seem mutually exclusive, on the one hand we need contact with this payment provider over HTTP to be part of the transaction, but this cannot easily happen because 1) its an external system 2) this system has to be scalable and thus avoid HTTP calls whilst we are in operation of our Postgres transaction. So how do we reconcile this?

### Eventual Consistency
In distributed systems, there is the concept of [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency), essentially where the state of an application in a multi node system becomes consistent with the "true" state over time. When you read about eventual consistency you will see how it often is used to describe some NoSQL high availability systems. However we can use this concept in our example, where we have 2 systems that need to be up to date with the state of the payment. Specifically our system, which has a balance and reference to the payment transaction, and the payment provider who needs to be aware of the payment to physically shift the funds. However there is no reason in this case that they both need to consistent in one atomic operation. We can simply make the system eventually consistent, by promising that both systems will be in sync with one another over time. 

There are some distributed patterns we can apply here to ensure we have a reliable eventually consistent system. Firstly before delving into these patterns, lets discuss the bare minimum we must ensure at the creation of a payment.
- The balance has to be avaliable.
- The balance has to be reduced, even though the payment itself will be eventual and not within the Postgres transaction.
- Mark the payment transaction on our side as pending, as given this eventually consistent approach, we make no certainty the transaction will even succeed.

### Background Job Processing
The next step we have taken to create an eventually consistent system is to use [Oban](https://github.com/sorentwo/oban). Oban is a reliable background job processing library, where jobs can be queued and executed by Oban and ensure strong reliability. Oban ensures uniqueness, specifically a unique job will only be ran by one node at once (but not exactly once processing, more on this later), it sits on top of Postgres which gives us consistency guarantees and also ensure we don't have to maintain additional systems, improving the maintainability of our payment system, its also [really performant and scalable](https://getoban.pro/articles/one-million-jobs-a-minute-with-oban).

By using this we ensure that our payment provider will receive the payment, even in the face of faults, it also allows for scalability due to its great performance.

### Outbox Pattern
Even though we are using a background job processing, we need to meet our minimum initial consistency guarantees (specifically that the balance is reduced, the transaction is created in a pending state, and we ensure a payment to the provider will be attempted in the future). We could easily break these guarentees if we did something like the following
```elixir
defp do_send_money(%CreatePaymentRequest{} = create_payment_request) do
  payment_idempotency_key = Ecto.UUID.generate()

  Ecto.Multi.new()
  |> prepare_update_balance_step(create_payment_request)
  |> prepare_create_transaction_step(payment_idempotency_key, create_payment_request)
  |> execute_masked_transaction()
  |> case do
    {:ok, %{create_transaction: new_transaction}} -> 
      # See here!
      Oban.insert(SendPaymentViaProvider, ...job_args)
      {:ok, new_transaction}
    {:error, _step, %Ecto.Changeset{errors: errors}, _rest} -> {:error, errors}
    otherwise -> otherwise
  end
end
```
This could cause an issue, as we reduced the balance, and created the payment transaction in a Postgres transaction, however we are creating the background job after this fact. Imagine a circumstance where the Postgres transaction succeeds, but the Oban job creation doesn't. Now we have a system where we have a transaction in our system, and a reduced balance, but no guarantee the payment provider will do anything about this. This is often described as the [dual write](https://thorben-janssen.com/dual-writes/) issue, whereby you are trying update multiple systems, but something fails, and thus the system is in a inconsistent state. Whilst we could use some kind of read repair, or write repair (see Conflict resolution [here](https://en.wikipedia.org/wiki/Eventual_consistency)), we have an easier way with Oban. As stated earlier it sits on top of Postgres, so it can take part in the same Postgres transaction like so.
```elixir
Ecto.Multi.new()
|> prepare_update_balance_step(create_payment_request)
|> prepare_payment_job_step(payment_idempotency_key, create_payment_request) #See here
|> prepare_create_transaction_step(payment_idempotency_key, create_payment_request)
|> execute_masked_transaction()
```
This can be best described by the [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html), whilst the context in the post is a bit different, the solution is the same. Rather than hitting the payment provider after reducing the balance and creating the payment transaction, thus hitting a dual write issue, we tell Oban to queue this job within the same initial Postgres transaction, therefore we ensure its an all or nothing situation. Either a step fails, and everything is rolled back, including this eventual consistency step with the provider payment, or all steps succeed and we have guarenteed that Oban eventually will communicate with this payment provider.

**Dont fear** if your unable to use a database based background job processor (although they certainly make things simpler), its not uncommon for some of these systems to run on entirely different stores, such as [SQS](https://aws.amazon.com/sqs/) or [Sidekiq](https://github.com/sidekiq/sidekiq) (which uses Redis), you can use the Outbox Pattern here aswell, whereby you create an Outbox table in your database, insert into that table as part of the Postgres transaction with the arguments the job requires. And then use something like [Polling Publisher](https://microservices.io/patterns/data/polling-publisher.html) Pattern, where you have a cron job that reads this table, and pushes any new items onto the given background job processor and then deletes it from the outbox.

By using the Outbox Pattern, we make a reliability guarantee where we ensure our system will make an attempt to communicate with the payment provider in the future as part of the initial Postgres transaction.

### Saga Pattern
At this point, we should have a reliable and scalable Postgres transaction in our system, we should also understand that our system will use eventual consistency to ensure the payment provider sends the payment whilst also ensuring we keep the initial Postgres transaction efficent, this is achievable by using a background job processor and the Outbox Pattern. However, at this point we only have a pending payment transaction on our side, and a guarantee on our system that we will contact the payment provider. So where do we go from here? We can use the [Saga Pattern](https://microservices.io/patterns/data/saga.html) which is a way we can manage a distributed transactions across multiple machines. Again in the post the context is slightly different, but the solution is the same whereby we are performing a distributed transaction that spans multiple systems, and most crucially we will use the concept of a "compensating transaction" to deal with failures.

Within the job we contact the provider to create the transaction (see full code [here](https://github.com/MikeyBower93/ex_bank/blob/main/lib/ex_bank/payments/jobs/send_payment_via_provider.ex))
```elixir
%{
  amount: amount,
  account_id: account_id,
  idempotency_key: payment_idempotency_key,
  sender_account_number: sender_account_number,
  sender_sort_code: sender_sort_code,
  sender_account_name: sender_account_name,
  receiver_account_number: receiver_account_number,
  receiver_sort_code: receiver_sort_code,
  receiver_account_name: receiver_account_name
}
|> PaymentProviderClient.create_transaction()
|> case do
  # Success, we can complete the transaction
  %{status: status} when status in [200, 201] ->
    {:ok, _} = Payments.complete_transaction(payment_idempotency_key)
    :ok

  # Bad request we need to fail and cancel the transaction.
  %{status: status, body: error} when status in [400] ->
    {:ok, _} = Payments.reverse_transaction(payment_idempotency_key, account_id, error)
    :ok

  # Unexpected case, oban to retry with error message for logging.
  %{body: error} ->
    {:error, error}
end
```
We can see here that the job makes the attempt to contact the provider. Once we receive a response that we are willing to accept, we update the payment transaction on our side, to either set it as completed, or with the use of a "compensation transaction" (a Saga Pattern concept) where we realise the payment couldn't happen, so we reverse the payment transaction by reinstating the balance to the customer and setting our payment transaction to the failed state with an error message. 

Whilst on face value this might seem like it isn't ideal, as a provider error (400 response) means we have to reverse out of the payment, its better than a system that cannot scale because we attempt to do this all within one Postgres transaction. Unfortuantely you cannot always have your cake and eat it in distributed systems, we have to make a trade off with eventual consistency, to ensure reliability and scalability.

This way we have achieved a distributed transaction across two systems, using a Saga approach.

### Idempotency
We have a few more things to consider before we can consider our eventual consistency complete. As stated throughout this post, reliability is key, to this extent we need to consider some edge cases. What happens if this job fails mid way through its execution? We know that failures are picked up by Oban and a retry mechanism is used. However there is even a deeper possible issue, its not impossible that we have a VM or container crash half way through execution whilst we are either in the middle of telling the payment provider to make the payment, or just after whilst we are updating our payment transaction. In this case, Oban will retry this job on recovery as it knows that it never completed, which means it will start from the beginning again (this is known as [at least once processing](https://www.cloudcomputingpatterns.org/at_least_once_delivery/)).

We need to ensure that money isn't sent twice, so to this end we always need to check the payment provider first to see if the payment has already happened with the provider. By doing this we are ensuring our job is idempotent, so that if this job is executed again at any time, it will ensure a consistent state no matter how many times we execute this job. We do this by assigning a random UUID to our Postgres transaction during creation called `payment_idempotency_key` (see [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments.ex#L108)).

When we then create the transaction with the provider in the job, we send the ID as the idempotency key (see full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments/jobs/send_payment_via_provider.ex#LL36C5-L47C50))
```elixir
%{
  amount: amount,
  account_id: account_id,
  idempotency_key: payment_idempotency_key,
  sender_account_number: sender_account_number,
  sender_sort_code: sender_sort_code,
  sender_account_name: sender_account_name,
  receiver_account_number: receiver_account_number,
  receiver_sort_code: receiver_sort_code,
  receiver_account_name: receiver_account_name
}
|> PaymentProviderClient.create_transaction()
```

Then, when the job begins, we ensure we check to see if the payment already happened (see full code [here](https://github.com/MikeyBower93/ex_bank/blob/main/lib/ex_bank/payments/jobs/send_payment_via_provider.ex))
```elixir
@impl Oban.Worker
def perform(%Oban.Job{
      args: args
    }) do
  # Check to see if the transaction already exists
  # in cases of this job being cancelled due to a container crash
  # half through the job
  args = Map.new(args, fn {k, v} -> {String.to_existing_atom(k), v} end)

  args.payment_idempotency_key
  |> PaymentProviderClient.get_transaction()
  |> maybe_send_money(args)
end

# Doesn't exist, so lets create payment with the provider
defp maybe_send_money(%{status: 404}, %{
        amount: amount,
        account_id: account_id,
        receiver_account_number: receiver_account_number,
        receiver_sort_code: receiver_sort_code,
        receiver_account_name: receiver_account_name,
        payment_idempotency_key: payment_idempotency_key
      }) do
  %{
    account_name: sender_account_name,
    account_number: sender_account_number,
    account_sort_code: sender_sort_code
  } = Payments.get_account(account_id)

  %{
    amount: amount,
    account_id: account_id,
    idempotency_key: payment_idempotency_key,
    sender_account_number: sender_account_number,
    sender_sort_code: sender_sort_code,
    sender_account_name: sender_account_name,
    receiver_account_number: receiver_account_number,
    receiver_sort_code: receiver_sort_code,
    receiver_account_name: receiver_account_name
  }
  |> PaymentProviderClient.create_transaction()
  |> case do
    # Success, we can complete the transaction
    %{status: status} when status in [200, 201] ->
      {:ok, _} = Payments.complete_transaction(payment_idempotency_key)
      :ok

    # Bad request we need to fail and cancel the transaction.
    %{status: status, body: error} when status in [400] ->
      {:ok, _} = Payments.reverse_transaction(payment_idempotency_key, account_id, error)
      :ok

    # Unexpected case, oban to retry with error message for logging.
    %{body: error} ->
      {:error, error}
  end
end

# Transaction already exists, lets just ensure its marked as completed.
defp maybe_send_money(%{status: 200}, %{
        payment_idempotency_key: payment_idempotency_key
      }) do
  {:ok, _} = Payments.complete_transaction(payment_idempotency_key)
  :ok
end

# Unexpected case, oban to retry with error message for logging.
defp maybe_send_money(%{body: error}, _data) do
  {:error, error}
end
```
By doing this we have ensured that should the payment attempt again, it won't create duplicates.

### Distributed Transaction Conclusion
We can now see with the use of eventual consistency, we can guarantee scalability of the initial transaction, and then apply patterns such as Idempotency, the Saga Pattern, and the Outbox Pattern to ensure reliability when processing this payment with the provider.

## Other Techniques Used
The previous sections are the main points around how scalability and reliability is achieved. However some additional techniques have been used to achieve relability and scalability, which I will discuss for completeness.

### Circuit Breaker
When hitting the payment provider over HTTP we have to deal with potential failures, such as 429 rate limiting. Should we hit a rate limit, that limit could apply for quite some time, the last thing we want to is to continue hitting the provider in quick succession, eating up our HTTP connection pool sending unnecessary requests to the payment provider, given the possible latency to the payment provider, this could even slow down our Oban job queue if we have [queue limits](https://hexdocs.pm/oban/2.11.0/smart_engine.html#usage-and-configuration). To deal with this we can use the [Circuit Breaker Pattern](https://microservices.io/patterns/reliability/circuit-breaker.html) whereby we realise that we have hit an error such as a 429, and automatically return the 429 error to the client for subsequent requests without hitting the provider until some time elapses.

We do this using the [Fuse](https://github.com/jlouis/fuse) erlang library, along side [Tesla](https://hexdocs.pm/tesla/readme.html) which we use to make HTTP requests, which looks like this (see full code [here](https://github.com/MikeyBower93/ex_bank/blob/main/lib/ex_bank/payments/clients/payment_provider_client.ex))
{% raw %}
```elixir
defp client() do
  api_key = Application.get_env(:ex_bank, :payment_provider_api_key)

  middleware = [
    {Tesla.Middleware.BaseUrl, "https://payment_provider"},
    Tesla.Middleware.JSON,
    {Tesla.Middleware.Headers, [{"api_key", api_key}]},
    {Tesla.Middleware.Fuse,
      opts: {{:standard, 2, 10_000}, {:reset, 60_000}},
      keep_original_error: true,
      should_melt: fn
        {:ok, %{status: status}} when status in [429] -> true
        {:ok, _} -> false
        {:error, _} -> true
      end,
      mode: :sync}
  ]

  Tesla.client(middleware)
end
```
{% endraw %}
One key thing to note here, we could achieve this in a different way with Oban Pro, as you can set queue rate limits for a given timespan (see [here](https://hexdocs.pm/oban/2.11.0/smart_engine.html#usage-and-configuration)), which would guarantee we only fire x amount of requests to the payment provider within a particular time span, this could work well if we know our rate limits up front with the provider.

### Indexes
Although this is quite simple and common piece of advice, it cannot be understated, make sure you index your tables correctly! This ensures as the data grows over time, we can select data performantly. The indexes we created in this case can be seen in our migrations [here](https://github.com/MikeyBower93/ex_bank/tree/main/priv/repo/migrations). Whilst indexes are created automatically for primary keys, in our application we have some other access paths, for example selecting the transaction by `payment_idempotency_key` for example (see full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments.ex#L53))
```elixir
transaction = Repo.get_by!(Transaction, payment_idempotency_key: idempotency_key)
```

the index can be seen as follows (see full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/priv/repo/migrations/20230202161959_create_transactions.exs#L20))
```elixir
create index(:transactions, [:payment_idempotency_key])
```

Its very hard to give clear advise on indexing, as the right indexes, and index types depends on the way you utilise the data in the application. So its on a case by case circumstance, however I recommend reading [use the index luke](https://use-the-index-luke.com/) for good guidance around this topic. By creating indexes we ensure non linear performance, so as the data grows significently, our application stays performant ensuring scalability.

### Constaints
One of my favourite talks on Postgres and Ecto is by Todd Resudek (see [here](https://www.youtube.com/watch?v=mliO5_XhAKE&t=711s&ab_channel=ElixirConf)), the talk goes into more ways to utilise Postgres instead of building things at the application code level. One points he makes in the post is the "belts and suspenders" approach to data validation. Essentially that its quite common to verify data at multiple levels, all the way from the user form, through the the application code. However in my experience it often seems forgotten to be applied at the database level. Which in my view is the most important place to provide validation, as it ensure data consistency independant of how data is modified into your system. This provides a greater level of reliability ensuring that our data is valid at the root level.

We do this in our example by utilising unique indexes, non null checks, and foreign key indexes which can be seen in our migrations [here](https://github.com/MikeyBower93/ex_bank/tree/main/priv/repo/migrations).

### Numeric Types
Some people viewing this example might notice the use of decimals to store monetary amounts, and wonder about precision. Can we really guarantee a reliable system if we have inprecise data leading to issues when we do money comparisons or arithmetic? There are many ways to solve this, such as using the Elixir Money [library](https://hexdocs.pm/money/Money.html) which has Ecto data structures, through to storing money as integer values where the 2 trailing numbers represent the amount after the decimal point.

However in this case we went with a simple solution whereby Postgres allows us to use the "numeric" datatype which allows us to store precision (see [here](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-SERIAL)). Whats nice about this is we are able to declare the type as "numeric" in our migration, but treat it as a decimal in our Ecto schemas.

# Conclusion
In this post I have tried to take a real problem, such as making payments with several system, which could be applied to e-commerce sites through to fintech companies, and explain how we can apply distributed patterns to make it reliable, scalable and maintainable (although a lot more could be discussed around that) using Elixir.

The reliability patterns/techniques we covered are
- ACID transactions
- Atomic increments
- The Outbox Pattern
- The Saga Pattern
- The Circuit Breaker Pattern
- Idempotency
- Background job processors
- Postgres constraints and data types
- Eventual consistency

The scalability patterns/techniques we covered are
- Indexes
- Eventual consistency
- Avoiding long running transactions
- Performant background job processors 

Whilst we didn't apply any particular patterns to maintainability, throughout this post I have tried to apply these patterns with as much simplicity as possible and achieved most of this with just Elixir libraries and Postgres. Maintainability is a big topic, but hopefully this can demonstrate how simplicity doesn't have to come at the cost of scalability or reliability.

One parting thought I would like to make is not unlike my point in my [previous post](https://mikeybower93.github.io/posts/loosely_coupled_phoenix_contexts). Specifically that on quite a few occasions I've seen a rejection of the need for microservices when using Elixir (this is just based on some conversations I've seen and had, this may not be the majority view), whilst this post is not to comment on that conversation, its to reiterate that behind both approaches (microservices and monoliths) we are dealing with distributed systems. Even within a monolith we are often communicating with a database and external systems, to such an end we can say that despite the architectural approach, both require use of distributed patterns. We should continue to learn from different architectural approaches in the community, and consider how such approaches can help us better our own.

Happy Coding!
