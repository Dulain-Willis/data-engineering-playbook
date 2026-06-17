# Question

We're a consumer lending company offering personal loans, auto loans, and mortgages, and each of those products carries its own standard interest rate and term length. A customer can hold several loans at once, and every payment lands as its own transaction against a loan. Design a schema that lets the operations team read each loan's outstanding balance from those payments and the risk team flag delinquent accounts.

<br>

---
# Schema

```text
loan_types
    loan_type_id INT PK 
    loan_type_name VARCHAR
    default_interest_rate DECIMAL
    default_term_length INT

customers 
    customer_id BIGINT PK 
    customer_name VARCHAR 

loans 
    loan_id BIGINT PK
    customer_id FK 
    loan_type_id FK 
    principal_amount DECIMAL
    status VARCHAR

loan_transactions
    transaction_id BIGINT 
    loan_id FK
    paid_amount DECIMAL
    paid_at TIMESTAMP
    transaction_type VARCHAR 

loan_scheduled_payments 
    scheduled_payment_id BIGINT
    loan_id FK
    expected_amount DECIMAL
    due_date DATE 
```

<br>

---
# SQL Queries

## Query: A given loan's outstanding balance
```SQL
with 

payments_by_loan as (

    select 
        loan_id,
        sum(paid_amount) as total_paid

    from loan_transactions
    
    where transaction_type = 'payment'

    group by loan_id 

)

select
    loans.loan_id,
    loans.principal_amount - coalesce(payments.total_paid, 0) as outstanding_balance

from loans 

left join payments 
    on loans.loan_id = payments.loan_id 

where loans.status = 'active'
```

<br>

## Query: Delinquent accounts
```SQL
with

expected_payments_by_loan as (

    select
        loan_id,
        sum(expected_amount) as total_expected_due

    from loan_scheduled_payments

    where due_date < current_date

    group by loan_id

),

payments_by_loan as (

    select
        loan_id,
        sum(paid_amount) as total_paid

    from loan_transactions

    where transaction_type = 'payment'

    group by loan_id

)

select
    loans.loan_id,
    expected_payments.total_expected_due,
    coalesce(payments.total_paid, 0) as total_paid,
    expected_payments.total_expected_due - coalesce(payments.total_paid, 0) as past_due_amount

from expected_payments_by_loan expected_payments

inner join loans
    on expected_payments.loan_id = loans.loan_id

left join payments_by_loan payments
    on expected_payments.loan_id = payments.loan_id

where loans.status = 'active'
    and coalesce(payments.total_paid, 0) < expected_payments.total_expected_due
```

<br>

---
# Notes 

This is a normalization puzzle dressed up as a lending system. The real skill being probed: can you tell which numbers are facts you store and which are answers you compute? The trap is outstanding balance. It looks like a column you keep on the loan, so candidates store it and try to keep it in sync. Cache it and the first missed update silently drifts the stored number away from the ledger, and now no two reports agree. Derive it from the transaction log and it can never be wrong.

<br>

## Trick to Solving
When a prompt lists several "types" of something (personal, auto, mortgage) with their own rates and terms, the trick is to lift those attributes into a dimension. Before drawing tables, a strong candidate asks: is outstanding balance a stored column or a derived aggregate?

- Pull loan types into a dim table with rate and term columns
- Keep loans as the instance fact of a customer taking a loan
- Model every payment as a row in loan_transactions
- Derive outstanding balance via SUM, never store it

<br>

## Break down the requirements
1. Identify the four entities 
customers, loan_types, loans, loan_transactions. Each has a clean grain and no redundant attributes.

2. Normalize loan types
Rate, term, and product name belong on loan_types. Repeating them per loan row invites update anomalies when the product team renames a product.

3. Grain of loan_transactions
One row equals one money movement on one loan: payment, disbursement, fee, or refund. paid_amount is signed or typed.

4. Compute outstanding balance
principal_amount - SUM(paid_amount) via a GROUP BY on loan_id. The aggregate is always in sync because the underlying fact is the ledger.

<br>

## Why this works
Deriving balance from a ledger is the accounting-grade pattern. Storing balance as a column looks efficient until the first missed update creates drift, at which point reconciliation is a nightmare. A SUM over a properly indexed fact is both correct and fast enough.

### Interviewers watch for
A strong candidate says "the balance is a derived view" out loud and pushes back on any design that caches it as an UPDATEable column without a corresponding event row.

### Common pitfall
Storing outstanding_balance as a column on loans and keeping it in sync via triggers. The first replication lag or missed trigger creates a silent drift between the ledger and the balance, which is exactly the failure the ledger model exists to prevent.

<br>

## Trade-offs and alternatives

### ✕ Derived balance from ledger
Balance is a SUM over loan_transactions at read time.
- One source of truth
- No drift possible
- Read cost scales with transaction count per loan

### ✓ Stored balance column
Balance maintained as an UPDATEable column on loans.

- O(1) read per loan
- Silent drift on missed updates
- Reconciliation jobs become mandatory

<br>

---
# Common Follow-Up Questions

<br>

## How do you handle a refinance where an old loan is closed and a new one is opened?
Tests whether the candidate models refinance as a status transition plus a parent_loan_id link.

**ANSWER:** You wouldn't just go in and update the old loan's fields to reflect the new terms — that destroys the history of what the borrower originally agreed to. Instead, you keep the old loan as-is and just flip its status to something like 'refinanced'. Then you create a brand new loan row with the new terms and add a field like parent_loan_id that points back to the old loan. That way you've got a clean link between the two — you can always trace back and see what the original loan looked like. You could also throw in opened_at and closed_at timestamps on the loans table so you know exactly when each one started and ended.


| loan_id | customer_id | parent_loan_id | principal_amount | status      | opened_at  | closed_at  |
|---------|-------------|----------------|------------------|-------------|------------|------------|
| 101     | 55          | NULL           | 25000.00         | refinanced  | 2023-01-15 | 2024-06-01 |
| 205     | 55          | 101            | 22000.00         | active      | 2024-06-01 | NULL       |


<br>

## How do you compute days past due for each loan?
Tests whether the candidate derives it from expected payment schedule vs actual transactions.

**ANSWER:** Payments aren't tied one-to-one to specific installments — a borrower might pay a lump sum that covers multiple months, or underpay and fall behind gradually. So you can't just look at one missed payment. Instead, you line up the scheduled payments in order by due date and build a running total of what should have been paid by each date. Then you compare that running total against the actual total the borrower has paid. The first scheduled payment where the running expected amount exceeds what's been paid is where the borrower fell behind. You take that due date and subtract it from today to get days past due. In production you'd probably materialize this as a daily snapshot table for performance, but the source of truth is always this derived calculation.

```SQL
with

total_paid_by_loan as (

    select
        loan_id,
        sum(paid_amount) as total_paid

    from loan_transactions

    where transaction_type = 'payment'

    group by loan_id

),

running_expected as (

    select
        loan_id,
        due_date,

        sum(expected_amount) over (
            partition by loan_id
            order by due_date
            rows between unbounded preceding and current row
        ) as cumulative_expected

    from loan_scheduled_payments

    where due_date < current_date

),

earliest_unpaid as (

    select
        running_expected.loan_id,
        min(running_expected.due_date) as first_missed_due_date

    from running_expected

    left join total_paid_by_loan payments
        on running_expected.loan_id = payments.loan_id

    where running_expected.cumulative_expected > coalesce(payments.total_paid, 0)

    group by running_expected.loan_id

)

select
    loans.loan_id,
    earliest_unpaid.first_missed_due_date,
    current_date - earliest_unpaid.first_missed_due_date as days_past_due

from earliest_unpaid

inner join loans
    on earliest_unpaid.loan_id = loans.loan_id

where loans.status = 'active'
```

<br>

## How would you partition loan_transactions if volume reached 500M rows?
Tests scale thinking: date partitioning and clustering by loan_id.

**ANSWER**: When you've got 500 million rows in a transactions table, you don't want the database scanning everything every time someone asks a question. So you partition the table by date — basically splitting it into smaller chunks, like one per month, based on when the payment came in. That way if someone queries for last month's data, the database only looks at that one chunk and ignores everything else. Then within each partition, you cluster by loan_id, which just means the rows for the same loan are physically stored next to each other on disk. So now when you ask "show me all payments for loan 101 in March," the database opens the March chunk, finds the loan 101 rows all sitting together, and you're done. You went from scanning half a billion rows to reading maybe a few hundred.

```SQL
-- create the new partitioned version
CREATE TABLE loan_transactions_partitioned (
    transaction_id BIGINT PRIMARY KEY,
    loan_id BIGINT REFERENCES loans(loan_id),
    paid_amount DECIMAL,
    paid_at TIMESTAMP,
    transaction_type VARCHAR
)
PARTITION BY RANGE (paid_at);

-- create the monthly partitions
CREATE TABLE loan_transactions_2024_01
    PARTITION OF loan_transactions_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- ... one partition per month ...

-- migrate the data
INSERT INTO loan_transactions_partitioned
SELECT * FROM loan_transactions;

-- swap the tables
ALTER TABLE loan_transactions RENAME TO loan_transactions_old;
ALTER TABLE loan_transactions_partitioned RENAME TO loan_transactions;
```

Note: This is Postgres syntax, where you can't partition an existing table in place — you have to create a new partitioned table, migrate the data, and swap. In cloud warehouses like Snowflake, BigQuery, or Redshift, partitioning is handled for you automatically — you just declare it at table creation and never think about it again.

<br>

## What changes if a payment applies partially to interest and partially to principal?
<sub>Tests whether transaction_type is fine-grained enough to decompose.</sub>

**ANSWER:** Right now every payment is just logged as one row where the transaction_type field says 'payment.' But when someone makes a loan payment, part of that money goes toward the interest the bank charged them, and part actually pays down what they owe. Those are two different things — the bank cares about tracking them separately. The good news is the schema already handles this without any changes. Instead of logging one row where transaction_type says 'payment' for $1,000, you log two rows — one where transaction_type says 'principal_payment' for $700 and one where it says 'interest_payment' for $300. Now when you want the outstanding balance, you just sum up the rows where transaction_type is 'principal_payment'. When the finance team wants to know how much interest income came in, they sum up the 'interest_payment' rows. Same table, same structure, you're just being more specific with the values in the transaction_type field.

Before (one row):

| transaction_id | loan_id | paid_amount | transaction_type |
|----------------|---------|-------------|------------------|
| 501            | 101     | 1000.00     | payment          |

After (two rows):

| transaction_id | loan_id | paid_amount | transaction_type  |
|----------------|---------|-------------|-------------------|
| 501            | 101     | 700.00      | principal_payment |
| 502            | 101     | 300.00      | interest_payment  |
