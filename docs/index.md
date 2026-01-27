---
title: Test Architect (TEA)
description: Risk-based testing workflows, automation guidance, and release gates for BMad Method
template: splash
hero:
  title: Test Architect (TEA)
  tagline: Risk-based testing workflows, automation guidance, and release gates for BMad Method
  actions:
    - text: Quick Start
      link: https://bmad-code-org.github.io/tutorials/tea-lite-quickstart/
      icon: right-arrow
      variant: primary
    - text: View on GitHub
      link: https://github.com/bmad-code-org/bmad-method-test-architecture-enterprise
      icon: external
---

## What is TEA?

TEA (Test Architect) is a BMAD module for testing strategy and automation. It provides eight workflows covering setup, design, automation, review, and release gates.

## Why TEA?

- **Risk-Based Testing**: P0-P3 prioritization based on probability and impact
- **Consistent Outputs**: Knowledge-base guidance keeps standards consistent
- **Release Gates**: Clear go/no-go decisions with traceability

## Core Workflows

| Workflow                                                  | Trigger | Purpose                                |
| --------------------------------------------------------- | ------- | -------------------------------------- |
| [Framework Setup](/how-to/workflows/setup-test-framework) | TF      | Scaffold test framework                |
| [CI/CD Integration](/how-to/workflows/setup-ci)           | CI      | Set up quality pipeline                |
| [Test Design](/how-to/workflows/run-test-design)          | TD      | Risk-based test planning               |
| [ATDD](/how-to/workflows/run-atdd)                        | AT      | Failing acceptance tests (TDD)         |
| [Test Automation](/how-to/workflows/run-automate)         | TA      | Expand automation coverage             |
| [Test Review](/how-to/workflows/run-test-review)          | RV      | Quality audit with scoring             |
| [Requirements Tracing](/how-to/workflows/run-trace)       | TR      | Coverage mapping + gate decision       |
| [NFR Assessment](/how-to/workflows/run-nfr-assess)        | NR      | Non-functional requirements evaluation |

## Getting Started

Pick a path:

- **TEA Lite**: Start with [Test Automation](/how-to/workflows/run-automate) only (30 minutes)
- **Full TEA**: Complete the [Quick Start Tutorial](https://bmad-code-org.github.io/tutorials/tea-lite-quickstart/) (1-2 hours)
- **Enterprise**: Integrate all 8 workflows into your development process

## Quick Install

```bash
npx bmad-method install
# Select: Test Architect (TEA)
```

Then trigger workflows via chat:

```
tea         # Load TEA agent
test-design # Run Test Design workflow
```

## Learn More

<div class="sl-flex">
  <a href="https://bmad-code-org.github.io/tutorials/tea-lite-quickstart/" class="action primary">
    Quick Start Tutorial →
  </a>
  <a href="/explanation/tea-overview/" class="action secondary">
    How TEA Works →
  </a>
</div>

## Documentation Structure

- **[Tutorials](https://bmad-code-org.github.io/tutorials/tea-lite-quickstart/)**: Learn TEA step-by-step
- **[How-To Guides](/how-to/workflows/run-test-design)**: Task-focused instructions
- **[Explanation](/explanation/tea-overview)**: Understand concepts and architecture
- **[Reference](/reference/commands)**: Commands, configuration, knowledge base
- **[Glossary](/glossary)**: Terminology and definitions

## Support

- **Documentation**: [test-architect.bmad-method.org](https://test-architect.bmad-method.org)
- **Issues**: [GitHub Issues](https://github.com/bmad-code-org/bmad-method-test-architecture-enterprise/issues)
- **Discussions**: [GitHub Discussions](https://github.com/bmad-code-org/bmad-method-test-architecture-enterprise/discussions)

---

Start with the [Quick Start Tutorial](https://bmad-code-org.github.io/tutorials/tea-lite-quickstart/).
