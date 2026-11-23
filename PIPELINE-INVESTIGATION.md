# CI/CD Pipeline Investigation Report

**Repository:** `sakxhamm/ci-reliability-investigation`  
**Assessment:** CI Reliability Investigation Challenge  
**Date:** November 23, 2026  
**Investigator:** DevOps Engineering Team  
**Status:** Completed  

---

## 1. Executive Summary

A comprehensive investigation was conducted on the GitHub Actions continuous integration (CI) pipeline of this repository. The pipeline was observed exhibiting critical failure, instability, and severe security over-privilege. 

Through analysis of the workflow configuration (`.github/workflows/ci.yml`), repository dependencies (`package.json`), unit test suites (`tests/app.test.js`), and actual GitHub Actions execution logs (Run ID: `45160043667`), three core defects and architectural shortcomings were identified:

1. **Fatal CI Script Mismatch (Defect A):** The workflow invoked `npm run test-ci`, a script nonexistent in `package.json`, causing instantaneous build failure (`npm error Missing script: "test-ci"`).
2. **Flaky Dynamic Timestamp Test Assertion (Defect B):** The unit test suite asserted strict equality against a dynamic ISO timestamp (`new Date().toISOString()`), introducing a race condition that causes inevitable failure in CI runners and local environments due to millisecond execution latency.
3. **Overly Permissive Workflow Permissions (Defect C):** The workflow granted `permissions: write-all`, creating a high-severity security vulnerability that exposes repository contents, packages, issues, and deployments to unauthorized modification if untrusted code or dependencies execute within the runner.
4. **Absence of Staged Quality Gates (Defect D):** Linting scripts defined in `package.json` (`npm run lint`) and `.eslintrc.json` were omitted from the CI pipeline, and the pipeline lacked multi-job dependency chaining (`needs:`).

This document details the forensic evidence, root causes, reliability impact, and engineering solutions to remediate these issues.

---

## 2. Issues Identified

### Issue 1: Missing CI Test Command in `package.json`
* **File Affected:** `.github/workflows/ci.yml` (Line 33) & `package.json` (Lines 6–10)
* **Severity:** Critical (Blocks entire CI pipeline)
* **Symptom:** Pipeline fails at step `Run Test Suite` with exit code 1.

### Issue 2: Flaky Timestamp Race Condition in Test Suite
* **File Affected:** `tests/app.test.js` (Lines 13–19)
* **Severity:** High (Intermittent / Consistent test failure, false-negative CI status)
* **Symptom:** Unit test fails when asserting that `res.body.timestamp` strictly equals `new Date().toISOString()`.

### Issue 3: Overly Permissive Runner Security Scopes
* **File Affected:** `.github/workflows/ci.yml` (Line 10)
* **Severity:** High (Security & Compliance Risk)
* **Symptom:** Workflow requests `permissions: write-all`, granting elevated privileges across all scopes of the GitHub token.

### Issue 4: Incomplete Pipeline Validation & Missing Quality Gates
* **File Affected:** `.github/workflows/ci.yml`
* **Severity:** Medium (Code Quality & Maintainability Risk)
* **Symptom:** Static code analysis (`npm run lint`) is not run during CI, allowing syntax and stylistic regressions to merge undetected.

---

## 3. Evidence from Workflow Configurations and Execution Logs

### 3.1 GitHub Actions Execution Failure (Run ID: 45160043667)
Inspection of GitHub Actions run `45160043667` triggered by commit `63e43e5` revealed the following fatal failure in job `Build and Test Application` (Job ID: `137642824175`):

```text
2026-11-23T10:52:56.8645870Z ##[group]Run npm run test-ci
2026-11-23T10:52:56.8646239Z npm run test-ci
2026-11-23T10:52:56.8679108Z shell: /usr/bin/bash -e {0}
2026-11-23T10:52:56.8679446Z ##[endgroup]
2026-11-23T10:52:57.0083053Z npm error Missing script: "test-ci"
2026-11-23T10:52:57.0083733Z npm error
2026-11-23T10:52:57.0084144Z npm error To see a list of scripts, run:
2026-11-23T10:52:57.0084550Z npm error   npm run
2026-11-23T10:52:57.0101455Z npm error A complete log of this run can be found in: /home/runner/.npm/_logs/2026-11-23T10_52_56_945Z-debug-0.log
2026-11-23T10:52:57.0146435Z ##[error]Process completed with exit code 1.
```

### 3.2 Comparison with `package.json`
Inspection of `package.json` confirms the available scripts:

```json
"scripts": {
  "start": "node src/app.js",
  "test": "jest",
  "lint": "eslint src/**/*.js"
}
```
There is no `test-ci` target defined. The workflow in `.github/workflows/ci.yml` attempts:
```yaml
- name: Run Test Suite
  run: npm run test-ci
```

### 3.3 Test Suite Flakiness Reproduction Evidence
Running the unit test suite locally (`npm test`) reveals an immediate failure due to dynamic timestamp evaluation:

```text
FAIL tests/app.test.js
  GET /api/status
    × should return a healthy status with a timestamp (37 ms)

  ● GET /api/status › should return a healthy status with a timestamp

    expect(received).toBe(expected) // Object.is equality

    Expected: "2026-11-23T10:55:31.367Z"
    Received: "2026-11-23T10:55:31.359Z"

      16 |     // this test will fail in environments with even slight latency (like GitHub Actions runners).
      17 |     const expectedTime = new Date().toISOString();
    > 18 |     expect(res.body.timestamp).toBe(expectedTime);
         |                                ^
      19 |   });
      20 | });

      at Object.toBe (tests/app.test.js:18:32)
```

The difference of 8 milliseconds (`10:55:31.359Z` vs `10:55:31.367Z`) between the HTTP response handler instantiation and test assertion causes `expect().toBe()` to fail.

### 3.4 Workflow Security Scope Evidence
In `.github/workflows/ci.yml`, the top-level permissions declaration reads:
```yaml
permissions: write-all
```
This is explicitly marked in the code comments as an intentional security defect:
`# SECURITY CONCERN: Overly permissive permissions (should be restricted to contents: read)`

---

## 4. Root Cause Analysis

### 4.1 Defect A: Missing Script Target in npm Lifecycle
* **Mechanism:** When `npm run <script>` is called, npm parses the `scripts` object in `package.json`. If the requested key does not exist, npm emits `npm error Missing script` and terminates with exit code 1.
* **Why it occurred:** The workflow author assumed `test-ci` was configured for headless CI execution, but only standard `test` was created.

### 4.2 Defect B: Non-Deterministic Assertion on Asynchronous Clock
* **Mechanism:** In `src/app.js`, the Express route `/api/status` calls `formatResponse({ status: 'healthy' })`. In `src/utils.js`, `formatResponse` dynamically stamps the current time using `timestamp: new Date().toISOString()`. In `tests/app.test.js`, the test makes an HTTP request via Supertest, awaits the response, and then evaluates `new Date().toISOString()`.
* **Why it occurred:** Physical execution time elapses between the request processing and assertion. Millisecond precision guarantees that `received !== expected` in almost 100% of runs, especially on virtualized CI runners with variable CPU scheduling.

### 4.3 Defect C: Violation of Principle of Least Privilege (PoLP)
* **Mechanism:** `permissions: write-all` elevates the implicit `GITHUB_TOKEN` secret to have write access across all GitHub API resources (issues, pull requests, contents, deployments, actions, packages).
* **Why it matters:** If an attacker submits a PR or compromises a third-party npm dependency that executes during `npm ci` or `npm test`, malicious code could use the runner's GitHub token to push backdoors to branches, overwrite releases, or access organizational repositories.

---

## 5. Impact on Pipeline Reliability and Developer Velocity

| Factor | Unfixed State | Impact on Team & Operations |
| :--- | :--- | :--- |
| **Build Stability** | 100% Failure Rate | Developers cannot merge code; continuous deployment is halted. |
| **Test Determinism** | Flaky / Non-deterministic | Even if the script command were fixed, tests would randomly fail due to clock jitter, eroding confidence in test results ("alert fatigue"). |
| **Security Posture** | Elevated Risk | Any dependency script or workflow action could compromise the repository. |
| **Feedback Loop** | Broken Quality Gates | Code styling or linting bugs bypass CI and only surface post-merge. |

---

## 6. Recommended Engineering Solutions

1. **Correct Test Invocation:**  
   Update `.github/workflows/ci.yml` to execute `npm test` instead of `npm run test-ci`.
2. **Refactor Test Assertion for Temporal Invariance:**  
   In `tests/app.test.js`, verify the structure, existence, and validity of the ISO date string rather than testing exact millisecond equality:
   ```javascript
   expect(res.body.timestamp).toBeDefined();
   expect(new Date(res.body.timestamp).getTime()).not.toBeNaN();
   ```
3. **Enforce Least Privilege Permissions:**  
   Restrict top-level workflow permissions to `contents: read`:
   ```yaml
   permissions:
     contents: read
   ```
4. **Implement Staged Multi-Job Validation:**  
   Decouple validation into two distinct, sequential jobs:
   - `lint` (Code Quality Linting) executing `npm run lint`
   - `test` (Run Test Suite) executing `npm test`, conditioned with `needs: lint`
   This guarantees that syntax and linting checks run first, saving compute time if code hygiene is violated before spinning up testing.

---

## 7. Expected Improvements After Implementation

* **100% Deterministic CI:** Pipeline succeeds reliably on every valid commit without timing flakiness.
* **Defense in Depth:** The runner token is read-only, drastically mitigating supply chain attack vectors.
* **Separation of Concerns:** Distinct `lint` and `test` jobs provide granular visibility into failures on GitHub Actions UI.
* **Engineering Standard Compliance:** Fully adheres to GitHub Actions Best Practices and the project's assessment requirements.
