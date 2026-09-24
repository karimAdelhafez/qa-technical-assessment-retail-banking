# Task 4 — Automation Framework Design & Test Architecture

This blueprint presents a scalable, maintainable test automation architecture for the dynamic **Bank Operations Console**. It prioritizes decoupled layer separation, locator resilience, dynamic DOM state management, self-healing test data isolation, and global scaling pipelines using **Playwright (TypeScript)**.

***Note on Missing Visual Reference:** The file `Bank_Admin_Console_Demo.html` was not provided in the assignment packet. This blueprint is designed abstractly around the semantic DOM constraints specified in the prompt, making it fully resilient against missing visual layouts.*

---

## 🏛️ 1. Project Structure & Page Object Component Design

To prevent the anti-pattern of a monolithic 900-line page object, the framework implements a **Composite Page Component Architecture**. The master page object acts strictly as an entry-point coordinator, while individual console tabs, data builders, and API helpers are isolated into independent, clean modules.

### 📁 Framework Directory Tree Structure
```text
task-4/
├── framework-architecture-design.md
├── config/                          # Centralized runtime engine configurations
│   ├── playwright.config.ts        # Dynamic parallel worker configurations and sharding profiles
│   ├── environment.config.ts       # Centralized target platform routing environments (Staging/UAT)
│   └── auth.config.ts              # Authorization protocols and browser context authentication profiles
└── src/
    ├── api-client/
    │   └── operations-api.client.ts # Handles pre-execution baseline capturing and backend verification
    ├── builders/
    │   ├── config-payload.builder.ts # Fluent factory pattern generating dynamic transaction data
    │   └── user-factory.builder.ts   # Factory pattern serving dynamic admin and staff session profiles
    ├── pages/                      # Page Object Model Layer
    │   ├── base.page.ts            # Houses custom interaction wrappers and dynamic wait mechanisms
    │   ├── bank-operations.page.ts # Master interface page coordinator class
    │   └── components/             # Decoupled tab sub-components (keeps domain classes <100 lines)
    │       ├── onboarding-tab.ts
    │       ├── kyc-thresholds-tab.ts
    │       ├── transfer-limits-tab.ts
    │       ├── fees-tab.ts
    │       ├── notifications-tab.ts
    │       └── audit-settings-tab.ts
    ├── schema/
    │   └── console-state.validator.ts # JSON Schema compiler verifying persistence at the contract level
    └── helpers/
        ├── field-manager.ts        # Polymorphic field controller wrapping ARIA interactions
        ├── security-sanitizer.ts    # Scans response metadata for secure HTTP headers (OWASP baseline)
        └── dynamic-wait.helper.ts   # Custom element hydration checking handlers
```

### 🧱 Architectural Deconstruction
* **What is a Page Object Here?** The **`bank-operations.page.ts`** class serves as the master Page Object (Coordinator). It contains zero selector definitions or field methods for individual tabs. It exposes only global layout elements, such as the navigation tab headers, the central Save configuration button, and page reload triggers.
* **How Tabs and Repeated Field Types Fit:** Each configuration tab is isolated into its own dedicated sub-component class inside the **`components/`** directory (e.g., `fees-tab.ts`). When a test interacts with the console, the master `BankOperationsPage` instantiates these components lazily. Repeated field types do not get hardcoded locators inside these tab classes; they are managed by a centralized, polymorphic locator utility (`BankConsoleFieldManager`) which dynamically identifies the element wrapper based on its stable text label anchor.
* **Avoiding One Giant Class File:** We enforce the **Single Responsibility Principle** and the **Open/Closed Principle**. Splitting the console into isolated tab components ensures that the code for any single business feature remains under 100 lines. If the bank introduces a new tab, an engineer creates a separate component module and registers it as a lazy-loaded property on the master coordinator class. The rest of the core automation codebase remains completely untouched.

---

## 🎯 2. Robust Locator Strategy & Polymorphic Field Abstraction

The console's child form controls (inputs, checkboxes, dropdowns) carry session-dynamic attributes (`slot="field-145"`). Hardcoding or using these dynamic identifiers as primary locators is entirely forbidden.

### ⚖️ Locator Critique: Playwright Global ARIA Roles vs. Scoped Layering
Invoking standard global ARIA roles (e.g., `page.getByRole('textbox', { name: 'Transfer Limits' })`) is usually best practice. However, **it is an anti-pattern in complex dynamic form consoles**. Because multiple tabs reuse identical input layouts and repeated field patterns, hitting global ARIA locators leads to massive selector ambiguity and brittle tests.

### 🛡️ Semantic Parent-to-Child Anchor Strategy
The locator engine bypasses dynamic attributes completely by anchoring onto the unique, immutable outer wrapper label first via text parameters, and then scoping an internal ARIA locator scan downward inside that bounded DOM tree branch to interact safely with the child input control:

```typescript
// src/helpers/field-manager.ts
import { Page } from '@playwright/test';

export interface FieldInteractionControls {
  setFieldValue(labelAnchor: string, inputPayload: string | boolean): Promise<void>;
}

export class BankConsoleFieldManager implements FieldInteractionControls {
  private page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async setFieldValue(labelAnchor: string, inputPayload: string | boolean): Promise<void> {
    // 1. Isolate the immutable, stable parent container wrapper via visible label text
    const parentContainer = this.page.locator('.form-item-wrapper', { hasText: labelAnchor });

    // 2. Resolve polymorphic input controls using scoped ARIA role bindings inside the container branch
    if (typeof inputPayload === 'boolean') {
      // Scopes down directly into the child checkbox element within this wrapper branch only
      await parentContainer.getByRole('checkbox').setChecked(inputPayload);
    } else {
      // Check if a select dropdown or textbox element populates this structural slot at runtime
      const isDropdown = await parentContainer.locator('select').count() > 0;
      if (isDropdown) {
        await parentContainer.getByRole('combobox').selectOption({ label: inputPayload });
      } else {
        await parentContainer.getByRole('textbox').fill(inputPayload);
      }
    }
  }
}
```

---

## ⏳ 3. Handling Dynamic DOM, Rehydration Gates & Waiting Strategy

Switching tabs or triggering deep page scrolls in an enterprise banking console initiates massive DOM rehydration, stale node destructions, and variable layout synchronization intervals.

### 🚫 Defensive Architecture: What NOT to Do
* **Never use static thread sleeps (`page.waitForTimeout(3000)`):** Hardcoded delays pollute compute performance and drastically increase CI run expenses.
* **Never compile static page locators:** Compiling elements inside page class constructors locks stale element references in memory, triggering instant caching errors when tabs tear down and rebuild nodes.

### 🛡️ Safe Execution & Dynamic Web Assertions Strategy
1. **Lazy Evaluation Locators:** All locator references are evaluated dynamically at the exact millisecond of the test interaction block.
2. **Tab-Hydration Gatekeepers:** Transitioning across tabs invokes a dedicated synchronization helper that forces the script engine to gate execution until a distinct, structural element unique to the incoming tab passes an immutable stability check.
3. **Polling Web Assertions:** The validation layer leverages auto-polling, asynchronous web-assertions to inspect live property changes continuously rather than mapping actions to static timers.

```typescript
// src/pages/bank-operations.page.ts
import { expect, Locator } from '@playwright/test';
import { BasePage } from './base.page';

export class BankOperationsPage extends BasePage {
  // Navigation trigger that safely handles tab-swapping DOM rehydration
  async switchToTab(tabHeaderName: string, transitionGateLocator: Locator): Promise<void> {
    // Target the specific header tab by its accessible ARIA role
    await this.page.getByRole('tab', { name: tabHeaderName }).click();
    
    // Action-driven assertion: Web-first assertion auto-polls the DOM until the state settles
    await expect(transitionGateLocator).toBeVisible({ timeout: 5000 });
    await expect(transitionGateLocator).toBeEnabled();
  }
}
```

---

## 📊 4. Test Data Strategy for Shared & Restricted Environments

Because the target test instance is actively shared across concurrent engineering groups, data mutations are highly restricted, and database resets cannot be triggered on demand, the architecture relies on an automated **Self-Healing State Isolation Strategy**.

### 🧩 Core Data Control Patterns
* **Fluent Payload Builders & User Factories:** Hardcoded parameter configurations are strictly banned. Dynamic configurations are handled via a fluent factory builder (`config-payload.builder.ts`), while user account credentials and authorization states are served dynamically via a strict user model builder (`user-factory.builder.ts`).
* **The Multi-Worker Overwrite Risk:** Parallel test executions or concurrent manual operations risk overwriting identical configuration rows, triggering false regression flags.
* **The Solution (Automated Baseline Seeding & Self-Healing Teardown):**


- **Pre-Execution Blueprint Capture:** Before typing values on the UI, the script calls the `operations-api.client.ts` to fetch and store the exact current live configurations of the targeted 12 fields into local execution context variables.
- **Atomic UI Regression Run:** The UI automation completes tab switches, triggers the polymorphic field writes across the 12 locations, saves changes, executes a hard page reload, and validates that values persisted.
- **Guaranteed Automated Teardown:** Regardless of test outcome (Success, Failure, or Timeout Error), an encapsulated `finally`/`teardown` execution wrapper is automatically executed. This block sends an asynchronous `POST`/`PUT` payload via the API client, instantly flashing the 12 fields back to their original baseline positions, keeping the environment clean.

---

## 📈 5. Reusability, Scalability & Enterprise Execution Pipelines

Maintaining execution velocity as the regression test suite scales from 7 tabs to 20 requires complete isolation of testing assets paired with high parallel distribution.


## 🚀 Developer Extension Workflow

### Adding a 20th Field
```typescript
async setMaxTransferLimit(limit: string) {
  await this.fieldManager.setFieldValue('Max Transfer Limit', limit);
}
```
### Adding a 5th Tab
The engineer constructs an independent component module file inside  
`src/pages/components/`  
and instantiates it as a property on the master layout class, guaranteeing zero code duplication or structural regression on the main codebase.

---

## ⏱️ Scale & Pipeline Execution Strategy

### Parallelization & Sharding
Tests are kept **100% atomic** and self‑healing.  
They control their own state transitions and can be safely executed across multi‑core systems using built‑in engine sharding.

### CI/CD Integration & Failure Trace Harvesting
Integrated into GitHub Actions CI workflows running headlessly on scheduled crons or PR triggers.  
The execution profile is configured to record interactive **HTML** traces, full video captures, and browser network log files exclusively on a failure condition, maintaining a clean pipeline storage footprint.

---
## 🛠️ 6. Tooling Stack Selection & Trade-Off Matrix

The advanced automation harness chooses **Playwright (TypeScript)** as the primary UI driver, integrated with **k6** for backend load validation.

### ⚖️ Tooling Evaluation & Accepted Trade-offs

| Tool Chosen | Direct Engineering Benefits | Accepted Architectural Trade-off |
| :--- | :--- | :--- |
| **Playwright (TypeScript)** | Native multi-tab browser context isolation; lightning-fast execution via Chrome DevTools Protocol; built-in dynamic auto-waiting; type safety catches schema compile errors at compile time. | **Bypasses the actual network proxy layer:** Playwright communicates directly with the internal browser rendering engines, meaning it does not catch low-level, vendor-specific network proxy routing quirks as natively as Selenium. |
| **k6 (JavaScript)** | Go-based, low-overhead execution shell; perfect for spinning up deep concurrent traffic stress tests to evaluate microservice locking and transaction race conditions. | **Zero native UI rendering capabilities:** k6 operates strictly at the network protocol layer and cannot execute client-side browser layouts, requiring an independent script directory from the main UI framework. |
---

### 🛠️ Tooling Exclusions
- **Cypress** was actively rejected: Cypress executes directly inside a single browser sandbox, rendering it fundamentally incapable of managing seamless multi‑tab orchestration and sequential browser page swaps, which are mandatory features for this console suite.  
- **Selenium WebDriver** was actively rejected: Selenium lacks out‑of‑the‑box auto‑waiting capabilities and relies heavily on manual explicit wait statements, exponentially increasing technical maintenance overhead across dynamic **DOM** architectures.
