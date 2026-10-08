# K6 Scenarios & Executors Guide

The `scenarios` API is the core engine for scheduling virtual users and modeling complex real-world traffic patterns in Grafana K6.

---

## 1. The 7 K6 Executors Matrix

k6 provides seven distinct executors, each designed for specific mathematical concurrency models:

| Executor | Model | Key Parameters | Best Use Case |
| :--- | :---: | :--- | :--- |
| **`shared-iterations`** | Closed | `vus`, `iterations`, `maxDuration` | Quick fixed-job scripts, database migrations, setup/seeding. |
| **`per-vu-iterations`** | Closed | `vus`, `iterations`, `maxDuration` | Batch tests where every VU must complete identical iteration counts. |
| **`constant-vus`** | Closed | `vus`, `duration` | Simple baseline tests with constant concurrent users. |
| **`ramping-vus`** | Closed | `startVUs`, `stages`, `gracefulRampDown` | Load and stress tests modeling traditional concurrent user ramp-up. |
| **`constant-arrival-rate`** | Open | `rate`, `timeUnit`, `duration`, `preAllocatedVUs`, `maxVUs` | Throughput testing (RPS/TPS) immune to Coordinated Omission. |
| **`ramping-arrival-rate`** | Open | `startRate`, `timeUnit`, `stages`, `preAllocatedVUs`, `maxVUs` | Production traffic modeling with varying requests-per-second stages. |
| **`externally-controlled`** | Either | `vus`, `maxVUs`, `duration` | Interactive CLI tests controlled via k6 REST API during execution. |

---

## 2. Open Model: `constant-arrival-rate` & `ramping-arrival-rate`

### Why Arrival Rate Matters:
In cloud microservices and public web applications, incoming traffic does not stop when the system experiences a slowdown. New requests keep arriving at an external rate determined by users.

If you use VU-based executors (`ramping-vus`):
- A slow server increases iteration latency.
- The VUs spend more time sleeping/waiting for responses.
- Total requests-per-second (RPS) plummets, creating a false impression of stability.

Arrival-rate executors decouple iterations from latency by maintaining the specified rate:

```javascript
// Constant Arrival Rate (fixed RPS throughput)
export const options = {
  scenarios: {
    target_throughput: {
      executor: 'constant-arrival-rate',
      rate: 100, // 100 iterations per second
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 50, // Initial VUs ready to execute
      maxVUs: 200,         // Ceiling of VUs k6 can spawn if server slows down
    },
  },
};
```

```javascript
// Ramping Arrival Rate (open model with progressive RPS stages)
export const options = {
  scenarios: {
    ramping_throughput: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      preAllocatedVUs: 30,
      maxVUs: 150,
      stages: [
        { duration: '2m', target: 50 },  // Ramp up to 50 RPS
        { duration: '5m', target: 100 }, // Sustain peak at 100 RPS
        { duration: '1m', target: 0 },   // Graceful cool down
      ],
    },
  },
};
```

> **Warning**: Always set `maxVUs` appropriately. If the server becomes extremely slow and `maxVUs` is reached, k6 will log:
> `Dropping iterations! Consider increasing maxVUs.`
> This indicates that requests are dropping because the system could not process them within the available VU pool.

---

## 3. Parallel & Staggered Multi-Scenario Orchestration

Scenarios can run in parallel, staggered by `startTime`, or target different entrypoint functions using `exec`:

```javascript
export const options = {
  scenarios: {
    // Scenario 1: Search traffic starts immediately
    search_traffic: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2m', target: 30 },
        { duration: '5m', target: 30 },
        { duration: '1m', target: 0 },
      ],
      exec: 'runSearchJourney',
      tags: { scenario: 'search' },
    },

    // Scenario 2: Checkout traffic starts after a 1-minute warm-up delay
    checkout_traffic: {
      executor: 'constant-arrival-rate',
      startTime: '1m', // Delayed start
      rate: 5,
      timeUnit: '1s',
      duration: '6m',
      preAllocatedVUs: 10,
      maxVUs: 40,
      exec: 'runCheckoutJourney',
      tags: { scenario: 'checkout' },
    },
  },
};

// Target entrypoint for scenario 1
export function runSearchJourney() {
  // ...
}

// Target entrypoint for scenario 2
export function runCheckoutJourney() {
  // ...
}
```

---

## 4. Graceful Ramp-Down & Graceful Stop

When a stage ends or a test finishes, abruptly killing in-flight requests corrupts error rates:

```javascript
export const options = {
  scenarios: {
    order_flow: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 50 },
        { duration: '1m', target: 0 },
      ],
      gracefulRampDown: '30s', // Gives active VUs 30s to finish during ramp-down
      gracefulStop: '30s',     // Gives active VUs 30s to finish when scenario completes
    },
  },
};
```
