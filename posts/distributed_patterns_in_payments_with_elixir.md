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
The state diagram defining the flow can be seen [here](https://mermaid.live/edit#pako:eNqNVcFu3CAQ_RXEsdqVrFV6qFXlkDSRGilSms2trioCs1m0NlgYokTR_nsBYxtc27EvtmfeezMMw_CBqWSAc9xoouEHJy-KVNvXXSGQfX5_-YO220t0rcB6H8h7BUI_KSIaQjWXokV56iwGfbSoWO8RmKFcvFyRkggKA2DkGIJb2wMIZl-RdI72hlJomkFgHpxqtWneyecljR7UcmVVcQ1sgjGV9qMsS2BXhJ5ydEt4aRSsTnMdN01vkTM4PdbuQ6QX1pV6zu1rdlcdOJiv3Zqd_ijwGm4oQF_TgylbcuqPKXbBcd9FZZjoNGv-ZcAAG1y9qQPcvAE1rqIJprd62C1oenRFV_KVM1Ah6sCYAXjyHrROd_lJ2qrXJfh2-nlAdUDDG290g7hAddBZF6HvidSXaDMJDRJyOciM0OfLWCfR2W6UklHUxNxti-slV57W6MYU-pplG3Sx-4ZsKZbYoSPH0yJSusiyKOMpuBeK_vsuH3hT3jDfuHDdOT3ixr5uweMj4J7Fmi8R-wJ60KE9l3_pEehpACXmTi06zrZ7AsRWzdg9vES77BN6e7YmuN8TbpL2_0MpzmNyMI2mXzeKQml8eGK7XbidRgqaWooGFqnJssk8r4uRpjVWSb14gytQFeHMXrV-TBVYH6GCAuf2kxF1KnAhzhZHjJb7d0FxrpWBDTY1G27mzgiMa6nu26vb3-Dnf5YQjmI) (it doesn't fit on the blog screen).

An overview of the complete flow is as follows:
- When a payment is initiated we create a Postgres transaction, within that transaction we reduce the customer account balance, insert a payment transaction into our system, and then insert an Oban job to be executed later.
- If any of those steps fail, the whole transaction is rolled back and therefore the payment is cancelled.
- If they succeed, then the payment is pending.
- The Oban job worker picks up the queued job and executes it.
- The first step is to check if the payment has already been created by the provider (its not impossible that the job was executing previously, successful hit the payment provider, but during this process our VM crashed, at which the job will be started again, but from the payment providers point of view it was already successful), this can lead to 2 possible outcomes
  - It already exists in the provider, in which case we just set our transaction to completed and we are done (payment complete).
  - It does not exist, in which case we attempt to create it, the provider could result in different possible results
    - If it returns a 400, we have sent it an invalid payment, in which case we mark the transaction as cancelled, and reinstate the balance to the customer account.
    - If it returns a 429 or 500 error (and other possible errors), we know that this is temporary so we mark the job as failed, and assuming it hasn't exceeded the failed count, it will re-attempt the job with a backoff period.
    - It returns 200, which means its successful and we can therefore mark the transaction as completed.
- If any of the steps fail unexpectedly within the Oban job, this will also follow the failed/retry flow Oban provides (this is omitted from the diagram to reduce bloating the diagram, and because its an Oban guarantee).
- Once the Oban job as completed, we can consider the payment complete, however if it was a 400 error its considered cancelled.

# Patterns
TODO - list the patterns, what they solve with code examples

# Conclusion
TODO - round things up