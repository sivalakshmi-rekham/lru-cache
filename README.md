# lru-cache

This repository implements an LRU (Least Recently Used) cache in Java.

## Table of contents

- What is an LRU Cache?
- Why use an LRU Cache?
- Design goals and constraints
- High-level system design
- Data structures & algorithm
- Java implementation
- Complexity
- Concurrency considerations
- Testing & validation
- Extensions & improvements
- Flow diagrams (Mermaid)

## What is an LRU Cache?

An LRU cache is a fixed-size cache that evicts the least recently used item when capacity is exceeded. It provides fast access and bounded memory usage while keeping recently accessed items available.

## Why use an LRU Cache?

- Improve performance by caching expensive or frequently accessed results (DB rows, computed values, disk reads).
- Bounded memory usage with a deterministic eviction policy.
- Works well when recent accesses are more likely to be accessed again (temporal locality).

## Design goals and constraints

- O(1) average time complexity for get(key) and put(key, value).
- Bounded memory: fixed capacity.
- Deterministic eviction using LRU policy.
- Clear API for get/put/remove/clear/size.
- Safe for single-threaded use by default; optionally thread-safe for concurrent environments.

## High-level system design

The design uses two synchronized data structures:

1. A HashMap (key -> node) for O(1) lookup of nodes.
2. A Doubly-Linked List to maintain usage order: head is most recently used (MRU), tail is least recently used (LRU).

Rationale: A HashMap gives constant-time lookup, while the linked list preserves the order of recent accesses and supports O(1) insertion and removal when nodes are known.

Operations:

- get(key): If key exists, move node to the head of the list and return its value. Otherwise return null (or Optional.empty(), or a sentinel like -1 depending on the API choice).
- put(key, value): If key exists, update value and move node to head. If not, create node, insert at head. If capacity exceeded, remove tail node and delete from map.
- remove(key): remove node from list and map if present.
- clear(): remove all entries.

## Data structures & algorithm

- Node: stores key, value, prev, next.
- Map: HashMap<K, Node>.
- List head/tail sentinel nodes simplify insertion/removal (avoid null checks).

Eviction: Always remove the tail node (LRU) when size > capacity.

## Java implementation

Below is a complete, idiomatic Java implementation you can drop into src/main/java:

```java
package com.example.cache;

import java.util.HashMap;
import java.util.Map;

public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head; // dummy head
    private final Node<K, V> tail; // dummy tail

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        Node(K k, V v) { key = k; value = v; }
    }

    public LRUCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("Capacity must be > 0");
        this.capacity = capacity;
        this.map = new HashMap<>(capacity);
        head = new Node<>(null, null);
        tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToHead(node);
            return;
        }
        Node<K, V> newNode = new Node<>(key, value);
        map.put(key, newNode);
        addToHead(newNode);
        if (map.size() > capacity) {
            Node<K, V> lru = removeTail();
            if (lru != null && lru.key != null) map.remove(lru.key);
        }
    }

    public V remove(K key) {
        Node<K, V> node = map.remove(key);
        if (node == null) return null;
        removeNode(node);
        return node.value;
    }

    public void clear() {
        head.next = tail;
        tail.prev = head;
        map.clear();
    }

    public int size() {
        return map.size();
    }

    private void addToHead(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }

    private Node<K, V> removeTail() {
        if (tail.prev == head) return null;
        Node<K, V> lru = tail.prev;
        removeNode(lru);
        return lru;
    }
}
```

Usage example:

```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");
cache.get(2); // 2 becomes most recently used
cache.put(4, "four"); // evicts key 1 (least recently used)
```

If you prefer a simpler approach in some cases, Java's LinkedHashMap already implements access-order eviction; you can subclass it and override removeEldestEntry to enforce capacity.

## Complexity

- Time: O(1) average for get and put (hashmap + linked-list operations).
- Space: O(capacity) for map + linked list nodes.

## Concurrency considerations

- Single-threaded: implementation above is sufficient.
- Multi-threaded: options:
  - Use external synchronization: synchronize public methods or use a ReentrantReadWriteLock to allow concurrent reads.
  - Wrap a LinkedHashMap with Collections.synchronizedMap for coarse-grained safety.
  - Use ConcurrentHashMap and implement a segmented LRU or use third-party libraries (Caffeine) for high-performance concurrent caches.

Be careful: making the linked list and map consistent across threads requires careful locking.

## Testing & validation

- Unit tests: basic get/put, eviction order, update existing key, capacity boundary.
- Concurrency tests: multiple threads performing gets/puts; validate no corruption and correctness.
- Property tests: random sequences of operations validated against a slow reference implementation.

## Extensions & improvements

- TTL (time-to-live) expiration for entries.
- Weighted eviction (evict based on entry size instead of count).
- Backing store (write-through/write-back) to persist evicted items.
- Statistics/metrics: hit rate, miss rate, eviction count.

## Flow diagrams (Mermaid)

Get flow:

```mermaid
flowchart TD
  G[Get(key)] --> C{map.containsKey(key)?}
  C -- Yes --> M[move node to head]
  M --> R[return value]
  C -- No --> RN[return null / miss]
```

Put flow:

```mermaid
flowchart TD
  P[Put(key, value)] --> E{map.containsKey(key)?}
  E -- Yes --> U[update value & move to head]
  U --> D[done]
  E -- No --> A[create node & add to head]
  A --> S{map.size() > capacity?}
  S -- Yes --> REM[remove tail node & map.remove(tail.key)] --> D
  S -- No --> D
```

You can paste these Mermaid blocks into GitHub README.md; GitHub will render them as diagrams.

---

What I changed: I replaced the short README with a detailed design-focused README that explains what an LRU cache is, why to use it, design goals, a full Java implementation, usage examples, complexity, concurrency notes, testing ideas, and Mermaid flowcharts.

What's next: I can add a complete Java project skeleton (Maven or Gradle) with the LRUCache implementation under src/main/java and unit tests under src/test/java, or convert the implementation to use LinkedHashMap, or add concurrency-safe variants. Which would you like me to add next?
