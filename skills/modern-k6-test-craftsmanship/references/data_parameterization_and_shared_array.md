# Data Parameterization & SharedArray Memory Hygiene

A common failure mode in large-scale performance tests is running out of memory on the load generator node before the target system even reaches saturation. This is almost always caused by inefficient test data parameterization.

---

## 1. Synthetic Data Generation vs. Seeded Fixtures

Performance testing requires two distinct data strategies depending on the scenario:

| Strategy | When to Use | Trade-offs & Implementation |
| :--- | :--- | :--- |
| **Dynamic Synthetic Data** | Write/Creation flows (`POST`, `PUT`). Ensures unique database records and avoids unnatural caching layers. | Generated dynamically on each VU iteration using pure JS functions (UUIDs, random names, timestamps). Negligible memory footprint. |
| **Seeded Fixtures (`SharedArray`)** | Read flows (`GET`), authentication flows (`POST /login`). Pre-existing database entities that must exist prior to testing. | Pre-loaded from CSV/JSON into read-only memory using `SharedArray`. Avoids file re-reading. |

### Anti-Pattern: Reusing Static Records for Write Tests
If 100 concurrent VUs submit `POST /orders` using the exact same static customer ID and SKU:
- The target database engine may suffer false lock contention or unique constraint violations.
- In-memory caching layers (Redis, Varnish, Hibernate L2 cache) will artificially return responses in 1ms without touching real I/O.
- Synthetic data generators eliminate this bias.

---

## 2. Dynamic Synthetic Data Generator Example

Implement lightweight, deterministic data generators without importing heavy external NPM libraries:

```javascript
// helpers/data-generator.js

export function randomString(length = 8) {
  const chars = 'abcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

export function randomIntBetween(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

export function generateCustomerPayload() {
  const idSuffix = `${Date.now()}-${randomString(4)}`;
  return {
    firstName: `User-${randomString(4)}`,
    lastName: 'LoadTest',
    email: `loadtest-${idSuffix}@example.internal`,
    tier: randomIntBetween(1, 3),
  };
}
```

---

## 3. The `SharedArray` Memory Hygiene Architecture

When loading large data sets from files (e.g. 50,000 pre-provisioned user credentials in `users.json`), standard JavaScript imports or `JSON.parse(open('users.json'))` duplicate the entire memory structure across every single virtual user.

### Memory Comparison:

```text
Without SharedArray (500 VUs, 10MB JSON):
VU 1:  [10MB JSON]
VU 2:  [10MB JSON]
...
VU 500: [10MB JSON]
TOTAL RAM: ~5,000 MB (5 GB) 💥 (Load generator runs out of memory)

With SharedArray:
Init Context: [10MB JSON in Shared Memory (Read-Only)]
VU 1 ──┐
VU 2 ──┼───> Shared Reference
...   ──┤
VU 500 ┘
TOTAL RAM: ~15 MB ✅
```

---

## 4. `SharedArray` Implementation Patterns

### Pattern A: Loading JSON Fixtures
```javascript
import { SharedArray } from 'k6/data';

export const userCatalog = new SharedArray('users-data', function () {
  // Executed ONCE during init stage
  const rawContent = open('../data/users.json');
  return JSON.parse(rawContent);
});

export default function () {
  // Strategy 1: Random sampling
  const randomUser = userCatalog[Math.floor(Math.random() * userCatalog.length)];

  // Strategy 2: Guaranteed unique slice per VU
  // Useful when each VU needs its own dedicated user account
  const assignedUser = userCatalog[__VU % userCatalog.length];
}
```

### Pattern B: Loading CSV Fixtures
k6 includes lightweight string utilities. For CSV parsing without extra dependencies:

```javascript
import { SharedArray } from 'k6/data';

export const accounts = new SharedArray('accounts-csv', function () {
  const file = open('../data/accounts.csv');
  const lines = file.split('\n').filter((line) => line.trim().length > 0);
  
  // Skip header line
  return lines.slice(1).map((line) => {
    const [accountId, apiKey, tier] = line.split(',');
    return { accountId: accountId.trim(), apiKey: apiKey.trim(), tier: tier.trim() };
  });
});
```

---

## 5. Parameterization Best Practices Checklist

1. **Keep SharedArray Read-Only**: Virtual Users must treat `SharedArray` elements as immutable. Do not mutate objects in the array.
2. **Pre-seed in Staging/Dev**: Ensure sufficient database records exist in the target environment matching the size of your fixture dataset.
3. **Partition Test Data by Environment**: Maintain separate data fixtures for staging vs local vs canary environments to avoid cross-environment conflicts.
