# Retry and Polling Patterns in Async Rust

## Introduction

In async applications, you often need to wait for external conditions to be met before proceeding. Whether you're waiting for a service to become available, a database migration to complete, or DNS records to propagate, you need a reliable way to poll for conditions and handle timeouts gracefully.

This article explores retry and polling patterns in async Rust, demonstrating how to build generic, reusable polling mechanisms that can wait for arbitrary conditions with configurable timeouts and intervals. We'll examine practical patterns inspired by real-world CLI tools that need to wait for external state changes.

---

## The Problem: Waiting for External Conditions

### Common Scenarios

Many applications need to wait for external conditions:

- **Service Availability**: Waiting for a service to become healthy after deployment
- **Resource Provisioning**: Waiting for cloud resources to be created
- **Data Propagation**: Waiting for data to replicate across systems
- **Status Changes**: Waiting for a job or process to reach a specific state
- **External Events**: Waiting for webhooks, notifications, or events

### Naive Approaches and Their Limitations

**Approach 1: Simple Sleep**
```rust
// Bad: Fixed wait time, no early exit ❌
async fn wait_for_condition() {
    tokio::time::sleep(Duration::from_secs(30)).await;
    // Hope the condition is met...
}
```

**Problems:**
- No way to check if condition is met early
- Wastes time if condition completes quickly
- May timeout before condition is met

**Approach 2: Simple Loop**
```rust
// Better but still limited ✅
async fn wait_for_condition() {
    loop {
        if check_condition().await {
            break;
        }
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
}
```

**Problems:**
- No timeout mechanism
- No configurable interval
- Not reusable for different conditions

---

## The Solution: Generic Polling with Timeouts

A robust polling mechanism should:
1. **Check conditions periodically** at configurable intervals
2. **Support timeouts** to prevent infinite waiting
3. **Be generic** and reusable for different conditions
4. **Provide early exit** when conditions are met
5. **Handle errors gracefully**

---

## Building a Generic Polling Function

### Basic Structure

Here's a generic polling function that accepts a condition closure:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

async fn poll_until_condition<F, Fut>(
    timeout_secs: u64,
    interval_secs: u64,
    mut condition: F,
) -> Result<bool, String>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<bool, String>>,
{
    let timeout = Duration::from_secs(timeout_secs);
    let interval = Duration::from_secs(interval_secs);
    let start_time = Instant::now();

    loop {
        // Check if timeout has been reached
        if start_time.elapsed() >= timeout {
            return Err(format!(
                "Condition not met after {} seconds",
                timeout_secs
            ));
        }

        // Check the condition
        match condition().await {
            Ok(true) => {
                return Ok(true); // Condition met!
            }
            Ok(false) => {
                // Condition not met yet, wait and retry
            }
            Err(e) => {
                // Log error but continue polling
                eprintln!("Error checking condition: {}", e);
            }
        }

        // Wait before next check
        sleep(interval).await;
    }
}
```

### Key Features

1. **Generic Condition Function**: Accepts any closure that returns a `Future<Output = Result<bool, String>>`
2. **Configurable Timeout**: Maximum time to wait
3. **Configurable Interval**: Time between condition checks
4. **Early Exit**: Returns immediately when condition is met
5. **Error Handling**: Continues polling even if condition check fails

---

## Real-World Use Cases

### 1. Waiting for Service Health

After deploying a service, you might need to wait for it to become healthy:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

struct HealthChecker {
    service_url: String,
}

impl HealthChecker {
    async fn check_health(&self) -> Result<bool, String> {
        // Simulated health check
        let response = reqwest::get(&self.service_url).await?;
        Ok(response.status().is_success())
    }

    async fn wait_for_healthy(
        &self,
        timeout_secs: u64,
        interval_secs: u64,
    ) -> Result<(), String> {
        let timeout = Duration::from_secs(timeout_secs);
        let interval = Duration::from_secs(interval_secs);
        let start_time = Instant::now();

        loop {
            if start_time.elapsed() >= timeout {
                return Err(format!(
                    "Service did not become healthy after {} seconds",
                    timeout_secs
                ));
            }

            match self.check_health().await {
                Ok(true) => {
                    println!("Service is healthy!");
                    return Ok(());
                }
                Ok(false) => {
                    println!("Service not healthy yet, waiting...");
                }
                Err(e) => {
                    eprintln!("Health check error: {}", e);
                }
            }

            sleep(interval).await;
        }
    }
}
```

### 2. Waiting for Resource Provisioning

When provisioning cloud resources, you need to wait for them to be ready:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

struct ResourceManager {
    api_client: ApiClient,
}

impl ResourceManager {
    async fn check_resource_status(&self, resource_id: &str) -> Result<bool, String> {
        // Simulated API call
        let status = self.api_client.get_resource_status(resource_id).await?;
        Ok(status == "ready")
    }

    async fn wait_for_resource_ready(
        &self,
        resource_id: String,
        timeout_secs: u64,
        interval_secs: u64,
    ) -> Result<(), String> {
        let timeout = Duration::from_secs(timeout_secs);
        let interval = Duration::from_secs(interval_secs);
        let start_time = Instant::now();

        loop {
            if start_time.elapsed() >= timeout {
                return Err(format!(
                    "Resource {} not ready after {} seconds",
                    resource_id, timeout_secs
                ));
            }

            match self.check_resource_status(&resource_id).await {
                Ok(true) => {
                    println!("Resource {} is ready!", resource_id);
                    return Ok(());
                }
                Ok(false) => {
                    let elapsed = start_time.elapsed().as_secs();
                    println!(
                        "Resource {} still provisioning... ({}s elapsed)",
                        resource_id, elapsed
                    );
                }
                Err(e) => {
                    eprintln!("Error checking resource status: {}", e);
                }
            }

            sleep(interval).await;
        }
    }
}
```

### 3. Waiting for Data Propagation

After updating records in a distributed system, you might need to wait for propagation:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

struct DataService {
    primary_endpoint: String,
    replica_endpoints: Vec<String>,
}

impl DataService {
    async fn check_propagation(&self, record_id: &str) -> Result<bool, String> {
        // Check if record exists on all replicas
        let mut all_propagated = true;

        for endpoint in &self.replica_endpoints {
            let url = format!("{}/records/{}", endpoint, record_id);
            match reqwest::get(&url).await {
                Ok(response) => {
                    if !response.status().is_success() {
                        all_propagated = false;
                        break;
                    }
                }
                Err(_) => {
                    all_propagated = false;
                    break;
                }
            }
        }

        Ok(all_propagated)
    }

    async fn wait_for_propagation(
        &self,
        record_id: String,
        timeout_secs: u64,
        interval_secs: u64,
    ) -> Result<(), String> {
        let timeout = Duration::from_secs(timeout_secs);
        let interval = Duration::from_secs(interval_secs);
        let start_time = Instant::now();

        loop {
            if start_time.elapsed() >= timeout {
                return Err(format!(
                    "Record {} did not propagate after {} seconds",
                    record_id, timeout_secs
                ));
            }

            match self.check_propagation(&record_id).await {
                Ok(true) => {
                    println!("Record {} propagated to all replicas!", record_id);
                    return Ok(());
                }
                Ok(false) => {
                    let elapsed = start_time.elapsed().as_secs();
                    println!(
                        "Waiting for propagation of {}... ({}s elapsed)",
                        record_id, elapsed
                    );
                }
                Err(e) => {
                    eprintln!("Error checking propagation: {}", e);
                }
            }

            sleep(interval).await;
        }
    }
}
```

---

## Advanced Patterns

### Pattern 1: Generic Polling Function with Logging

A more sophisticated version with better logging and error handling:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

async fn poll_with_logging<F, Fut>(
    timeout_secs: u64,
    interval_secs: u64,
    mut condition: F,
    condition_name: &str,
) -> Result<(), String>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<bool, String>>,
{
    let timeout = Duration::from_secs(timeout_secs);
    let interval = Duration::from_secs(interval_secs);
    let start_time = Instant::now();

    println!("Waiting for {} (timeout: {}s, interval: {}s)",
             condition_name, timeout_secs, interval_secs);

    loop {
        let elapsed = start_time.elapsed();

        if elapsed >= timeout {
            return Err(format!(
                "{} not met after {} seconds",
                condition_name, timeout_secs
            ));
        }

        match condition().await {
            Ok(true) => {
                println!(
                    "{} met after {} seconds",
                    condition_name,
                    elapsed.as_secs()
                );
                return Ok(());
            }
            Ok(false) => {
                println!(
                    "Waiting for {}... ({}s elapsed, {}s remaining)",
                    condition_name,
                    elapsed.as_secs(),
                    (timeout - elapsed).as_secs()
                );
            }
            Err(e) => {
                eprintln!("Error checking {}: {}", condition_name, e);
            }
        }

        sleep(interval).await;
    }
}
```

### Pattern 2: Exponential Backoff

For scenarios where you want to reduce polling frequency over time:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

async fn poll_with_exponential_backoff<F, Fut>(
    timeout_secs: u64,
    initial_interval_secs: u64,
    max_interval_secs: u64,
    mut condition: F,
) -> Result<(), String>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<bool, String>>,
{
    let timeout = Duration::from_secs(timeout_secs);
    let mut current_interval = Duration::from_secs(initial_interval_secs);
    let max_interval = Duration::from_secs(max_interval_secs);
    let start_time = Instant::now();

    loop {
        let elapsed = start_time.elapsed();

        if elapsed >= timeout {
            return Err(format!(
                "Condition not met after {} seconds",
                timeout_secs
            ));
        }

        match condition().await {
            Ok(true) => {
                return Ok(());
            }
            Ok(false) => {
                // Exponential backoff: double the interval, up to max
                current_interval = (current_interval * 2).min(max_interval);
                println!(
                    "Condition not met, waiting {}s before next check",
                    current_interval.as_secs()
                );
            }
            Err(e) => {
                eprintln!("Error checking condition: {}", e);
            }
        }

        sleep(current_interval).await;
    }
}
```

### Pattern 3: Cancellation Support

Adding support for cancellation via a channel:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;
use tokio::sync::oneshot;

async fn poll_with_cancellation<F, Fut>(
    timeout_secs: u64,
    interval_secs: u64,
    mut condition: F,
    mut cancel_rx: oneshot::Receiver<()>,
) -> Result<bool, String>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<bool, String>>,
{
    let timeout = Duration::from_secs(timeout_secs);
    let interval = Duration::from_secs(interval_secs);
    let start_time = Instant::now();

    loop {
        // Check for cancellation
        if cancel_rx.try_recv().is_ok() {
            return Err("Polling cancelled".to_string());
        }

        if start_time.elapsed() >= timeout {
            return Err(format!(
                "Condition not met after {} seconds",
                timeout_secs
            ));
        }

        match condition().await {
            Ok(true) => {
                return Ok(true);
            }
            Ok(false) => {
                // Continue polling
            }
            Err(e) => {
                eprintln!("Error checking condition: {}", e);
            }
        }

        // Use tokio::select! to wait for either interval or cancellation
        tokio::select! {
            _ = sleep(interval) => {
                // Continue polling
            }
            _ = &mut cancel_rx => {
                return Err("Polling cancelled".to_string());
            }
        }
    }
}
```

---

## Complete Rust Implementation Example

Here's a complete, educational example demonstrating the polling pattern:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

/// Generic polling function that waits for a condition to be met
async fn wait_for_condition<F, Fut>(
    timeout_secs: u64,
    interval_secs: u64,
    mut condition: F,
) -> Result<(), String>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<bool, String>>,
{
    let timeout = Duration::from_secs(timeout_secs);
    let interval = Duration::from_secs(interval_secs);
    let start_time = Instant::now();

    println!(
        "Waiting for condition (timeout: {}s, check interval: {}s)",
        timeout_secs, interval_secs
    );

    loop {
        let elapsed = start_time.elapsed();

        // Check timeout
        if elapsed >= timeout {
            return Err(format!(
                "Condition not met after {} seconds",
                timeout_secs
            ));
        }

        // Check condition
        match condition().await {
            Ok(true) => {
                println!(
                    "Condition met after {} seconds",
                    elapsed.as_secs()
                );
                return Ok(());
            }
            Ok(false) => {
                let remaining = (timeout - elapsed).as_secs();
                println!(
                    "Condition not met yet ({}s elapsed, {}s remaining)",
                    elapsed.as_secs(),
                    remaining
                );
            }
            Err(e) => {
                eprintln!("Error checking condition: {}", e);
                // Continue polling despite error
            }
        }

        // Wait before next check
        sleep(interval).await;
    }
}

// Example: Waiting for a file to exist
async fn wait_for_file(path: &str, timeout_secs: u64) -> Result<(), String> {
    wait_for_condition(timeout_secs, 1, || async {
        Ok(std::path::Path::new(path).exists())
    })
    .await
}

// Example: Waiting for an API endpoint to return success
struct ApiChecker {
    url: String,
}

impl ApiChecker {
    async fn check(&self) -> Result<bool, String> {
        // Simulated API check
        tokio::time::sleep(Duration::from_millis(100)).await;

        // Simulate: succeed after some time
        let random = (Instant::now().elapsed().as_secs() % 10) == 0;
        Ok(random)
    }

    async fn wait_until_ready(&self, timeout_secs: u64) -> Result<(), String> {
        wait_for_condition(timeout_secs, 2, || self.check()).await
    }
}

#[tokio::main]
async fn main() {
    // Example 1: Wait for file
    println!("Example 1: Waiting for file");
    match wait_for_file("/tmp/ready.txt", 5).await {
        Ok(_) => println!("File found!"),
        Err(e) => println!("Error: {}", e),
    }

    // Example 2: Wait for API
    println!("\nExample 2: Waiting for API");
    let checker = ApiChecker {
        url: "https://api.example.com/health".to_string(),
    };

    match checker.wait_until_ready(10).await {
        Ok(_) => println!("API is ready!"),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## Best Practices

### 1. **Choose Appropriate Intervals**

- **Too frequent**: Wastes resources and may overwhelm the system
- **Too infrequent**: Slower response time when condition is met

```rust
// Good: Reasonable interval based on expected time
wait_for_condition(60, 2, condition).await; // Check every 2s, timeout 60s

// Less ideal: Too frequent
wait_for_condition(60, 0.1, condition).await; // Checks 600 times!

// Less ideal: Too infrequent
wait_for_condition(60, 30, condition).await; // Only checks twice
```

### 2. **Set Realistic Timeouts**

Consider the expected time for the condition to be met:

```rust
// Good: Timeout allows for normal operation time
wait_for_condition(300, 5, condition).await; // 5 minutes for deployment

// Less ideal: Too short
wait_for_condition(10, 1, condition).await; // May timeout too early

// Less ideal: Too long
wait_for_condition(3600, 1, condition).await; // 1 hour may be excessive
```

### 3. **Handle Errors Gracefully**

Don't fail immediately on condition check errors - they might be transient:

```rust
match condition().await {
    Ok(true) => return Ok(()),
    Ok(false) => continue, // Expected: condition not met yet
    Err(e) => {
        // Log but continue - might be transient network issue
        eprintln!("Transient error: {}", e);
        continue;
    }
}
```

### 4. **Provide Progress Feedback**

Users appreciate knowing that something is happening:

```rust
let elapsed = start_time.elapsed().as_secs();
let remaining = (timeout - elapsed).as_secs();
println!("Waiting... ({}s elapsed, {}s remaining)", elapsed, remaining);
```

### 5. **Use Exponential Backoff for Rate-Limited Operations**

If you're hitting rate limits, increase intervals over time:

```rust
let mut interval = Duration::from_secs(1);
let max_interval = Duration::from_secs(30);

loop {
    // ... check condition ...

    // Exponential backoff
    interval = (interval * 2).min(max_interval);
    sleep(interval).await;
}
```

---

## Comparison with Alternatives

### Polling vs Event-Driven

| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Polling** | Simple, works with any system | Wastes resources, latency | When events aren't available |
| **Event-Driven** | Efficient, real-time | Requires event system | When events are available |

### Polling vs Fixed Sleep

| Approach | Pros | Cons |
|----------|------|------|
| **Polling** | Early exit, configurable | More complex |
| **Fixed Sleep** | Simple | Wastes time, no early exit |

---

## Testing Polling Logic

Testing polling functions requires careful consideration of time:

```rust
#[tokio::test]
async fn test_condition_met_before_timeout() {
    use std::sync::{Arc, Mutex};
    use tokio::time::timeout;

    let condition_met = Arc::new(Mutex::new(false));
    let condition_met_clone = condition_met.clone();

    let condition = || {
        let condition_met_clone = condition_met_clone.clone();
        async move {
            let mut met = condition_met_clone.lock().unwrap();
            if *met {
                Ok(true)
            } else {
                *met = true; // Simulate condition being met
                Ok(false)
            }
        }
    };

    // Should complete quickly since condition is met on first check
    let result = timeout(
        Duration::from_secs(5),
        wait_for_condition(10, 1, condition)
    ).await;

    assert!(result.is_ok());
    assert!(result.unwrap().is_ok());
}

#[tokio::test]
async fn test_timeout_reached() {
    use tokio::time::timeout;

    let condition = || async {
        // Condition never met
        Ok(false)
    };

    // Should timeout
    let result = timeout(
        Duration::from_secs(3),
        wait_for_condition(2, 1, condition)
    ).await;

    // The inner future should complete (with error), not timeout
    assert!(result.is_ok());
    assert!(result.unwrap().is_err());
}
```

---

## Common Pitfalls

### 1. **Forgetting to Check Timeout**

Always check timeout at the start of each iteration:

```rust
// Good ✅
loop {
    if start_time.elapsed() >= timeout {
        return Err("Timeout");
    }
    // ... check condition ...
}

// Bad ❌
loop {
    // ... check condition ...
    if start_time.elapsed() >= timeout {  // Too late!
        return Err("Timeout");
    }
}
```

### 2. **Not Handling Condition Check Errors**

Always handle errors from condition checks:

```rust
// Good ✅
match condition().await {
    Ok(true) => return Ok(()),
    Ok(false) => continue,
    Err(e) => {
        eprintln!("Error: {}", e);
        continue; // Don't fail on transient errors
    }
}

// Bad ❌
if condition().await? {  // Fails immediately on error
    return Ok(());
}
```

### 3. **Blocking the Async Runtime**

Don't use blocking operations in condition checks:

```rust
// Bad ❌
async fn check_condition() -> Result<bool, String> {
    std::thread::sleep(Duration::from_secs(1)); // Blocks!
    Ok(true)
}

// Good ✅
async fn check_condition() -> Result<bool, String> {
    tokio::time::sleep(Duration::from_secs(1)).await; // Non-blocking
    Ok(true)
}
```

---

## Conclusion

Retry and polling patterns are essential for building robust async applications that need to wait for external conditions. The key principles are:

1. **Generic Design**: Use closures to make polling functions reusable
2. **Configurable Timeouts**: Prevent infinite waiting
3. **Appropriate Intervals**: Balance responsiveness with resource usage
4. **Error Handling**: Handle transient errors gracefully
5. **Progress Feedback**: Keep users informed of progress

Whether you're waiting for services to become healthy, resources to be provisioned, or data to propagate, these patterns provide a solid foundation for reliable async operations.

### Key Takeaways

1. Use generic polling functions with closure-based conditions
2. Always implement timeouts to prevent infinite loops
3. Choose intervals based on expected operation time
4. Handle errors gracefully - don't fail on transient issues
5. Provide progress feedback for better user experience
6. Consider exponential backoff for rate-limited operations

### Further Reading

- [Tokio time documentation](https://docs.rs/tokio/latest/tokio/time/index.html)
- [Rust async book](https://rust-lang.github.io/async-book/)
- [Exponential backoff algorithms](https://en.wikipedia.org/wiki/Exponential_backoff)
- [Circuit breaker pattern](https://martinfowler.com/bliki/CircuitBreaker.html)

---
