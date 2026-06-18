# Question

The FinOps team is investigating the top spending tiers and needs the 3rd highest cost amount on record. Multiple entries can share the same amount, so return the specific value that ranks third.

## Schema

```text 
cloud_costs 
    cost_id INT 
    provider TEXT 
    svc_name TEXT 
    region TEXT 
    amount REAL 
    acct_id TEXT 
    bill_date TEXT 
```

<br>

--- 

# Notes 

## Why this problem exists in real interviews
Top-N selection with ties is the classic ranking trap. The interviewer is checking that you pick a window function whose tie behavior matches the prompt and that you understand why LIMIT 1 OFFSET 2 is wrong here: when two line items share the highest amount, the third row in a sorted list is still rank 2.

<br> 

## Trick to Solving
Three ranking functions, three different answers when there are ties on amount:

- ROW_NUMBER: every row gets a unique number; OFFSET-style logic.
- RANK: ties share the rank, then jumps (1, 1, 3).
- DENSE_RANK: ties share the rank, no jump (1, 1, 2).

The prompt says "the specific value that ranks third," which is DENSE_RANK semantics: skip-no-ranks counting of distinct values.

<br>

## Break down the requirements
1. Rank rows by amount with DENSE_RANK
Inside a CTE, attach `DENSE_RANK() OVER (ORDER BY amount DESC)` to every row. Tied amounts share a rank, and the rank counter only advances when the next distinct value appears.

2. Pick rank 3 and collapse to one row
Filter the ranked output to rnk = 3 and project amount. Multiple rows can satisfy rnk = 3 if several line items share the third-highest amount. They all carry the same value, so LIMIT 1 collapses them to the single answer.

<br>

## Interviewers Watch For
Strong candidates name the tie semantics out loud ("third distinct amount, not third row") before writing SQL. They also reach for DENSE_RANK rather than LIMIT 1 OFFSET 2, which silently returns whichever row happens to land in the third slot of a sort order, not the third distinct value.

<br>

## Common Pitfall
Using SELECT DISTINCT amount ... LIMIT 1 OFFSET 2. It works when there are at least 3 distinct amounts, but the moment ties matter the prompt and a tied row appear, the interviewer wants to see you defend the tie behavior. DENSE_RANK encodes that behavior in the query.

<br>

--- 
# Solution 

```SQL 
with

ranked_cost_amount as (

    select
      cost_id,
      amount,
      dense_rank() over (
        order by amount desc
      ) as cost_rank

    from cloud_costs

)
    
select amount

from ranked_cost_amount

where cost_rank = 3

limit 1
```


## Time and Space Complexity
**Time:** O(n log n) for the global sort over 5M cloud_costs rows. The window function is one ordered pass.

**Space:** O(n) for the ranked materialization, though planners that pipeline the window cut that down.

<br> 

--- 

# Common Follow-Up Questions 


