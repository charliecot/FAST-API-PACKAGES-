# Python Generators — How They Work, Why They Matter, and Where APIs Use Them

## 1. What is a generator?

A generator is an iterator that produces values one at a time, instead of building and returning the entire sequence at once.

A normal function uses `return` to finish and return a result. A generator function uses `yield` to produce a value and pause its execution.

```python
def get_numbers():
    yield 1
    yield 2
    yield 3

numbers = get_numbers()

print(next(numbers))  # 1
print(next(numbers))  # 2
print(next(numbers))  # 3
```

Calling `get_numbers()` does not immediately run the whole function and return a list. It creates a generator object. Each `next()` resumes execution until the next `yield`.

After the last value, the generator raises `StopIteration`. A `for` loop handles that automatically.

## 2. Function with return vs generator with yield

```python
def normal_function():
    return [1, 2, 3]

def generator_function():
    yield 1
    yield 2
    yield 3
```

The first returns a list. The second returns a generator iterator.

A generator is usually consumed once. After it is exhausted, iterating over it again produces no values. Create a new generator if you need a fresh iteration.

## 3. Why generators matter

### Memory efficiency

```python
# Builds a complete list in memory.
numbers = [number for number in range(10_000_000)]

# Produces values as needed.
numbers = (number for number in range(10_000_000))
```

The generator does not store all ten million generated integers at once. This is useful for large files, database result processing, and data pipelines.

### Lazy evaluation

Values are computed only when requested. This can avoid expensive work when a consumer stops early.

### Streaming and pipelines

One generator can feed another without creating many intermediate lists.

## 4. Generator expressions

A generator expression resembles a list comprehension but uses parentheses:

```python
squares_list = [number * number for number in range(5)]
squares_generator = (number * number for number in range(5))
```

Use a list when you need a reusable collection with indexing and length. Use a generator when you want to process values incrementally.

## 5. Iterables, iterators, and generators

- **Iterable**: an object you can loop over, such as a list, tuple, string, or set.
- **Iterator**: an object that provides `__next__()` and remembers iteration state.
- **Generator**: a convenient kind of iterator created by a generator function or expression.

```python
items = [10, 20, 30]  # iterable
iterator = iter(items)

print(next(iterator))  # 10
print(next(iterator))  # 20
```

A list is iterable but is not itself an iterator. `iter(list)` returns an iterator.

## 6. Generator state

```python
def counter():
    print("Starting")
    yield 1
    print("Resumed")
    yield 2
    print("Finished")

g = counter()
print("Generator created")
print(next(g))
print(next(g))
```

The function body starts when the first `next()` is called, not when the generator object is created. Local variables and the current execution position are preserved between yields.

## 7. yield from

`yield from` delegates iteration to another iterable or generator:

```python
def combined():
    yield "first"
    yield from [1, 2, 3]
    yield "last"

print(list(combined()))
# ['first', 1, 2, 3, 'last']
```

It is useful for composing generators.

## 8. Generator expressions for file processing

Avoid loading a huge file completely if you only need to process it line by line:

```python
def read_nonempty_lines(path):
    with open(path, encoding="utf-8") as file:
        for line in file:
            line = line.strip()
            if line:
                yield line
```

Usage:

```python
for line in read_nonempty_lines("large.log"):
    process(line)
```

The file is read incrementally. Be aware that the file remains open while the generator is being consumed. Exhaust or close the generator appropriately if the consumer may stop early. A `with` block ensures closure when the generator finishes or is closed.

## 9. Generators and FastAPI dependencies

FastAPI commonly uses a dependency with `yield` to provide a resource and clean it up afterwards.

```python
from sqlmodel import Session

def get_session():
    with Session(engine) as session:
        yield session
```

Use it:

```python
from fastapi import Depends

@app.get("/bands/")
def list_bands(session: Session = Depends(get_session)):
    return session.exec(select(Band)).all()
```

Conceptual lifecycle:

1. FastAPI calls the dependency.
2. The dependency creates a database session.
3. `yield session` provides it to the endpoint.
4. The endpoint runs.
5. FastAPI resumes dependency cleanup.
6. The `with` block closes the session.

This is a resource-lifecycle pattern, not merely a way to return multiple values.

### Dependency with explicit try/finally

```python
def get_session():
    session = Session(engine)
    try:
        yield session
    finally:
        session.close()
```

The context-manager form is usually clearer.

## 10. Yield dependencies and exceptions

A dependency can use `try/finally` to guarantee cleanup:

```python
def get_resource():
    resource = open_resource()
    try:
        yield resource
    finally:
        resource.close()
```

If an exception occurs while the endpoint runs, cleanup still happens as control unwinds. Do not swallow exceptions in a dependency; re-raise them after any required cleanup.

## 11. Streaming a response from FastAPI

Generators can produce response chunks incrementally.

```python
from fastapi.responses import StreamingResponse

def generate_text():
    yield "First line\n"
    yield "Second line\n"
    yield "Third line\n"

@app.get("/stream")
def stream():
    return StreamingResponse(
        generate_text(),
        media_type="text/plain",
    )
```

For a CSV export:

```python
import csv
import io
from fastapi.responses import StreamingResponse

def csv_rows():
    buffer = io.StringIO()
    writer = csv.writer(buffer)

    writer.writerow(["id", "name"])
    yield buffer.getvalue()
    buffer.seek(0)
    buffer.truncate(0)

    for band in load_bands_in_batches():
        writer.writerow([band.id, band.name])
        yield buffer.getvalue()
        buffer.seek(0)
        buffer.truncate(0)

@app.get("/bands/export")
def export_bands():
    return StreamingResponse(
        csv_rows(),
        media_type="text/csv",
        headers={"Content-Disposition": 'attachment; filename="bands.csv"'},
    )
```

This is a simplified illustration. For very large datasets, retrieve database rows in batches and avoid loading the entire result into memory. Ensure the database session remains open for the duration of streaming if the generator depends on it.

## 12. Streaming large database results

A conceptual batch-processing pattern:

```python
def export_bands(session: Session):
    offset = 0
    batch_size = 500

    while True:
        statement = (
            select(Band)
            .offset(offset)
            .limit(batch_size)
        )
        batch = session.exec(statement).all()

        if not batch:
            break

        for band in batch:
            yield band

        offset += batch_size
```

Offset pagination can become expensive or inconsistent for large, changing tables. Keyset pagination (for example, `WHERE id > last_id ORDER BY id LIMIT n`) can be more efficient and stable when the ordering key is appropriate.

## 13. Async generators

An async generator uses `async def` and `yield`:

```python
async def event_stream():
    yield "event: message\ndata: hello\n\n"
```

It can be consumed asynchronously with `async for`. This is useful for asynchronous streaming sources, event streams, and server-sent events.

Do not use `await` simply because you see `yield`; they solve different problems:
- `yield`: produce a value and suspend the generator.
- `await`: suspend a coroutine while waiting for an awaitable operation.

## 14. Common generator mistakes

1. Calling `list(generator)` consumes all values and removes the memory advantage.
2. Trying to iterate over an exhausted generator again.
3. Returning a generator where the consumer expects a JSON list without intentionally streaming.
4. Forgetting resource cleanup if the generator is abandoned early.
5. Confusing `yield` in a FastAPI dependency with streaming an HTTP response. They are different uses of generators.
6. Using a synchronous generator for blocking work in a way that harms an async request path; understand the execution model.

## 15. When should you use generators?

Good use cases:
- processing large files;
- log parsing;
- database exports in batches;
- streaming CSV/text responses;
- pipelines and transformations;
- FastAPI dependencies that need setup and teardown;
- asynchronous event streams.

A list is often better for small results that need indexing, repeated iteration, or a JSON array response.

## 16. Practice

1. Write a generator that yields even numbers up to a limit.
2. Write a generator that reads a large text file line by line.
3. Build a FastAPI dependency that opens and closes a resource.
4. Create a streaming text endpoint.
5. Compare memory use between a list comprehension and a generator expression.
6. Write a generator pipeline that filters and transforms records.

## 17. Official references

- Python generator expressions: https://docs.python.org/3/reference/expressions.html#generator-expressions
- Python yield: https://docs.python.org/3/reference/simple_stmts.html#yield
- FastAPI dependencies with yield: https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/
- FastAPI streaming responses: https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse
