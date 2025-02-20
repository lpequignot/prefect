---
description: Learn how to control concurrency and apply rate limits using Prefect's provided utilities.
tags:
    - concurrency
    - rate limits
search:
  boost: 2
---

# Global concurrency limits and rate limits

Global concurrency limits allow you to manage task execution efficiently, controlling how many tasks can run simultaneously. They are ideal when optimizing resource usage, preventing bottlenecks, and customizing task execution are priorities.

Rate Limits ensure system stability by governing the frequency of requests or operations. They are suitable for preventing overuse, ensuring fairness, and handling errors gracefully.

When selecting between Concurrency and Rate Limits, consider your primary goal. Choose Concurrency Limits for resource optimization and task management. Choose Rate Limits to maintain system stability and fair access to services.

The core difference between a rate limit and a concurrency limit is the way in which slots are released. With a rate limit, slots are released at a controlled rate, controlled by `slot_decay_per_second` whereas with a concurrency limit, slots are released when the concurrency manager is exited. 

## Managing Global concurrency limits and rate limits

You can create, read, edit, and delete concurrency limits via the Prefect UI. 

When creating a concurrency limit, you can specify the following parameters:

- **Name**: The name of the concurrency limit. This name is also how you'll reference the concurrency limit in your code. Special characters, such as `/`, `%`, `&`, `>`, `<`, are not allowed.
- **Concurrency Limit**: The maximum number of slots that can be occupied on this concurrency limit.
- **Slot Decay Per Second**: Controls the rate at which slots are released when the concurrency limit is used as a rate limit. This value must be configured when using the `rate_limit` function.
- **Active**: Whether or not the concurrency limit is in an active state.

### Active vs inactive limits

Global concurrency limits can be in either an `active` or `inactive` state.

- **Active**: In this state, slots can be occupied, and code execution will be blocked when slots are unable to be acquired.
- **Inactive**: In this state, slots will not be occupied, and code execution will not be blocked. Concurrency enforcement occurs only when you activate the limit.

### Slot decay

Global concurrency limits can be configured with slot decay. This is used when the concurrency limit is used as a rate limit, and it governs the pace at which slots are released or become available for reuse after being occupied. These slots effectively represent the concurrency capacity within a specific concurrency limit. The concept is best understood as the rate at which these slots "decay" or refresh.

To configure slot decay, you can set the `slot_decay_per_second` parameter when defining or adjusting a concurrency limit.

For practical use, consider the following:

- *Higher values*: Setting `slot_decay_per_second` to a higher value, such as 5.0, results in slots becoming available relatively quickly. In this scenario, a slot that was occupied by a task will free up after just `0.2` (`1.0 / 5.0`) seconds.

- *Lower values*: Conversely, setting `slot_decay_per_second` to a lower value, like 0.1, causes slots to become available more slowly. In this scenario it would take `10` (`1.0 / 0.1`) seconds for a slot to become available again after occupancy

Slot decay provides fine-grained control over the availability of slots, enabling you to optimize the rate of your workflow based on your specific requirements.

## Using the `concurrency` context manager
The `concurrency `context manager allows control over the maximum number of concurrent operations. You can select either the synchronous (`sync`) or asynchronous (`async`) version, depending on your use case. Here's how to use it:

!!! tip "Concurrency limits are implicitly created"
    When using the `concurrency` context manager, the concurrency limit you use will be created, in an inactive state, if it does not already exist.

**Sync**

```python
from prefect import flow, task
from prefect.concurrency.sync import concurrency


@task
def process_data(x, y):
    with concurrency("database", occupy=1):
        return x + y


@flow
def my_flow():
    for x, y in [(1, 2), (2, 3), (3, 4), (4, 5)]:
        process_data.submit(x, y)


if __name__ == "__main__":
    my_flow()
```


**Async**

```python
import asyncio
from prefect import flow, task
from prefect.concurrency.asyncio import concurrency


@task
async def process_data(x, y):
    async with concurrency("database", occupy=1):
        return x + y


@flow
async def my_flow():
    for x, y in [(1, 2), (2, 3), (3, 4), (4, 5)]:
        await process_data.submit(x, y)


if __name__ == "__main__":
    asyncio.run(my_flow())
```


1. The code imports the necessary modules and the concurrency context manager. Use the `prefect.concurrency.sync` module for sync usage and the `prefect.concurrency.asyncio` module for async usage.
2. It defines a `process_data` task, taking `x` and `y` as input arguments. Inside this task, the concurrency context manager controls concurrency, using the `database` concurrency limit and occupying one slot. If another task attempts to run with the same limit and no slots are available, that task will be blocked until a slot becomes available.
3. A flow named `my_flow` is defined. Within this flow, it iterates through a list of tuples, each containing pairs of x and y values. For each pair, the `process_data` task is submitted with the corresponding x and y values for processing.


## Using `rate_limit`
The Rate Limit feature provides control over the frequency of requests or operations, ensuring responsible usage and system stability. Depending on your requirements, you can utilize `rate_limit` to govern both synchronous (sync) and asynchronous (async) operations. Here's how to make the most of it:

!!! tip "Slot decay"
    When using the `rate_limit` function the concurrency limit you use must have a slot decay configured.

**Sync**

```python
from prefect import flow, task
from prefect.concurrency.sync import rate_limit


@task
def make_http_request():
    rate_limit("rate-limited-api")
    print("Making an HTTP request...")


@flow
def my_flow():
    for _ in range(10):
        make_http_request.submit()


if __name__ == "__main__":
    my_flow()
```


**Async**

```python
import asyncio

from prefect import flow, task
from prefect.concurrency.asyncio import rate_limit


@task
async def make_http_request():
    await rate_limit("rate-limited-api")
    print("Making an HTTP request...")


@flow
async def my_flow():
    for _ in range(10):
        await make_http_request.submit()


if __name__ == "__main__":
    asyncio.run(my_flow())
```

1. The code imports the necessary modules and the `rate_limit` function. Use the `prefect.concurrency.sync` module for sync usage and the `prefect.concurrency.asyncio` module for async usage.
2. It defines a `make_http_request` task. Inside this task, the `rate_limit` function is used to ensure that the requests are made at a controlled pace.
3. A flow named `my_flow` is defined. Within this flow the `make_http_request` task is submitted 10 times.

## Using `concurrency` and `rate_limit` outside of a flow

`concurreny` and `rate_limit` can be used outside of a flow to control concurrency and rate limits for any operation.

```python
import asyncio

from prefect.concurrency.asyncio import rate_limit


async def main():
    for _ in range(10):
        await rate_limit("rate-limited-api")
        print("Making an HTTP request...")



if __name__ == "__main__":
    asyncio.run(main())
```

## Use cases

### Throttling task submission

Throttling task submission to avoid overloading resources, to comply with external rate limits, or ensure a steady, controlled flow of work.

In this scenario the `rate_limit` function is used to throttle the submission of tasks. The rate limit acts as a bottleneck, ensuring that tasks are submitted at a controlled rate, governed by the `slot_decay_per_second` setting on the associated concurrency limit.

```python
from prefect import flow, task
from prefect.concurrency.sync import rate_limit


@task
def my_task(i):
    return i


@flow
def my_flow():
    for _ in range(100):
        rate_limit("slow-my-flow", occupy=1)
        my_task.submit(1)


if __name__ == "__main__":
    my_flow()
```

### Managing database connections

Managing the maximum number of concurrent database connections to avoid exhausting database resources.

In this scenario we've setup a concurrency limit named `database` and given it a maximum concurrency limit that matches the maximum number of database connections we want to allow. We then use the `concurrency` context manager to control the number of database connections allowed at any one time.

```python
from prefect import flow, task, concurrency
import psycopg2

@task
def database_query(query):
    # Here we request a single slot on the 'database' concurrency limit. This
    # will block in the case that all of the database connections are in use
    # ensuring that we never exceed the maximum number of database connections.
    with concurrency("database", occupy=1):
        connection = psycopg2.connect("<connection_string>")
        cursor = connection.cursor()
        cursor.execute(query)
        result = cursor.fetchall()
        connection.close()
        return result

@flow
def my_flow():
    queries = ["SELECT * FROM table1", "SELECT * FROM table2", "SELECT * FROM table3"]

    for query in queries:
        database_query.submit(query)

if __name__ == "__main__":
    my_flow()
```

### Parallel data processing

Limiting the maximum number of parallel processing tasks.

In this scenario we want to limit the number of `process_data` tasks to five at any one time. We do this by using the `concurrency` context manager to request five slots on the `data-processing` concurrency limit. This will block until five slots are free and then submit five more tasks, ensuring that we never exceed the maximum number of parallel processing tasks. 

```python
import asyncio
from prefect.concurrency.sync import concurrency


async def process_data(data):
    print(f"Processing: {data}")
    await asyncio.sleep(1)
    return f"Processed: {data}"


async def main():
    data_items = list(range(100))
    processed_data = []

    while data_items:
        with concurrency("data-processing", occupy=5):
            chunk = [data_items.pop() for _ in range(5)]
            processed_data += await asyncio.gather(
                *[process_data(item) for item in chunk]
            )

    print(processed_data)


if __name__ == "__main__":
    asyncio.run(main())
```
