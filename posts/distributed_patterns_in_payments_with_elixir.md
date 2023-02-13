# Distributed Patterns in Payments with Elixir

# Introduction
Recently I was reading [this](https://t.co/T6CjcTfn7g) excellent post by Jon Chew from Airbnb about how they achieve consistency, performance and correctness in their microservice architecture when dealing with payments internally. This got me reflecting on a number things, such as my personal experiences during my career with such problems, and also considering excellent books I had read about distributed systems such as Martin Kleppmann's "Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems" and Chris Richardson's "Microservices Patterns: With Examples in Java".

I personally find that it can often be hard to piece together different information from blogs, books, small code examples etc to paint the full picture, this is unfortuantely true when trying to apply this to a language I'm passionate about (Elixir), as these examples are often in Java or Object Oriented heavy books. To that end, I wanted to create a fairly extensive example in Elixir about how you tie together these distributed patterns to achieve reliability and scalability in payments. The example is demonstrated in [this](https://github.com/MikeyBower93/ex_bank) repository called "ex_bank", this is to demonstrate a small banking application, where a customer can have an account, which has a balance, and that customer can send money from that account to someone via their sort code and account number, this in turn has to reduce the balance on our system, create a transaction, and notify a payment provider to send the money.

# Considerations
There are a number of considerations that need to be made in such an application, but before listing these, I want to lay out what "types" of considerations that are being made here, as any application can have a number of considerations, from scalability, to testing, to deployment etc, however we are dealing specifically with the questions around distributed systems here. To this end, I will demonstrate considerations that apply to 3 properties of a distributed system laid out in Martin Kleppmann's "Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems" which are as following
- Reliability, specifically is the application fault tolerant?, how does it respond to faults?
- Scalability, specifically can this application scale to a high load of traffic (high load often being hard to define)?
- Maintainability, specifically is the application easy to maintain through its existance, whilst we won't cover this too much, it will be covered to some extent.

## What We Will Consider
- Idempotency, specifically no matter how many time you execute you the payment, you get the same result. 
- Consistency, specifically ensuring that this payment and the many steps that occur within it (reducing the balance, creating a transaction, sending a request to a payment provider) happens atomically, put another way, this payment either completely happens or doesn't happen at all.
- We also need to ensure consistency in the balance, we cannot let a customer spend money that they don't have.
- Scalability, as described earlier, this is an application that can scale to a high load, but often defining this is hard, without putting some unverified number of payments per second etc here, I will simply describe scalability as the "effective" use of Postgres and Elixir, meaning that we are using those tools in the way they are intented, with the features they provide to providing good performance. To this end, we need to ensure that we ensure we avoid exhausting transactions by having them execute for longer than necessary (> ~100ms), and also avoid too much row locking to ensure we don't get any deadlocks.
- Reliability, specifically when contacting the payment provider API, we need to deal with their system being down, potentially returning error responses, and even our system crashing mid payment.

## What Isn't In Scope
I want to ensure that anyone reading the application or this blog can take inspiration from the things that have had consideration put into them, but also not be mislead with things that haven't had too much consideration (I don't want people taking bad with the good), here are some things that haven't been considered, which you may want consider to in your own system
- PPI and GDPR, storing payment data can be critical to get correct, ensure you store this data securely and wisely.
- Tests, whilst I have creating some tests, these are acceptance tests that demonstrate an end to end flow to prove the application works as suggested, however there is a lot going on in this code, you will want to consider your tests more wisely including more unit tests to check the individual pieces.
- General code architecture, whilst in general this follows the context/domain driven design structure, it is by no means perfect, for example there is quite a lot of logic in the Oban job.
- More constraints/validation to ensure the correct formatting of data which could be achieved by Ecto, Postgres constraints, or both.
- This payment provider emulation is just an example, I can make no guarantee that a third party you integrate with offers API's like the one demonstrated, however I will say that if such a critical system is unreliable or not idempotent this could be a big issue, whilst some of these patterns demonstrated can help mitigate against things such as an the third party API having downtime, it cannot solve for everything, the applications we build are only as good as the systems we use.

# Flow
The state diagram defining the flow can be seen [here](https://mermaid.ink/img/pako:eNqNVcFu3CAQ_RXEsdqVrFV6qFXlkDSRGilSms2trioCs1m0NlgYokTR_nsBYxtc27EvtmfeezMMw_CBqWSAc9xoouEHJy-KVNvXXSGQfX5_-YO220t0rcB6H8h7BUI_KSIaQjWXokV56iwGfbSoWO8RmKFcvFyRkggKA2DkGIJb2wMIZl-RdI72hlJomkFgHpxqtWneyecljR7UcmVVcQ1sgjGV9qMsS2BXhJ5ydEt4aRSsTnMdN01vkTM4PdbuQ6QX1pV6zu1rdlcdOJiv3Zqd_ijwGm4oQF_TgylbcuqPKXbBcd9FZZjoNGv-ZcAAG1y9qQPcvAE1rqIJprd62C1oenRFV_KVM1Ah6sCYAXjyHrROd_lJ2qrXJfh2-nlAdUDDG290g7hAddBZF6HvidSXaDMJDRJyOciM0OfLWCfR2W6UklHUxNxti-slV57W6MYU-pplG3Sx-4ZsKZbYoSPH0yJSusiyKOMpuBeK_vsuH3hT3jDfuHDdOT3ixr5uweMj4J7Fmi8R-wJ60KE9l3_pEehpACXmTi06zrZ7AsRWzdg9vES77BN6e7YmuN8TbpL2_0MpzmNyMI2mXzeKQml8eGK7XbidRgqaWooGFqnJssk8r4uRpjVWSb14gytQFeHMXrV-TBVYH6GCAuf2kxF1KnAhzhZHjJb7d0FxrpWBDTY1G27mzgiMa6nu26vb3-Dnf5YQjmI?type=png) (it doesn't fit on the blog screen).

An overview of the complete flow is as follows
- When a payment is initiated we create a Postgres transaction, within that transaction we reduce the customer account balance, insert a payment transaction into our system, and then insert an Oban job to be executed later.
- If any of those steps fail, the whole transaction is rolled back and therefore the payment is cancelled.
- If they succeed, then the payment is pending.
- The Oban job worker picks up the queued job and executes it.
- The first step is to check if the payment has already been created by the provider (its not impossible that the job was executing previously, successful hit the payment provider, but during this process our VM crashed, at which the job will be started again, but from the payment providers point of view it was already successful), this can lead to 2 possible outcomes.
  - It already exists in the provider, in which case we just set our transaction to completed and we are done (payment complete).
  - It does not exist, in which case we attempt to create it, the provider could result in different possible results.
    - If it returns a 400, we have sent it an invalid payment, in which case we mark the transaction as cancelled, and reinstate the balance to the customer account.
    - If it returns a 429 or 500 error (and other possible errors), we know that this is temporary so we mark the job as failed, and assuming it hasn't exceeded the failed count, it will re-attempt the job with a backoff period.
    - It returns 200, which means its successful and we can therefore mark the transaction as completed.
- If any of the steps fail unexpectedly within the Oban job, this will also follow the failed/retry flow Oban provides (this is omitted from the diagram to reduce bloating the diagram, and because its an Oban guarantee).
- Once the Oban job as completed, we can consider the payment complete, however if it was a 400 error its considered cancelled.

# Patterns
I will now go into the patterns that has been used to achieve scalable and reliability in this payment process.

## ACID
Arguably the most important reliability guarantee we can make in this system, is in ensuring we only allow a payment to be processed if account has enough balance, otherwise the customer could be sending money they simply don't have. Its also important to us that if any step fails, whether that be communicating with the provider, creating the transaction or reducing the balance, non of these steps succeed, leaving our system in an inconsistent state. Luckily we are using Postgres which is an ACID compliant database which allows us to make guarantees on the consistency of our data (there are several resources to that can be read around ACID, [here](https://www.ibm.com/docs/en/cics-ts/5.4?topic=processing-acid-properties-transactions) is a simple one from IBM).

However whilst Postgres is ACID compliant, its very easy to fall into some potential issues, lets investigate theses one by one, and show how we can apply distributed patterns to solve them.

**No ACID at All**
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

This could easily lead to consistency issues as its not concurrency safe. Lets imagine a scenario, where Pete and Becca have access to the same account, and that acount has £50 in it.
- Pete creates a payment £40
- Concurrently Becca creates a payment for £20.
- Due to concurrency, its possible that both Becca and Pete access this account at the same time, and receive the account struct with £50 in it.
- Both do greater than check and succeed (as both £40 or £20 is above £50).
- Both subsequently update the balance and we have a negative balance which isn't allowed.

**Long Running Transactions**
Given the non ACID approach isn't a solution, we can introduce transactions that provide us ACID guarantees, however there are some interesting concerns when using transactions, so its important to apply them correctly.

Firstly, always [check what transaction isolation you need](https://www.postgresql.org/docs/current/transaction-iso.html), in our case (for reasons that will become clear soon) we are fine with the default which is read committed. However its easy to end up assuming things about transactions that might not hold true, read the documentation to ensure you use the correct approach.

Secondly, consider how long a transaction is running for, when we communicate to Postgres with our code we have connection pools, with typically limit the amount of connections concurrently, we want to keep these as free as possible to handle multiple requests, moreover transactions use [MVCC](https://www.postgresql.org/docs/7.1/mvcc.html) which means it will create snapshots of data, and finally these transactions can halt other requests if they are modifying similar data. The longer these transactions exist for, the bigger they could provide scalability issues. Whilst that might not be a huge issue in some cases such as something like this
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
as the latency for the transaction to be created, followed by a select request, processing on the application server, and the performing a database update will likely be low, there are still ways we can provide even swifter performance here which will be demonstrated shortly.

However its even possible for this to become, a bigger issues if we do some heavy network requests within this transaction. Whilst I want to avoid describing other patterns yet, its crucial that when we contact the payment provider, it does this as part of a distributed transaction, otherwise consistency could be a huge issue. Imagine a case where we reduce the balance, commit the transaction and try to hit the provider, which fails, we will have reduced the balance but failed in operating with the payment provider, this is known as the [dual write](https://thorben-janssen.com/dual-writes/) issue (which will will cover in more depth later). A naive way to solve this would be to communicate with the payment provider from within the transaction itself, something like this
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

this would now mean we are performing network operations within the transaction which can be a big concern for scalability. You can even read about this in Jon Chew's Airbnb [post](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb). Whilst this particular problem will be covered later in more depth, its easy to see how transactions can be misused.

**Atomic Increments and Constraints**
A simple solution to ensure ACID guarantees in a scalable way is use a mix of [atomic increments](https://mbukowicz.github.io/databases/2020/05/03/simple-locking-use-cases-in-postgresql.html) and [Postgres check constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

By using a check constraint in our database schema on the account we can do this (full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/priv/repo/migrations/20230202161530_create_accounts.exs#L15))
```elixir
create constraint(:accounts, :balance_must_be_positive, check: "balance >= 0")
```
which ensures that the balance stays positive whenever we run an update statement.

Coupled with an atomic increment we can ensure consistency within our transaction (full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments.ex#L136))
```elixir
Ecto.Multi.update_all(
  multi,
  :update_balance,
  from(a in Account, where: a.id == ^account_id),
  inc: [balance: -amount]
)
``` 
This performs a consistent update, which looks something like this in SQL which [safe for concurrency](https://mbukowicz.github.io/databases/2020/05/03/)
```SQL
update accounts
set balance = balance - $2
where id = $1
```
This also doesn't provide inconsistency with other transactions, as we know that other transactions wanting to edit the same data will be locked until the completion of the transaction (see part 13.2.1 chapter 2 of [this](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED) documentation).

**Pairing with a Transaction**
By pairing this atomic increment and constraint check with a transaction, we guarantee a nice scalable ACID operation. Whereby we safety reduce the balance, create a transaction and prepare the payment job within one transaction which performs swiftly. We couple this together with Ecto's wonderful [Multi library](https://hexdocs.pm/ecto/Ecto.Multi.html) which allows us to create a nice clean pipeline approach to our transaction steps, like so (full code [here](https://github.com/MikeyBower93/ex_bank/blob/1d133e21b350b304be2d98e84364451ef4a89153/lib/ex_bank/payments.ex#L107))
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
Please note that the `execute_masked_transaction` catches any constraint errors we receive from Postgres, such as negative balance, whilst its not ideal to do exception handling in Elixir, this is an atomic increment, which means we cannot make use of [Ecto's constraint check function](https://hexdocs.pm/ecto/Ecto.Changeset.html#check_constraint/3).

**ACID Conclusion**
This summerises the first set of patterns that have been applied which includes
- ACID
- Atomic increments
- Check constraints

By using these approaches and patterns we have guaranteed scalability, reliability and even maintainability
- scalability as these are fast an efficent database transactions
- reliable because we now know the payments are not left if an inconsistent state, specifically the balance is safely reduced and stays positive (or zero), and if the payment fails in the creation of any steps the whole payment is cancelled.
- maintainable as this doesn't require any bloated tools or libraries, its achievable within nothing more than Ecto Multis and Ecto Changesets.

## Distributed Transactions
Expand on previous section about dual write issue, and how we use eventual consistency.
Includes
- dual write
- eventual consistency
- outbox pattern
- sagas

Note how "and if the payment fails in the creation of any steps the whole payment is cancelled", isn't strictly true as we cannot for scalability reasons apply the provider request in the transaction, but with the use of eventual consistency, we make an eventual guarantee of a good state, the above transaction makes a promise that it will contact the provider through an oban job.

# Conclusion
TODO - round things up
Distributed patterns in Elixir monoliths still being relevant.