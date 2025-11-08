# FuturesUnordered: Mastering Concurrent Async Operations in Rust

## Introduction

When building async applications in Rust, you often need to execute multiple asynchronous operations concurrently. While you could await each operation sequentially, this approach can be slow and inefficient. The `FuturesUnordered` type from the `futures` crate provides an elegant solution for managing and executing multiple futures concurrently, processing results as they complete.

This article explores how `FuturesUnordered` enables efficient concurrent async operations, with practical examples inspired by real-world CLI tools that need to query multiple APIs, process bulk operations, and handle parallel I/O efficiently.

---

## What is FuturesUnordered?

### The Type

`FuturesUnordered` is a collection that holds multiple futures and allows them to execute concurrently. It implements the `Stream` trait, which means you can poll it to get results as futures complete, rather than waiting for all of them to finish.

### Key Characteristics

1. **Concurrent Execution**: All futures in the collection execute concurrently
2. **Stream-Based**: Results are yielded as they become available
3. **Dynamic**: You can add futures to the collection at runtime
4. **Efficient**: Uses a heap-based priority queue internally for efficient polling

### Basic Concept

```rust
use futures::stream::{FuturesUnordered, StreamExt};

// Create a collection of concurrent futures
let mut tasks = FuturesUnordered::new();

// Add multiple futures
tasks.push(async_task_1());
tasks.push(async_task_2());
tasks.push(async_task_3());

// Process results as they complete
while let Some(result) = tasks.next().await {
    // Handle each result as it arrives
    println!("Task completed: {:?}", result);
}
```

---

## Sequential vs Concurrent Execution

### The Problem with Sequential Execution

Consider a scenario where you need to fetch data from multiple API endpoints:

```rust
// Sequential approach - SLOW ❌
async fn fetch_sequential() -> Vec<String> {
    let mut results = Vec::new();

    results.push(api_client.fetch_user("alice").await?);
    results.push(api_client.fetch_user("bob").await?);
    results.push(api_client.fetch_user("charlie").await?);
    results.push(api_client.fetch_user("dave").await?);

    results
}
```

If each API call takes 100ms, this approach takes **400ms total**.

### The Concurrent Solution

Using `FuturesUnordered`, all requests execute in parallel:

```rust
// Concurrent approach - FAST ✅
use futures::stream::{FuturesUnordered, StreamExt};

async fn fetch_concurrent() -> Vec<String> {
    let mut tasks = FuturesUnordered::new();

    // Start all requests concurrently
    tasks.push(api_client.fetch_user("alice"));
    tasks.push(api_client.fetch_user("bob"));
    tasks.push(api_client.fetch_user("charlie"));
    tasks.push(api_client.fetch_user("dave"));

    let mut results = Vec::new();

    // Collect results as they complete
    while let Some(result) = tasks.next().await {
        results.push(result?);
    }

    results
}
```

With concurrent execution, all four requests run simultaneously, taking approximately **100ms total** (the time of the slowest request).

### Performance Comparison

| Approach | Time (4 requests @ 100ms each) |
|----------|-------------------------------|
| Sequential | 400ms |
| Concurrent | ~100ms |
| **Speedup** | **4x faster** |

---

## Real-World Use Cases

### 1. Parallel API Queries

When building applications that interact with REST APIs, you often need to query multiple endpoints simultaneously. For example, fetching user profiles and user preferences for multiple users:

```rust
use futures::stream::{FuturesUnordered, StreamExt};

struct ApiClient {
    api_base: String,
}

impl ApiClient {
    async fn fetch_user_profile(&self, user_id: &str) -> Result<UserProfile> {
        // Simulated API call
        tokio::time::sleep(Duration::from_millis(100)).await;
        Ok(UserProfile { id: user_id.to_string() })
    }

    async fn fetch_user_preferences(&self, user_id: &str) -> Result<UserPreferences> {
        // Simulated API call
        tokio::time::sleep(Duration::from_millis(150)).await;
        Ok(UserPreferences { user_id: user_id.to_string() })
    }

    async fn fetch_all_user_data(&self, user_ids: Vec<String>) -> Vec<CompleteUserData> {
        let mut profile_queries = FuturesUnordered::new();
        let mut preference_queries = FuturesUnordered::new();

        // Start all queries concurrently
        for user_id in &user_ids {
            profile_queries.push(self.fetch_user_profile(user_id));
            preference_queries.push(self.fetch_user_preferences(user_id));
        }

        let mut profiles = Vec::new();
        let mut preferences = Vec::new();

        // Process profile results as they complete
        while let Some(result) = profile_queries.next().await {
            match result {
                Ok(profile) => profiles.push(profile),
                Err(e) => eprintln!("Error fetching user profile: {}", e),
            }
        }

        // Process preference results as they complete
        while let Some(result) = preference_queries.next().await {
            match result {
                Ok(prefs) => preferences.push(prefs),
                Err(e) => eprintln!("Error fetching user preferences: {}", e),
            }
        }

        // Combine results (simplified)
        profiles.into_iter()
            .zip(preferences.into_iter())
            .map(|(profile, prefs)| CompleteUserData { profile, preferences: prefs })
            .collect()
    }
}
```

**Benefits:**
- All API calls execute in parallel
- Results are processed as they arrive
- Total time is determined by the slowest request, not the sum

### 2. Concurrent DNS Resolution

When resolving multiple hostnames, you can resolve them all concurrently:

```rust
use futures::stream::{FuturesUnordered, StreamExt};

struct DnsResolver;

impl DnsResolver {
    async fn resolve_hostname(&self, hostname: &str) -> Option<String> {
        // Simulated DNS lookup
        tokio::time::sleep(Duration::from_millis(50)).await;
        Some(format!("{}.resolved.example.com", hostname))
    }

    async fn resolve_multiple(&self, hostnames: Vec<String>) -> Vec<(String, Option<String>)> {
        let mut resolver_tasks = FuturesUnordered::new();

        // Start all resolutions concurrently
        for hostname in hostnames.clone() {
            resolver_tasks.push(async move {
                let resolver = DnsResolver;
                (hostname.clone(), resolver.resolve_hostname(&hostname).await)
            });
        }

        let mut results = Vec::new();

        // Collect results as they complete
        while let Some(result) = resolver_tasks.next().await {
            results.push(result);
        }

        results
    }
}
```

**Benefits:**
- All DNS lookups happen simultaneously
- No waiting for one lookup to complete before starting the next
- Significantly faster for multiple resolutions

### 3. Bulk Operations

For bulk operations like processing multiple items, `FuturesUnordered` enables concurrent processing:

```rust
use futures::stream::{FuturesUnordered, StreamExt};

struct OperationResult {
    item_id: String,
    success: bool,
    message: String,
}

async fn process_bulk_operations(
    items: Vec<String>,
    operation: impl Fn(String) -> Pin<Box<dyn Future<Output = Result<String>> + Send>>,
) -> Vec<OperationResult> {
    let mut operations = FuturesUnordered::new();

    // Start all operations concurrently
    for item in items {
        operations.push(async move {
            match operation(item.clone()).await {
                Ok(msg) => OperationResult {
                    item_id: item,
                    success: true,
                    message: msg,
                },
                Err(e) => OperationResult {
                    item_id: item,
                    success: false,
                    message: e.to_string(),
                },
            }
        });
    }

    let mut results = Vec::new();

    // Process results as they complete
    while let Some(result) = operations.next().await {
        results.push(result);
    }

    results
}

// Usage example
async fn bulk_disable_instances(instance_ids: Vec<String>) -> Vec<OperationResult> {
    process_bulk_operations(instance_ids, |id| {
        Box::pin(async move {
            // Simulated disable operation
            tokio::time::sleep(Duration::from_millis(200)).await;
            Ok(format!("Instance {} disabled", id))
        })
    })
    .await
}
```

**Benefits:**
- All operations run concurrently
- Results are available as soon as each operation completes
- Failed operations don't block successful ones

### 4. Parallel Configuration Updates

When applying configuration changes to multiple targets:

```rust
use futures::stream::{FuturesUnordered, StreamExt};

async fn update_replication_delay(
    instances: Vec<String>,
    delay_seconds: u64,
) -> Vec<(String, Result<(), String>)> {
    let mut delay_tasks = FuturesUnordered::new();

    // Start all delay updates concurrently
    for instance in instances.clone() {
        delay_tasks.push(async move {
            let result = apply_delay(&instance, delay_seconds).await;
            (instance, result)
        });
    }

    let mut results = Vec::new();

    // Collect results as they complete
    while let Some(result) = delay_tasks.next().await {
        results.push(result);
    }

    results
}

async fn apply_delay(instance: &str, seconds: u64) -> Result<(), String> {
    // Simulated API call to apply delay
    tokio::time::sleep(Duration::from_millis(100)).await;
    println!("Applied {}s delay to {}", seconds, instance);
    Ok(())
}
```

---

## Performance Considerations

### When to Use FuturesUnordered

✅ **Use FuturesUnordered when:**
- You have multiple independent async operations
- Operations can run concurrently without blocking each other
- You want to process results as they complete (not wait for all)
- You need to add futures dynamically at runtime
- Operations are I/O-bound (network, disk, etc.)

❌ **Consider alternatives when:**
- You need a fixed number of concurrent operations (use `join!` or `try_join!`)
- You need to limit concurrency (use a semaphore or `FuturesUnordered` with a limit)
- All operations must complete before processing results (use `join_all` or `try_join_all`)
- Operations are CPU-bound (consider using a thread pool)

### Concurrency Limits

Sometimes you want to limit how many operations run concurrently. You can combine `FuturesUnordered` with a semaphore:

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use tokio::sync::Semaphore;

async fn process_with_limit(items: Vec<String>, max_concurrent: usize) -> Vec<String> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let mut tasks = FuturesUnordered::new();

    for item in items {
        let sem = semaphore.clone();
        tasks.push(async move {
            // Acquire permit (limits concurrency)
            let _permit = sem.acquire().await.unwrap();

            // Perform the operation
            process_item(&item).await

            // Permit is released when dropped
        });
    }

    let mut results = Vec::new();
    while let Some(result) = tasks.next().await {
        results.push(result);
    }

    results
}
```

### Memory Considerations

`FuturesUnordered` stores all futures in memory. For very large numbers of futures:
- Consider batching operations
- Use concurrency limits to prevent memory exhaustion
- Process results incrementally rather than collecting all at once

---

## Complete Rust Implementation Example

Here's a complete, educational example demonstrating `FuturesUnordered` in a practical scenario:

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use std::time::{Duration, Instant};
use tokio::time::sleep;

// Simulated API client
struct ApiClient {
    base_url: String,
}

impl ApiClient {
    async fn fetch_resource(&self, resource_id: &str) -> Result<Resource, String> {
        // Simulate network delay
        sleep(Duration::from_millis(100)).await;

        // Simulate occasional failures
        if resource_id == "resource-3" {
            return Err("Resource not found".to_string());
        }

        Ok(Resource {
            id: resource_id.to_string(),
            data: format!("Data for {}", resource_id),
        })
    }

    async fn fetch_multiple_resources(
        &self,
        resource_ids: Vec<String>,
    ) -> Vec<(String, Result<Resource, String>)> {
        let mut fetch_tasks = FuturesUnordered::new();

        // Start all fetches concurrently
        for resource_id in resource_ids.clone() {
            let client = self;
            fetch_tasks.push(async move {
                let result = client.fetch_resource(&resource_id).await;
                (resource_id, result)
            });
        }

        let mut results = Vec::new();

        // Process results as they complete
        while let Some(result) = fetch_tasks.next().await {
            results.push(result);
        }

        results
    }
}

#[derive(Debug)]
struct Resource {
    id: String,
    data: String,
}

#[tokio::main]
async fn main() {
    let client = ApiClient {
        base_url: "https://api.example.com".to_string(),
    };

    let resource_ids = vec![
        "resource-1".to_string(),
        "resource-2".to_string(),
        "resource-3".to_string(),
        "resource-4".to_string(),
        "resource-5".to_string(),
    ];

    println!("Fetching {} resources concurrently...", resource_ids.len());
    let start = Instant::now();

    let results = client.fetch_multiple_resources(resource_ids).await;

    let elapsed = start.elapsed();
    println!("Completed in {:?}", elapsed);

    // Process results
    let mut success_count = 0;
    let mut failure_count = 0;

    for (id, result) in results {
        match result {
            Ok(resource) => {
                println!("✓ {}: {}", id, resource.data);
                success_count += 1;
            }
            Err(e) => {
                println!("✗ {}: Error - {}", id, e);
                failure_count += 1;
            }
        }
    }

    println!("\nSummary: {} succeeded, {} failed", success_count, failure_count);
}
```

### Expected Output

```
Fetching 5 resources concurrently...
✓ resource-1: Data for resource-1
✓ resource-2: Data for resource-2
✗ resource-3: Error - Resource not found
✓ resource-4: Data for resource-4
✓ resource-5: Data for resource-5
Completed in 100.123456ms

Summary: 4 succeeded, 1 failed
```

Notice that all requests completed in approximately 100ms (the time of a single request), not 500ms (5 × 100ms).

---

## Advanced Patterns

### Error Handling in Concurrent Operations

When dealing with concurrent operations, you often want to collect both successes and failures:

```rust
use futures::stream::{FuturesUnordered, StreamExt};

struct OperationStats {
    successful: Vec<String>,
    failed: Vec<(String, String)>,
}

async fn execute_with_error_tracking(
    operations: Vec<impl Fn() -> Pin<Box<dyn Future<Output = Result<String, String>> + Send>>>,
) -> OperationStats {
    let mut tasks = FuturesUnordered::new();

    for (index, operation) in operations.into_iter().enumerate() {
        tasks.push(async move {
            let result = operation().await;
            (index, result)
        });
    }

    let mut stats = OperationStats {
        successful: Vec::new(),
        failed: Vec::new(),
    };

    while let Some((index, result)) = tasks.next().await {
        match result {
            Ok(msg) => stats.successful.push(format!("Operation {}: {}", index, msg)),
            Err(e) => stats.failed.push((index.to_string(), e)),
        }
    }

    stats
}
```

### Early Termination

You can implement early termination by breaking out of the loop when a condition is met:

```rust
use futures::stream::{FuturesUnordered, StreamExt};

async fn find_first_successful(
    operations: Vec<impl Fn() -> Pin<Box<dyn Future<Output = Result<String>> + Send>>>,
) -> Option<String> {
    let mut tasks = FuturesUnordered::new();

    for operation in operations {
        tasks.push(operation());
    }

    while let Some(result) = tasks.next().await {
        if let Ok(value) = result {
            // Cancel remaining tasks (they'll complete but we ignore results)
            return Some(value);
        }
    }

    None
}
```

### Combining Multiple Streams

You can process multiple `FuturesUnordered` streams simultaneously:

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use futures::stream;

async fn process_multiple_streams() {
    let mut stream1 = FuturesUnordered::new();
    let mut stream2 = FuturesUnordered::new();

    // Add tasks to both streams
    stream1.push(async_task_1());
    stream1.push(async_task_2());
    stream2.push(async_task_3());
    stream2.push(async_task_4());

    // Process both streams concurrently
    loop {
        tokio::select! {
            result1 = stream1.next() => {
                if let Some(r) = result1 {
                    println!("Stream 1: {:?}", r);
                } else {
                    break;
                }
            }
            result2 = stream2.next() => {
                if let Some(r) = result2 {
                    println!("Stream 2: {:?}", r);
                } else {
                    break;
                }
            }
        }
    }
}
```

---

## Best Practices

### 1. **Process Results Incrementally**

Don't wait for all futures to complete if you can process results as they arrive:

```rust
// Good: Process as results arrive
while let Some(result) = tasks.next().await {
    handle_result(result).await;
}

// Less ideal: Wait for all, then process
let all_results: Vec<_> = tasks.collect().await;
for result in all_results {
    handle_result(result).await;
}
```

### 2. **Handle Errors Gracefully**

Always handle errors in concurrent operations:

```rust
while let Some(result) = tasks.next().await {
    match result {
        Ok(value) => process_success(value),
        Err(e) => {
            eprintln!("Operation failed: {}", e);
            // Decide: continue, retry, or abort
        }
    }
}
```

### 3. **Use Appropriate Concurrency Limits**

For operations that might overwhelm resources, use semaphores or other limiting mechanisms:

```rust
let semaphore = Arc::new(Semaphore::new(10)); // Max 10 concurrent
// ... use semaphore to limit concurrency
```

### 4. **Consider Using `join_all` for Fixed Sets**

If you have a fixed, known set of futures and want to wait for all:

```rust
use futures::future::join_all;

// All futures are known upfront
let results = join_all(vec![
    task1(),
    task2(),
    task3(),
]).await;
```

### 5. **Profile Your Code**

Measure the performance improvement:

```rust
let start = Instant::now();
// ... concurrent operations
let elapsed = start.elapsed();
println!("Completed in {:?}", elapsed);
```

---

## Comparison with Alternatives

### FuturesUnordered vs join!/try_join!

| Feature | FuturesUnordered | join!/try_join! |
|---------|------------------|-----------------|
| Number of futures | Dynamic | Fixed at compile time |
| Process as complete | Yes | No (wait for all) |
| Add at runtime | Yes | No |
| Use case | Dynamic collections | Fixed sets |

### FuturesUnordered vs join_all/try_join_all

| Feature | FuturesUnordered | join_all/try_join_all |
|---------|------------------|----------------------|
| Process as complete | Yes | No (wait for all) |
| Memory efficiency | Better (stream-based) | Collects all results |
| Use case | Stream processing | Batch processing |

---

## Conclusion

`FuturesUnordered` is a powerful tool for concurrent async operations in Rust. Its key advantages include:

- **Concurrent Execution**: All futures run simultaneously
- **Stream-Based Processing**: Results available as they complete
- **Dynamic Collections**: Add futures at runtime
- **Efficient**: Optimized for concurrent polling

Whether you're building CLI tools that query multiple APIs, processing bulk operations, or handling parallel I/O, `FuturesUnordered` provides an elegant and efficient solution for concurrent async programming in Rust.

### Key Takeaways

1. Use `FuturesUnordered` when you need dynamic, concurrent execution
2. Process results incrementally for better responsiveness
3. Always handle errors gracefully in concurrent contexts
4. Consider concurrency limits for resource-intensive operations
5. Measure performance to validate improvements

### Further Reading

- [futures crate documentation](https://docs.rs/futures/)
- [FuturesUnordered API reference](https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html)
- [Tokio async runtime](https://tokio.rs/)
- [Rust async book](https://rust-lang.github.io/async-book/)

---
