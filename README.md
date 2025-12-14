# mdb-guide-markdown

---

**Title:** Vibe Coding with MongoDB: How "Rules Files" Keep Your AI Assistant on the Rails

**Subtitle:** Modern software development is a high-speed dance with AI. Here’s how defining clear architectural rules enables you to stay in the "vibe coding" flow while ensuring your MongoDB implementation is robust, asynchronous, and production-ready.

-----

We’ve all experienced "vibe coding." It’s that flow state where the music is right, the coffee is kicking in, and complex features seem to flow effortlessly from your brain through your fingertips.

In the modern era, AI coding assistants (like Cursor, Copilot, or Windsurf) act as accelerators for this flow. They handle the boilerplate, suggest the next logical step, and auto-complete your thoughts.

But there’s a screeching halt that often breaks the vibe: **infrastructure complexity.**

Suddenly, you aren't just writing business logic; you're wrestling with architectural decisions. *Should this db call be async? How do I define an Atlas Vector Search index programmatically? Why is my aggregation pipeline slow?*

You stop coding. You open twenty browser tabs of documentation. You browse Stack Overflow. The vibe is dead.

### The Challenge with Modern MongoDB

MongoDB today is far more than just a JSON document store. It's a sophisticated data platform that includes full-text search, vector search for AI applications, and reactive, non-blocking driver implementations.

While powerful, these features come with nuanced best practices that are easy to miss if you aren't working with them daily:

1.  **The Async Trap:** In modern frameworks like FastAPI or Node.js, a single synchronous database call can block your entire server's event loop, tanking performance under load.
2.  **The Search Index Delay:** Unlike standard indexes, Atlas Search and Vector indexes build asynchronously in the background. If your code creates an index and immediately tries to query it (common in tests or startup logic), it will fail unless you implement a robust polling mechanism.
3.  **Aggregation ordering:** Placing a `$match` stage *before* a `$vectorSearch` stage seems logical, but it’s a massive performance anti-pattern.

We want our AI assistants to know these things automatically so we don't have to constantly course-correct them.

### Enter the "Rules File" (.mdc)

This is where context files (like Cursor's `.mdc` format) change the game.

Think of an `.mdc` file not just as a repository of code snippets, but as the **architectural conscience** of your project. It tells the AI *how* we build things here.

By codifying MongoDB best practices into a rules file, we transform the AI assistant from a generic coder into a specialized MongoDB architect that knows your project's specific constraints.

Let’s look at the value of having a well-defined MongoDB rules file.

#### 1\. Enforcing Architecture (Async vs. Sync)

If you are building a high-performance FastAPI backend, mixing synchronous `pymongo` calls with asynchronous `motor` calls is a recipe for disaster.

Without rules, an AI might see a simple data fetch and suggest the simpler, synchronous code. With a rules file explicitly stating, "Async Contexts MUST use asynchronous drivers," the AI stays on the rails.

**Without rules:** The AI might suggest blocking code that looks fine but fails at scale.

**With rules:** The AI knows: "Ah, this is a FastAPI route. I must use `motor` and `await` the result."

#### 2\. Handling the "Wait for Ready" Pattern

Creating vector indexes programmatically is vital for Infrastructure-as-Code. But the logic to robustly create an index, handle race conditions if it already exists, and poll Atlas until it's "queryable" is tedious, dozens of lines of boilerplate.

Nobody wants to type that out during a vibe coding session.

By defining this pattern once in the rules file, you can simply ask the AI: *"Create a function to ensure a vector index exists on the 'bio' field."*

Because the rules file defines the "Wait for Ready" pattern and idempotency checks, the AI generates the entire, robust, production-grade block of code instantly. You stay in the flow; the infrastructure gets handled correctly.

#### 3\. Optimizing Aggregations Automatically

Performance tuning aggregation pipelines often involves subtle ordering rules. A common mistake in vector search is filtering results *after* the search instead of *during* the search.

A rule stating: *"Specialized operators like `$vectorSearch` MUST be the very first stage; use pre-filtering inside the operator"* ensures the AI doesn't offer sub-optimal pipelines. It guides the AI toward the performant solution automatically.

### Conclusion: Guardrails for Speed

The goal of AI-assisted coding isn't just to write code faster; it's to write *better* code faster.

By taking the time to define a comprehensive rules file for your MongoDB implementation, you provide the necessary guardrails. You allow junior developers to produce code that adheres to senior architectural standards, and you allow senior developers to stay in the "vibe coding" zone without getting bogged down in infrastructure boilerplate.

Below is a comprehensive `.mdc` rule file you can drop into your project to immediately level up your AI assistant's MongoDB capabilities.

-----

### Resource: The Ultimate MongoDB `.mdc` Rules File

Save the following as `.cursor/rules/mongodb-best-practices.mdc` (or the equivalent context file for your AI tool of choice).

````markdown
---
description: Comprehensive rules and best practices for MongoDB projects, focusing on Driver Selection, Atlas Search/Vector Indexing, and Aggregation Pipeline optimization.
globs: **/*.py, **/*.js, **/*.ts, **/*.go, **/*.java
---

# MongoDB & Atlas Search Implementation Guidelines

## 1. Architecture & Driver Selection

**Context:** The choice of database driver must strictly match the application's I/O architecture. Mixing blocking and non-blocking patterns causes severe performance degradation or deadlocks.

* **Rule (Async Contexts):** When working in asynchronous frameworks (e.g., FastAPI, Node.js, Go/Goroutines), you **MUST** use asynchronous drivers (e.g., `motor`, native Node driver).
    * *Constraint:* Never use synchronous/blocking driver calls inside an async function. This blocks the main event loop.
* **Rule (Sync Contexts):** In synchronous scripts, CLI tools, or legacy WSGI apps, prefer standard synchronous drivers (e.g., `pymongo`) for simplicity.
* **Pattern:** Wrap database utilities in the native paradigm of the framework.

```python
# ✅ Correct: Async usage (FastAPI/Tornado)
async def get_user_async(collection, user_id):
    return await collection.find_one({"user_id": user_id})

# ✅ Correct: Sync usage (CLI/Scripts)
def get_user_sync(collection, user_id):
    return collection.find_one({"user_id": user_id})
````

## 2\. Robust Index Management (Atlas Search & Vector)

**Context:** Unlike standard B-tree indexes, Atlas Search (Vector/Lucene) indexes are built asynchronously on the cloud side. The API command returns *immediately*, but the index is not ready for query execution until the build finishes.

### A. Programmatic Definition (SearchIndexModel)

  * **Rule:** Define index configurations programmatically using typed models (e.g., `pymongo.operations.SearchIndexModel`) rather than raw dictionaries or manual UI creation. This ensures infrastructure-as-code.
  * **Rule:** Explicitly specify the index `type` (`vectorSearch` or `search`) and `name`.
  * **Best Practice:** For Vector Search, always index "filter" fields to allow for efficient pre-filtering.

<!-- end list -->

```python
from pymongo.operations import SearchIndexModel

# Example: Vector Search Index Definition
vector_model = SearchIndexModel(
    definition={
        "fields": [
            {
                "type": "vector",
                "path": "embedding",
                "numDimensions": 1536,
                "similarity": "cosine"
            },
            # Optimize: Index fields used in filters!
            {"type": "filter", "path": "category"},
            {"type": "filter", "path": "user_id"}
        ]
    },
    name="vector_index",
    type="vectorSearch"
)
```

### B. The "Wait for Ready" Pattern

  * **Rule:** Code that depends on an index immediately after creation (e.g., app startup, integration tests) **MUST** implement a polling mechanism.
  * **Logic:**
    1.  Submit `create_search_index`.
    2.  Poll `$listSearchIndexes` periodically.
    3.  **Crucial Check:** Wait until `queryable == true` AND `status == "READY"`.
    4.  Timeout gracefully if it takes too long.

<!-- end list -->

```python
# Implementation Logic (Generic)
async def wait_for_search_index(collection, index_name):
    start = time.time()
    while time.time() - start < 300: # 5 min timeout
        # Fetch status via aggregation
        cursor = collection.aggregate([{"$listSearchIndexes": {"name": index_name}}])
        
        # Cursor might be empty initially if index creation just started
        async for index_info in cursor:
            # Check both status and queryable flag
            if index_info.get("queryable") is True:
                return True
            elif index_info.get("status") == "FAILED":
                raise Exception(f"Index build failed: {index_info}")
        
        await asyncio.sleep(5)
    raise TimeoutError(f"Index {index_name} not ready in time.")
```

### C. Idempotency & Creation Logic

  * **Rule:** Index creation scripts must be **idempotent**.
  * **Rule:** Handle `OperationFailure` specifically for race conditions (e.g., "IndexAlreadyExists").
  * **Pattern:** The "Check-Compare-Act" loop.

## 3\. Aggregation Pipeline Optimization

**Context:** The order of operations in MongoDB aggregation pipelines significantly impacts performance, especially with Vector Search.

  * **Rule (Ordering):** Specialized operators like `$vectorSearch`, `$search`, and `$geoNear` **MUST** be the very first stage in the pipeline.
  * **Rule (Pre-filtering):** Do not use a separate `$match` stage immediately after `$vectorSearch` to filter results.
      * *Why:* This forces the vector engine to scan irrelevant vectors (ANN search), only to discard them later.
      * *Fix:* Use the dedicated `filter` property *inside* the `$vectorSearch` definition.

<!-- end list -->

```javascript
// ✅ Correct: Filter INSIDE vectorSearch
{
  "$vectorSearch": {
    "index": "vector_index",
    "path": "embedding",
    "queryVector": [...],
    "filter": { "category": "books" } // Efficient pre-filtering
  }
}

// ⚠️ Incorrect: Match AFTER vectorSearch
[
  { "$vectorSearch": { ... } },
  { "$match": { "category": "books" } }
]
```

## 4\. General Implementation Guidelines

### Error Handling

  * **Rule:** Wrap administrative commands (create/drop/update) in `try/except` blocks to handle benign errors.
  * **Targeted Exceptions:** Handle `NamespaceNotFound` (dropping non-existent index) and `CollectionInvalid` (creating existing collection).

### Scoping & Wrapping

  * **Pattern:** Use a wrapper or repository pattern to inject standard filters (e.g., `tenant_id`, `experiment_id`) automatically in multi-tenant applications.
