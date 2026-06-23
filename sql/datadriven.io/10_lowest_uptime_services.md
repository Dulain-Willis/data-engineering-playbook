# Question

The FinOps team is investigating the top spending tiers and needs the 3rd highest cost amount on record. Multiple entries can share the same amount, so return the specific value that ranks third.

## Schema

```text 
svc_health
    check_id INTEGER
    svc_name TEXT
    status TEXT
    latency REAL
    uptime REAL
    checked TEXT
    region TEXT 
```

<br>

--- 

# Notes 

## Why this problem exists in real interviews
This probes two distinct skills: whether you understand that aggregation must precede ranking (a common source of incorrect results), and whether you know the semantic differences between ROW_NUMBER, RANK, and DENSE_RANK. These are foundational window function concepts that appear in nearly every SQL round at L5+.

<br> 

## Trick to Solving
The phrase "if multiple are tied, include all of them" is the signal. Whenever a top-N question requires tie inclusion, the answer is DENSE_RANK. Spot it by looking for language about ties, "at least N", or "include all at position N".

- Recognize the tie-inclusion language in the prompt
- Use DENSE_RANK() instead of ROW_NUMBER() or LIMIT
- Aggregate to the correct grain before ranking

<br>

## Break down the requirements
1. Aggregate to one row per service
`GROUP BY` svc_name with `MIN(uptime)` collapses the 63M health check rows into 200 rows (one per service). This is the correct grain for ranking.

2. Rank with DENSE_RANK
`DENSE_RANK() OVER (ORDER BY MIN(uptime) ASC)` assigns the same rank to tied values and never skips numbers, so filtering to rank ≤ 10 includes all ties at position 10.

3. Filter and order 
Wrap in a subquery, filter `WHERE rnk ≤ 10`, and `ORDER BY min_uptime ASC` to surface the worst services first.

<br>

## Interviewers Watch For
The most common failure is using `LIMIT 10`, which silently drops tied rows and produces non-deterministic output. Strong candidates immediately flag "include ties" as a `DENSE_RANK` signal without being prompted.

<br>

## Common Pitfall
Ranking before aggregating inverts the logic: you would rank individual health checks instead of services. Always aggregate to the output grain before applying window functions.

<br>

--- 

# Solution 

```SQL
with
  
svc_name_ranked_by_min_uptime as (

  select
    svc_name,
    min(uptime) as minimum_uptime,
  
    dense_rank() over (
      order by min(uptime) asc
    ) as uptime_rank

  from svc_health
  
  group by svc_name
  
)
  
select
  svc_name,
  minimum_uptime

from svc_name_ranked_by_min_uptime
  
where uptime_rank <= 10
  
order by minimum_uptime asc
```


<br>

### Cost Analysis
The `GROUP BY` reduces 63M rows to 200 before the window function runs. The ranking step is trivially cheap. The bottleneck is the full-table scan for the aggregate. A partial index on (`svc_name`, `uptime`) or a materialized view would help at production scale.

<br> 

--- 

# Common Follow-Up Questions 

<br>

## What if you needed the bottom 10 by average uptime instead of minimum?
> Tests whether you can swap the aggregate without changing the ranking logic.

<br>

Just swap `MIN(uptime)` for `AVG(uptime)` — everything else stays exactly the same. The CTE aggregates to one row per service, DENSE_RANK ranks them, and the filter cuts at 10. You're only changing which number represents the service.

```sql
with

svc_name_ranked_by_avg_uptime as (

  select
    svc_name,
    avg(uptime) as average_uptime,

    dense_rank() over (
      order by avg(uptime) asc
    ) as uptime_rank

  from svc_health

  group by svc_name

)
```

<br>

## How would the query change if the table had 50,000 distinct services instead of 200?
> At scale, the subquery output grows and the window sort becomes non-trivial.

<br>

The query structure doesn't change at all — but the performance profile does. With 200 services the ranked CTE is tiny and the sort is instant. With 50,000 services, the window sort over 50,000 rows is still fast in absolute terms, but now worth thinking about. The real bottleneck stays the same: the full-table scan to aggregate all the raw health check rows. A composite index on `(svc_name, uptime)` lets the engine compute `MIN(uptime)` per service without reading every column, which is where you'd focus first.

<br>

## What if you needed to break ties by service name alphabetically?
> Tests compound ORDER BY inside DENSE_RANK: ORDER BY MIN(uptime) ASC, svc_name ASC.

<br>

Add `svc_name ASC` as a second sort key inside the window function. Now two services with the same minimum uptime get different ranks — the one that comes first alphabetically gets the lower rank. This is just a tiebreaker, so DENSE_RANK still won't skip numbers.

```sql
with

svc_name_ranked_by_min_uptime as (

  select
    svc_name,
    min(uptime) as minimum_uptime,

    dense_rank() over (
      order by min(uptime) asc, svc_name asc
    ) as uptime_rank

  from svc_health

  group by svc_name

)
```

<br>

## Could you solve this without a window function?
> A self-join or correlated subquery approach is valid but less readable and often slower.

<br>

Yes — a correlated subquery works. For each service, count how many other services have a strictly lower minimum uptime. If that count is less than 10, the service is in the bottom 10. It gets the right answer, but it's harder to read and the database has to re-execute the inner query for every row in the outer one. DENSE_RANK does one ordered pass; this approach does many.

```sql
select
  svc_name,
  min(uptime) as minimum_uptime

from svc_health

group by svc_name

having (

  select count(distinct min_up.min_uptime)

  from (

    select 
        svc_name, 
        min(uptime) as min_uptime

    from svc_health

    group by svc_name

  ) as min_up

  where min_up.min_uptime < min(svc_health.uptime)

) < 10

order by minimum_uptime asc

```

<br>

The self-join version expresses the same logic differently — join each service against all services with a strictly lower min uptime, then use `HAVING` to keep only those with fewer than 10 such services below them.

```sql
with

svc_min_uptimes as (

  select
    svc_name,
    min(uptime) as min_uptime

  from svc_health

  group by svc_name

)

select
  s.svc_name,
  s.min_uptime as minimum_uptime

from svc_min_uptimes s

left join svc_min_uptimes o 
    on o.min_uptime < s.min_uptime

group by s.svc_name, s.min_uptime

having count(distinct o.min_uptime) < 10

order by s.min_uptime asc
```

The `LEFT JOIN` is important here — services with the absolute lowest uptime have no rows to join against, so without it they'd be dropped entirely. `count(distinct o.min_uptime)` mirrors DENSE_RANK tie behavior the same way the correlated subquery does.


