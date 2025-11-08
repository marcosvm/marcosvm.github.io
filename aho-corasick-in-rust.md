# Aho-Corasick Algorithm: Efficient Multi-Pattern String Matching in Rust

## Introduction

When building command-line tools and applications that process text, one common challenge is efficiently replacing multiple patterns in a single pass. Traditional approaches like iterating through patterns and using simple string replacement can be slow and error-prone, especially when dealing with overlapping matches or large datasets.

This presentation explores the **Aho-Corasick algorithm**, a powerful string-searching algorithm that excels at finding multiple patterns simultaneously. We'll dive into how it works, why it's beneficial, and how it's used in real-world applications—specifically in the `orc` CLI tool for database orchestration.

---

## What is Aho-Corasick?

### The Algorithm

The **Aho-Corasick algorithm** is a string-searching algorithm invented by Alfred V. Aho and Margaret J. Corasick in 1975. It's designed to find all occurrences of multiple patterns (a dictionary) within a text in a single pass.

### Key Concepts

1. **Trie Structure**: The algorithm builds a trie (prefix tree) from all the patterns to search for
2. **Failure Links**: Similar to the KMP algorithm's failure function, it uses failure links to handle mismatches efficiently
3. **Output Links**: These links point to other patterns that are suffixes of the current state, ensuring all matches are found

### How It Works

```
Patterns: ["he", "she", "his", "hers"]
Text: "ushers"

1. Build a trie from all patterns
2. Add failure links (like KMP, but for multiple patterns)
3. Add output links (for overlapping matches)
4. Traverse the text once, following the automaton
```

The algorithm constructs a **finite automaton** that can efficiently search for all patterns simultaneously, making it much faster than naive approaches that search for each pattern separately.

---

## Benefits of Using Aho-Corasick

### 1. **Time Complexity**

- **Naive approach**: O(n × m × k) where n = text length, m = average pattern length, k = number of patterns
- **Aho-Corasick**: O(n + m + z) where z = number of matches found
- **Preprocessing**: O(m) where m = total length of all patterns

This makes it particularly efficient when:
- Searching for many patterns in large texts
- Patterns share common prefixes
- The same text needs to be searched multiple times (automaton can be reused)

### 2. **Single-Pass Processing**

Unlike naive approaches that require multiple passes through the text, Aho-Corasick processes the input text exactly once, making it:
- **Faster**: No redundant scanning
- **Memory efficient**: Lower memory overhead
- **Stream-friendly**: Can process streaming data efficiently

### 3. **Handles Overlapping Matches**

The algorithm correctly handles cases where patterns overlap or one pattern is a substring of another, ensuring all matches are found without conflicts.

### 4. **Deterministic Behavior**

With configurable match kinds (LeftmostFirst, LeftmostLongest, Standard), you can control which match is selected when multiple patterns match at the same position, providing predictable and consistent results.

### 5. **Ideal for Text Replacement**

When replacing multiple patterns, Aho-Corasick ensures:
- All occurrences are found
- No double-replacement issues
- Consistent ordering based on match kind

---

## How Aho-Corasick is Used in the orc CLI

### Context: The orc Tool

The `orc` CLI is an opinionated command-line client for the Orchestrator API, used for managing database clusters. One of its features is displaying cluster topology and status information in a human-readable format.

### The Problem

When displaying database cluster information, the API returns full hostnames like:
```
production.sys.az1.eng.pdx.wd:3306
drprimary.sys.az1.eng.pdx.wd:3306
drreplica.sys.az1.eng.pdx.wd:3306
```

These are hard to read and take up significant screen space. However, the system maintains CNAME mappings that provide shorter, more readable aliases:
- `production.sys.az1.eng.pdx.wd:3306` → `production:3306`
- `drprimary.sys.az1.eng.pdx.wd:3306` → `drprimary:3306`
- `drreplica.sys.az1.eng.pdx.wd:3306` → `drreplica:3306`

### The Solution

The `process_resolution` function in the `client::text` module uses Aho-Corasick to replace all CNAME patterns with their shorter aliases in a single pass:

```rust
// Simplified version of the actual implementation
pub(super) fn process_resolution(text: &str, hosts: &Vec<Host>) -> String {
    let mut from = vec![];
    let mut to = vec![];

    // Build replacement patterns
    for host in hosts {
        from.push(format!("{}:{}", &host.cname, &host.port));
        to.push(format!("{}:{}", &host.hostname, &host.port));
    }

    // Build Aho-Corasick automaton
    let ac = AhoCorasickBuilder::new()
        .match_kind(MatchKind::LeftmostFirst)
        .build(from);

    // Replace all occurrences in a single pass
    let mut result = String::new();
    ac.replace_all_with(text, &mut result, |mat, _, dst| {
        dst.push_str(&to[mat.pattern()]);
        true
    });

    result
}
```

### Why Aho-Corasick?

1. **Multiple Patterns**: The function needs to replace potentially dozens of hostname patterns simultaneously
2. **Performance**: Status output can be large, and the replacement happens frequently during CLI operations
3. **Correctness**: Using `LeftmostFirst` ensures predictable replacement when patterns might overlap
4. **User Experience**: The transformation happens transparently, improving readability without affecting functionality

### Example Output

**Before resolution:**
```
production.sys.az1.eng.pdx.wd:3306   |0s|ok|5.7.31-log|rw|STATEMENT|>>,P-GTID
+ drprimary.sys.az1.eng.pdx.wd:3306  |0s|ok|5.7.31-log|ro|STATEMENT|>>,P-GTID
+ drreplica.sys.az1.eng.pdx.wd:3306|0s|ok|5.7.31-log|ro|STATEMENT|>>,P-GTID
```

**After resolution:**
```
production:3306   |0s|ok|5.7.31-log|rw|STATEMENT|>>,P-GTID
+ drprimary:3306  |0s|ok|5.7.31-log|ro|STATEMENT|>>,P-GTID
+ drreplica:3306|0s|ok|5.7.31-log|ro|STATEMENT|>>,P-GTID
```

Much more readable! 🎉

---

## Real-World Examples

### 1. **Text Editors and IDEs**

Modern code editors use Aho-Corasick for:
- **Syntax highlighting**: Finding keywords, operators, and patterns simultaneously
- **Search and replace**: Multi-pattern search across files
- **Code analysis**: Finding multiple code patterns for refactoring or linting

### 2. **Network Intrusion Detection Systems (IDS)**

Security systems use Aho-Corasick to:
- **Signature matching**: Detect multiple attack patterns in network traffic
- **Malware detection**: Identify known malicious code patterns
- **Log analysis**: Search for multiple threat indicators simultaneously

### 3. **Bioinformatics**

In DNA/RNA sequence analysis:
- **Pattern matching**: Finding multiple gene sequences in genomic data
- **Motif discovery**: Identifying regulatory patterns
- **Sequence alignment**: Efficiently locating multiple reference patterns

### 4. **Web Application Firewalls (WAF)**

WAFs use Aho-Corasick to:
- **SQL injection detection**: Match multiple SQL injection patterns
- **XSS prevention**: Detect cross-site scripting attack patterns
- **Input validation**: Check for multiple malicious input patterns

### 5. **Log Processing and Monitoring**

Tools like log aggregators use it for:
- **Pattern extraction**: Finding multiple log patterns simultaneously
- **Alerting**: Detecting multiple alert conditions in logs
- **Data transformation**: Replacing multiple patterns during log parsing

### 6. **Content Filtering**

Applications use Aho-Corasick for:
- **Profanity filtering**: Replacing multiple inappropriate words
- **Sensitive data masking**: Redacting multiple types of PII (SSN, credit cards, etc.)
- **Template processing**: Replacing multiple placeholders in templates

---

## Rust Implementation Example

Here's a complete, educational Rust example demonstrating Aho-Corasick for text replacement. This example is different from the proprietary codebase but illustrates the same concepts:

```rust
use aho_corasick::{AhoCorasick, AhoCorasickBuilder, MatchKind};

/// A simple text processor that replaces multiple patterns efficiently
pub struct TextProcessor {
    patterns: Vec<String>,
    replacements: Vec<String>,
    automaton: AhoCorasick,
}

impl TextProcessor {
    /// Create a new TextProcessor with patterns and their replacements
    pub fn new(patterns: Vec<String>, replacements: Vec<String>) -> Self {
        assert_eq!(
            patterns.len(),
            replacements.len(),
            "Patterns and replacements must have the same length"
        );

        // Build the Aho-Corasick automaton
        let automaton = AhoCorasickBuilder::new()
            .match_kind(MatchKind::LeftmostFirst) // Use leftmost-first matching
            .build(&patterns)
            .expect("Failed to build automaton");

        Self {
            patterns,
            replacements,
            automaton,
        }
    }

    /// Replace all occurrences of patterns in the input text
    pub fn replace_all(&self, text: &str) -> String {
        let mut result = String::new();
        
        self.automaton.replace_all_with(text, &mut result, |mat, _, dst| {
            // Get the replacement string for the matched pattern
            let replacement = &self.replacements[mat.pattern()];
            dst.push_str(replacement);
            true // Continue processing
        });

        result
    }

    /// Count how many times each pattern appears in the text
    pub fn count_matches(&self, text: &str) -> Vec<(String, usize)> {
        let mut counts = vec![0; self.patterns.len()];
        
        for mat in self.automaton.find_iter(text) {
            counts[mat.pattern()] += 1;
        }

        self.patterns
            .iter()
            .zip(counts.iter())
            .map(|(pattern, &count)| (pattern.clone(), count))
            .collect()
    }
}

// Example usage
fn main() {
    // Define patterns and their replacements
    let patterns = vec![
        "Rust".to_string(),
        "Python".to_string(),
        "JavaScript".to_string(),
    ];

    let replacements = vec![
        "🦀 Rust".to_string(),
        "🐍 Python".to_string(),
        "🟨 JavaScript".to_string(),
    ];

    // Create the processor
    let processor = TextProcessor::new(patterns, replacements);

    // Process some text
    let input = r#"
    I love programming in Rust, Python, and JavaScript.
    Rust is fast, Python is versatile, and JavaScript runs everywhere.
    When I write Rust code, I feel productive.
    "#;

    println!("Original text:\n{}", input);
    println!("\n---\n");

    // Replace all patterns
    let output = processor.replace_all(input);
    println!("After replacement:\n{}", output);
    println!("\n---\n");

    // Count matches
    let counts = processor.count_matches(input);
    println!("Pattern counts:");
    for (pattern, count) in counts {
        println!("  {}: {}", pattern, count);
    }
}
```

### Example Output

```
Original text:

    I love programming in Rust, Python, and JavaScript.
    Rust is fast, Python is versatile, and JavaScript runs everywhere.
    When I write Rust code, I feel productive.
    

---

After replacement:

    I love programming in 🦀 Rust, 🐍 Python, and 🟨 JavaScript.
    🦀 Rust is fast, 🐍 Python is versatile, and 🟨 JavaScript runs everywhere.
    When I write 🦀 Rust code, I feel productive.
    

---

Pattern counts:
  Rust: 3
  Python: 2
  JavaScript: 2
```

### Key Features of This Implementation

1. **Reusable Automaton**: The automaton is built once and can be reused for multiple texts
2. **Efficient Replacement**: Single-pass replacement using `replace_all_with`
3. **Match Counting**: Demonstrates how to count pattern occurrences
4. **Configurable Matching**: Uses `LeftmostFirst` for predictable behavior

### Advanced Example: Sensitive Data Masking

Here's a more practical example for masking sensitive information:

```rust
use aho_corasick::{AhoCorasickBuilder, MatchKind};

fn mask_sensitive_data(text: &str) -> String {
    // Patterns to mask (simplified - real implementations use regex for validation)
    let patterns = vec![
        "SSN",
        "credit card",
        "password",
        "API key",
    ];

    let replacements = vec![
        "[REDACTED-SSN]",
        "[REDACTED-CARD]",
        "[REDACTED-PASSWORD]",
        "[REDACTED-API-KEY]",
    ];

    let ac = AhoCorasickBuilder::new()
        .match_kind(MatchKind::LeftmostFirst)
        .build(&patterns)
        .unwrap();

    let mut result = String::new();
    ac.replace_all_with(text, &mut result, |mat, _, dst| {
        dst.push_str(&replacements[mat.pattern()]);
        true
    });

    result
}

fn main() {
    let log_entry = "User logged in with password: secret123, SSN: 123-45-6789";
    let masked = mask_sensitive_data(log_entry);
    println!("Original: {}", log_entry);
    println!("Masked:   {}", masked);
}
```

---

## Performance Considerations

### When to Use Aho-Corasick

✅ **Use Aho-Corasick when:**
- You need to search for multiple patterns simultaneously
- Patterns are known at compile time or can be preprocessed
- You're processing large texts or many texts with the same patterns
- You need to replace multiple patterns efficiently
- Patterns might overlap or be substrings of each other

❌ **Consider alternatives when:**
- You only need to search for a single pattern (use KMP or Boyer-Moore)
- Patterns are complex regex patterns (use regex engines)
- Patterns change frequently and preprocessing cost isn't amortized
- Memory is extremely constrained (automaton can be large)

### Benchmarking Tips

When benchmarking Aho-Corasick:
1. **Preprocess once**: Build the automaton outside the hot path
2. **Reuse the automaton**: Don't rebuild for each text
3. **Consider match kind**: `LeftmostFirst` is faster than `LeftmostLongest`
4. **Profile your use case**: Real-world performance depends on pattern characteristics

---

## Conclusion

The Aho-Corasick algorithm is a powerful tool for efficient multi-pattern string matching and replacement. Its key advantages include:

- **Efficiency**: O(n + m + z) time complexity for searching
- **Single-pass processing**: Processes text exactly once
- **Handles complexity**: Correctly deals with overlapping patterns
- **Production-ready**: Well-tested implementations available in Rust

In the `orc` CLI, Aho-Corasick enables seamless hostname resolution, improving the user experience by making output more readable without sacrificing performance. This same technique can be applied to countless other use cases where efficient multi-pattern text processing is needed.

### Further Reading

- [Aho-Corasick Algorithm (Wikipedia)](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm)
- [aho-corasick crate documentation](https://docs.rs/aho-corasick/)
- [Original paper by Aho and Corasick (1975)](https://dl.acm.org/doi/10.1145/360825.360855)

---

*This presentation was created to explain the use of Aho-Corasick in the orc CLI tool. For questions or contributions, please refer to the project repository.*
