---
title: Subagent Architecture
description: Parallel execution pattern for TEA workflows
---

# Subagent Architecture for TEA Workflows

**Version**: 1.0
**Date**: 2026-01-27
**Status**: Implementation Guide

---

## Overview

TEA workflows use **subagent patterns** to parallelize independent tasks, improving performance and maintaining clean separation of concerns. Five workflows benefit from this architecture:

1. **automate** - Parallel test generation (API + E2E)
2. **atdd** - Parallel failing test generation (API + E2E)
3. **test-review** - Parallel quality dimension checks
4. **nfr-assess** - Parallel NFR domain assessments
5. **trace** - Two-phase workflow separation

---

## Core Subagent Pattern

### Architecture

```
Main Workflow (Orchestrator)
├── Step 1: Setup & Context Loading
├── Step 2: Launch Subagents
│   ├── Subagent A → temp-file-a.json
│   ├── Subagent B → temp-file-b.json
│   ├── Subagent C → temp-file-c.json
│   └── (All run in parallel, isolated 200k containers)
└── Step 3: Aggregate Results
    ├── Read all temp files
    ├── Merge/synthesize outputs
    └── Generate final artifact
```

### Key Principles

1. **Independence**: Each subagent is completely independent (no shared state)
2. **Isolation**: Each subagent runs in separate 200k context container
3. **Output Format**: All subagents output structured JSON to temp files
4. **Aggregation**: Main workflow reads temp files and synthesizes final output
5. **Error Handling**: Each subagent reports success/failure in JSON output

---

## Workflow-Specific Designs

### 1. automate - Parallel Test Generation

**Goal**: Generate API and E2E tests in parallel

#### Architecture

```
automate workflow
├── Step 1: Analyze codebase & identify features
├── Step 2: Load relevant knowledge fragments
├── Step 3: Launch parallel test generation
│   ├── Subagent A: Generate API tests → /tmp/api-tests-{timestamp}.json
│   └── Subagent B: Generate E2E tests → /tmp/e2e-tests-{timestamp}.json
├── Step 4: Aggregate tests
│   ├── Read API tests JSON
│   ├── Read E2E tests JSON
│   └── Generate fixtures (if needed)
├── Step 5: Verify all tests pass
└── Step 6: Generate DoD summary
```

#### Subagent A: API Tests

**Input** (passed via temp file):

```json
{
  "features": ["feature1", "feature2"],
  "knowledge_fragments": ["api-request", "data-factories"],
  "config": {
    "use_playwright_utils": true,
    "framework": "playwright"
  }
}
```

**Output** (`/tmp/api-tests-{timestamp}.json`):

```json
{
  "success": true,
  "tests": [
    {
      "file": "tests/api/feature1.spec.ts",
      "content": "import { test, expect } from '@playwright/test';\n...",
      "description": "API tests for feature1"
    }
  ],
  "fixtures": [],
  "summary": "Generated 5 API test cases"
}
```

#### Subagent B: E2E Tests

**Input** (passed via temp file):

```json
{
  "features": ["feature1", "feature2"],
  "knowledge_fragments": ["fixture-architecture", "network-first"],
  "config": {
    "use_playwright_utils": true,
    "framework": "playwright"
  }
}
```

**Output** (`/tmp/e2e-tests-{timestamp}.json`):

```json
{
  "success": true,
  "tests": [
    {
      "file": "tests/e2e/feature1.spec.ts",
      "content": "import { test, expect } from '@playwright/test';\n...",
      "description": "E2E tests for feature1 user journey"
    }
  ],
  "fixtures": ["authFixture", "dataFixture"],
  "summary": "Generated 8 E2E test cases"
}
```

#### Step 4: Aggregation Logic

```javascript
// Read both subagent outputs
const apiTests = JSON.parse(fs.readFileSync('/tmp/api-tests-{timestamp}.json', 'utf8'));
const e2eTests = JSON.parse(fs.readFileSync('/tmp/e2e-tests-{timestamp}.json', 'utf8'));

// Merge test suites
const allTests = [...apiTests.tests, ...e2eTests.tests];

// Collect unique fixtures
const allFixtures = [...new Set([...apiTests.fixtures, ...e2eTests.fixtures])];

// Generate combined DoD summary
const summary = {
  total_tests: allTests.length,
  api_tests: apiTests.tests.length,
  e2e_tests: e2eTests.tests.length,
  fixtures: allFixtures,
  status: apiTests.success && e2eTests.success ? 'PASS' : 'FAIL',
};
```

---

### 2. atdd - Parallel Failing Test Generation

**Goal**: Generate failing API and E2E tests in parallel (TDD red phase)

#### Architecture

```
atdd workflow
├── Step 1: Load story acceptance criteria
├── Step 2: Load relevant knowledge fragments
├── Step 3: Launch parallel test generation
│   ├── Subagent A: Generate failing API tests → /tmp/atdd-api-{timestamp}.json
│   └── Subagent B: Generate failing E2E tests → /tmp/atdd-e2e-{timestamp}.json
├── Step 4: Aggregate tests
├── Step 5: Verify tests fail (red phase)
└── Step 6: Output ATDD checklist
```

**Key Difference from automate**: Tests must be written to **fail** before implementation exists.

#### Subagent Outputs

Same JSON structure as automate, but:

- Tests include failing assertions (e.g., `expect(response.status).toBe(200)` when endpoint doesn't exist yet)
- Summary includes: `"expected_to_fail": true`

---

### 3. test-review - Parallel Quality Dimension Checks

**Goal**: Run independent quality checks in parallel, aggregate into 0-100 score

#### Architecture

```
test-review workflow
├── Step 1: Load test files & context
├── Step 2: Launch parallel quality checks
│   ├── Subagent A: Determinism check → /tmp/determinism-{timestamp}.json
│   ├── Subagent B: Isolation check → /tmp/isolation-{timestamp}.json
│   ├── Subagent C: Maintainability check → /tmp/maintainability-{timestamp}.json
│   ├── Subagent D: Coverage check → /tmp/coverage-{timestamp}.json
│   └── Subagent E: Performance check → /tmp/performance-{timestamp}.json
└── Step 3: Aggregate findings
    ├── Calculate weighted score (0-100)
    ├── Synthesize violations
    └── Generate review report with suggestions
```

#### Subagent Output Format

Each quality dimension subagent outputs:

```json
{
  "dimension": "determinism",
  "score": 85,
  "max_score": 100,
  "violations": [
    {
      "file": "tests/api/user.spec.ts",
      "line": 42,
      "severity": "HIGH",
      "description": "Test uses Math.random() - non-deterministic",
      "suggestion": "Use faker with fixed seed"
    }
  ],
  "passed_checks": 12,
  "failed_checks": 3,
  "summary": "Tests are mostly deterministic with 3 violations"
}
```

#### Step 3: Aggregation Logic

```javascript
// Read all dimension outputs
const dimensions = ['determinism', 'isolation', 'maintainability', 'coverage', 'performance'];
const results = dimensions.map((d) => JSON.parse(fs.readFileSync(`/tmp/${d}-{timestamp}.json`, 'utf8')));

// Calculate weighted score
const weights = { determinism: 0.25, isolation: 0.25, maintainability: 0.2, coverage: 0.15, performance: 0.15 };
const totalScore = results.reduce((sum, r) => sum + r.score * weights[r.dimension], 0);

// Aggregate violations by severity
const allViolations = results.flatMap((r) => r.violations);
const highSeverity = allViolations.filter((v) => v.severity === 'HIGH');
const mediumSeverity = allViolations.filter((v) => v.severity === 'MEDIUM');
const lowSeverity = allViolations.filter((v) => v.severity === 'LOW');

// Generate final report
const report = {
  overall_score: Math.round(totalScore),
  grade: getGrade(totalScore), // A/B/C/D/F
  dimensions: results,
  violations_summary: {
    high: highSeverity.length,
    medium: mediumSeverity.length,
    low: lowSeverity.length,
    total: allViolations.length,
  },
  top_suggestions: prioritizeSuggestions(allViolations),
};
```

---

### 4. nfr-assess - Parallel NFR Domain Assessments

**Goal**: Assess independent NFR domains in parallel

#### Architecture

```
nfr-assess workflow
├── Step 1: Load system context
├── Step 2: Launch parallel NFR assessments
│   ├── Subagent A: Security assessment → /tmp/nfr-security-{timestamp}.json
│   ├── Subagent B: Performance assessment → /tmp/nfr-performance-{timestamp}.json
│   ├── Subagent C: Reliability assessment → /tmp/nfr-reliability-{timestamp}.json
│   └── Subagent D: Scalability assessment → /tmp/nfr-scalability-{timestamp}.json
└── Step 3: Aggregate NFR report
    ├── Synthesize domain assessments
    ├── Identify cross-domain risks
    └── Generate compliance documentation
```

#### Subagent Output Format

Each NFR domain subagent outputs:

```json
{
  "domain": "security",
  "risk_level": "MEDIUM",
  "findings": [
    {
      "category": "Authentication",
      "status": "PASS",
      "description": "OAuth2 with JWT tokens implemented",
      "recommendations": []
    },
    {
      "category": "Data Encryption",
      "status": "CONCERN",
      "description": "Database encryption at rest not enabled",
      "recommendations": ["Enable database encryption", "Use AWS KMS for key management"]
    }
  ],
  "compliance": {
    "SOC2": "PARTIAL",
    "GDPR": "PASS",
    "HIPAA": "N/A"
  },
  "priority_actions": ["Enable database encryption within 30 days"]
}
```

#### Step 3: Aggregation Logic

```javascript
// Read all NFR domain outputs
const domains = ['security', 'performance', 'reliability', 'scalability'];
const assessments = domains.map((d) => JSON.parse(fs.readFileSync(`/tmp/nfr-${d}-{timestamp}.json`, 'utf8')));

// Calculate overall risk
const riskLevels = { HIGH: 3, MEDIUM: 2, LOW: 1, NONE: 0 };
const maxRiskLevel = Math.max(...assessments.map((a) => riskLevels[a.risk_level]));
const overallRisk = Object.keys(riskLevels).find((k) => riskLevels[k] === maxRiskLevel);

// Aggregate compliance status
const allCompliance = assessments.flatMap((a) => Object.entries(a.compliance));
const complianceSummary = {};
allCompliance.forEach(([std, status]) => {
  if (!complianceSummary[std]) complianceSummary[std] = [];
  complianceSummary[std].push(status);
});

// Synthesize cross-domain risks
const crossDomainRisks = identifyCrossDomainRisks(assessments); // e.g., "Performance + scalability concern"

// Generate final report
const report = {
  overall_risk: overallRisk,
  domains: assessments,
  compliance_summary: complianceSummary,
  cross_domain_risks: crossDomainRisks,
  priority_actions: assessments.flatMap((a) => a.priority_actions),
  executive_summary: generateExecutiveSummary(assessments),
};
```

---

### 5. trace - Two-Phase Workflow Separation

**Goal**: Clean separation of coverage matrix generation and gate decision

#### Architecture

```
trace workflow
├── Phase 1: Coverage Matrix
│   ├── Step 1: Load requirements
│   ├── Step 2: Analyze test suite
│   └── Step 3: Generate traceability matrix → /tmp/trace-matrix-{timestamp}.json
└── Phase 2: Gate Decision (depends on Phase 1 output)
    ├── Step 4: Read coverage matrix
    ├── Step 5: Apply decision tree logic
    ├── Step 6: Calculate coverage percentages
    └── Step 7: Generate gate decision (PASS/CONCERNS/FAIL/WAIVED)
```

**Note**: This isn't parallel subagents, but subagent-like **phase separation** where Phase 2 depends on Phase 1 output.

#### Phase 1 Output Format

```json
{
  "requirements": [
    {
      "id": "REQ-001",
      "description": "User can login with email/password",
      "priority": "P0",
      "tests": ["tests/auth/login.spec.ts::should login with valid credentials"],
      "coverage": "FULL"
    },
    {
      "id": "REQ-002",
      "description": "User can reset password",
      "priority": "P1",
      "tests": [],
      "coverage": "NONE"
    }
  ],
  "total_requirements": 50,
  "covered_requirements": 42,
  "coverage_percentage": 84
}
```

#### Phase 2: Gate Decision Logic

```javascript
// Read Phase 1 output
const matrix = JSON.parse(fs.readFileSync('/tmp/trace-matrix-{timestamp}.json', 'utf8'));

// Apply decision tree
const p0Coverage = matrix.requirements.filter((r) => r.priority === 'P0' && r.coverage === 'FULL').length;
const totalP0 = matrix.requirements.filter((r) => r.priority === 'P0').length;

let gateDecision;
if (p0Coverage === totalP0 && matrix.coverage_percentage >= 90) {
  gateDecision = 'PASS';
} else if (p0Coverage === totalP0 && matrix.coverage_percentage >= 75) {
  gateDecision = 'CONCERNS';
} else if (p0Coverage < totalP0) {
  gateDecision = 'FAIL';
} else {
  gateDecision = 'WAIVED'; // Manual review required
}

// Generate gate report
const report = {
  decision: gateDecision,
  coverage_matrix: matrix,
  p0_coverage: `${p0Coverage}/${totalP0}`,
  overall_coverage: `${matrix.coverage_percentage}%`,
  recommendations: generateRecommendations(matrix, gateDecision),
  uncovered_requirements: matrix.requirements.filter((r) => r.coverage === 'NONE'),
};
```

---

## Implementation Guidelines

### Temp File Management

**Naming Convention**:

```
/tmp/{workflow}-{subagent-name}-{timestamp}.json
```

**Examples**:

- `/tmp/automate-api-tests-20260127-143022.json`
- `/tmp/test-review-determinism-20260127-143022.json`
- `/tmp/nfr-security-20260127-143022.json`

**Cleanup**:

- Temp files should be cleaned up after successful aggregation
- Keep temp files on error for debugging
- Implement retry logic for temp file reads (race conditions)

### Error Handling

Each subagent JSON output must include:

```json
{
  "success": true|false,
  "error": "Error message if failed",
  "data": { ... }
}
```

Main workflow aggregation step must:

1. Check `success` field for each subagent
2. If any subagent failed, aggregate error messages
3. Decide whether to continue (partial success) or fail (critical subagent failed)

### Performance Considerations

**Subagent Isolation**:

- Each subagent runs in separate 200k context container
- No shared memory or state
- Communication only via JSON files

**Parallelization**:

- Resolve execution mode via config (`tea_execution_mode`, `tea_capability_probe`)
- Probe runtime support for agent-team and subagent launch before dispatch
- Fallback order in `auto` mode: `agent-team` → `subagent` → `sequential`
- Ensure temp file paths are unique (timestamp-based)
- Implement proper synchronization (wait for all subagents to complete)

---

## Testing Subagent Workflows

### Test Checklist

For each workflow with subagents:

- [ ] **Unit Test**: Test each subagent in isolation
  - Provide mock input JSON
  - Verify output JSON structure
  - Test error scenarios

- [ ] **Integration Test**: Test full workflow
  - Launch all subagents
  - Verify parallel execution
  - Verify aggregation logic
  - Test with real project data

- [ ] **Performance Test**: Measure speedup
  - Benchmark sequential vs parallel
  - Measure subagent overhead
  - Verify memory usage acceptable

- [ ] **Error Handling Test**: Test failure scenarios
  - One subagent fails
  - Multiple subagents fail
  - Temp file read/write errors
  - Timeout scenarios

### Expected Performance Gains

**automate**:

- Sequential: ~5-10 minutes (API then E2E)
- Parallel: ~3-6 minutes (both at once)
- **Speedup: ~40-50%**

**test-review**:

- Sequential: ~3-5 minutes (5 quality checks)
- Parallel: ~1-2 minutes (all checks at once)
- **Speedup: ~60-70%**

**nfr-assess**:

- Sequential: ~8-12 minutes (4 NFR domains)
- Parallel: ~3-5 minutes (all domains at once)
- **Speedup: ~60-70%**

---

## Documentation for Users

Users don't need to know about subagent implementation details, but they should know:

1. **Performance**: Certain workflows are optimized for parallel execution
2. **Temp Files**: Workflows create temporary files during execution (cleaned up automatically)
3. **Progress**: When running workflows, they may see multiple "subagent" indicators
4. **Debugging**: If workflow fails, temp files may be preserved for troubleshooting

---

## Future Enhancements

1. **Subagent Pooling**: Reuse subagent containers for multiple operations
2. **Adaptive Parallelization**: Dynamically decide whether to parallelize based on workload
3. **Progress Reporting**: Real-time progress updates from each subagent
4. **Caching**: Cache subagent outputs for identical inputs (idempotent operations)
5. **Distributed Execution**: Run subagents on different machines for massive parallelization

---

## References

- BMad Builder subagent examples: `_bmad/bmb/workflows/*/subagent-*.md`
- Runtime-specific agent/subagent documentation (Codex, Claude Code, etc.)
- TEA Workflow validation reports (proof of 100% compliance)

---

**Status**: Ready for implementation across 5 workflows
**Next Steps**: Implement subagent patterns in workflow step files, test, document
