# K6 Script Architecture & Modular Design

A well-architected Grafana K6 testing repository treats performance code with the same software engineering standards as production microservices: modularity, reusable components, strict separation of concerns, and clean abstraction boundaries.

---

## 1. Architectural Layers & Separation of Concerns

Monolithic scripts (combining options, config, data generation, HTTP requests, checks, and custom metrics in a single 1,000-line file) become unmaintainable and impossible to reuse across test profiles (smoke, load, stress, spike).

A modern architecture separates the codebase into five distinct layers:

```text
 ┌─────────────────────────────────────────────────────────────┐
 │ 1. Configuration Layer (config/)                            │
 │    - Environment parameters (local, dev, staging, prod)     │
 │    - Dynamic overrides from __ENV (TARGET_URL, K6_ENV)       │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 2. Helper & Utility Layer (helpers/)                        │
 │    - Standardized HTTP client with default headers & trace  │
 │    - Domain-specific synthetic data generators              │
 │    - Centralized custom metrics declarations                │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 3. User Journey / Scenario Flow Layer (scenarios/)          │
 │    - Atomic business transactions (login, browse, order)    │
 │    - Receives client & returns validated domain objects     │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 4. Runnable Test Suites (tests/)                            │
 │    - Defines options: workloads, scenarios, thresholds     │
 │    - Composes journey functions with think times            │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 5. Documentation & Reporting (README.md, reports/)          │
 │    - Dedicated Living Runbook (README.md)                   │
 │    - Automated HTML dashboard & JSON exports                │
 └─────────────────────────────────────────────────────────────┘
```

---

## 2. Dynamic Environment Configuration Pattern

Performance tests frequently need to execute against local Docker containers, development clusters, pre-production staging, or production canary environments.

Never hardcode URLs or API keys. Always use a centralized configuration resolver:

```javascript
// config/environments.js
export const ENVIRONMENTS = {
  local: {
    baseUrl: 'http://localhost:8080',
    timeout: '5s',
  },
  dev: {
    baseUrl: 'https://dev-api.example.com',
    timeout: '10s',
  },
  staging: {
    baseUrl: 'https://staging-api.example.com',
    timeout: '10s',
  },
  prod: {
    baseUrl: 'https://api.example.com',
    timeout: '15s',
  },
};

/**
 * Resolves configuration from environment variables.
 * Precedence: TARGET_URL > K6_ENV preset > default 'local'
 */
export function getConfig() {
  const envKey = __ENV.K6_ENV || 'local';
  const baseConfig = ENVIRONMENTS[envKey] || ENVIRONMENTS.local;

  const targetUrl = __ENV.TARGET_URL || baseConfig.baseUrl;
  const authToken = __ENV.AUTH_TOKEN || null;

  return {
    environment: envKey,
    baseUrl: targetUrl,
    timeout: baseConfig.timeout,
    headers: {
      'Accept': 'application/json',
      ...(authToken ? { 'Authorization': `Bearer ${authToken}` } : {}),
    },
  };
}
```

---

## 3. The User Journey Flow Pattern

A **User Journey** (or scenario flow) is an atomic, reusable business action. Flows take an initialized HTTP client and optional input parameters, execute the HTTP request sequence, validate business invariants, and return domain artifacts for subsequent flows.

### Example: Decoupled Flow Implementations

```javascript
// scenarios/customer-flows.js
import { check } from 'k6';
import { generateCustomerPayload } from '../helpers/data-generator.js';

export function createCustomerFlow(client) {
  const payload = generateCustomerPayload();
  const res = client.post('/api/v1/customers', payload, {
    name: 'POST /api/v1/customers',
  });

  const isSuccess = check(res, {
    'createCustomer: status is 201': (r) => r.status === 201,
    'createCustomer: returns id': (r) => {
      try {
        const body = JSON.parse(r.body);
        return body && Boolean(body.id);
      } catch {
        return false;
      }
    },
  });

  if (!isSuccess) {
    return null;
  }

  return JSON.parse(res.body);
}

export function getCustomerByIdFlow(client, customerId) {
  const res = client.get(`/api/v1/customers/${customerId}`, {
    name: 'GET /api/v1/customers/:id',
  });

  return check(res, {
    'getCustomerById: status is 200': (r) => r.status === 200,
    'getCustomerById: correct id echoed': (r) => {
      try {
        return JSON.parse(r.body).id === customerId;
      } catch {
        return false;
      }
    },
  });
}
```

---

## 4. Assembling Runnable Test Suites

With modular flows, creating different test types (smoke, load, stress) is trivial because the execution logic is shared, while the workload profiles (`options`) vary:

```javascript
// tests/01-smoke.js
import { getConfig } from '../config/environments.js';
import { ResilientHttpClient } from '../helpers/http-client.js';
import { createCustomerFlow, getCustomerByIdFlow } from '../scenarios/customer-flows.js';
import { sleepWithJitter } from '../helpers/pacing.js';

export const options = {
  vus: 1,
  duration: '15s',
  thresholds: {
    http_req_failed: ['rate==0'], // Zero failures tolerated in smoke
    http_req_duration: ['p(95)<300'],
  },
};

const config = getConfig();
const client = new ResilientHttpClient(config);

export default function () {
  const customer = createCustomerFlow(client);
  if (customer) {
    getCustomerByIdFlow(client, customer.id);
  }
  sleepWithJitter(1, 2);
}
```

---

## 5. Summary of Key Benefits

1. **DRY (Don't Repeat Yourself)**: Changing an endpoint path or adding a header requires modifying exactly one file.
2. **Multi-Environment Ready**: Tests run identically on local developer machines, staging, and production canaries via simple CLI environment variables (`-e K6_ENV=staging`).
3. **Composable Workloads**: Different test profiles compose the same business flows with different workload shapes.
