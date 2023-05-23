stateDiagram-v2
    [*] --> CreatePaymentTransaction
    state CreatePaymentTransaction {
        [*] --> ReducingBalance
        ReducingBalance --> CreatingPaymentJob: Success
        CreatingPaymentJob --> CreatingPendingTransaction: Success
        CreatingPendingTransaction --> Commited: Success
        ReducingBalance --> RolledBack: Failure
        CreatingPendingTransaction --> RolledBack: Failure
        CreatingPaymentJob --> RolledBack: Failure
        RolledBack --> [*]
        Commited --> [*]
    }
    CreatePaymentTransaction --> PaymentCancelled: Failure
    CreatePaymentTransaction --> PaymentPending: Successful
    PaymentPending --> PaymentJob
    state PaymentJob {
        [*] --> JobQueued
        JobQueued --> JobExecuting
        JobExecuting --> FetchingProviderPayment
        FetchingProviderPayment --> SettingTransactionToCompleted: If payment exists in provider
        FetchingProviderPayment --> CreatingProviderPayment: If payment does not exists in provider
        CreatingProviderPayment --> SettingTransactionToCompleted
        CreatingProviderPayment --> ProviderError
        ProviderError --> JobFailed: Error code 500, 429 etc
        ProviderError --> CancellingTransaction: Error code 400
        CancellingTransaction --> TransactionCancelled
        TransactionCancelled --> ReinstatingBalance
        ReinstatingBalance --> JobSuccessful
        SettingTransactionToCompleted --> JobSuccessful
        JobFailed --> failure_check
        failure_check --> JobCancelled: If failure count > 20
        failure_check --> JobQueued: If failure count < 20
        JobSuccessful --> [*]
        JobCancelled --> [*]
    }
    PaymentJob --> PaymentComplete: If a non 400 response
    PaymentJob --> PaymentCancelled: If a 400 response
    PaymentComplete --> [*]
    PaymentCancelled --> [*]
    