# BC Testing Skill

A comprehensive skill for AI coding agents that teaches how to write automated tests for **Microsoft Dynamics 365 Business Central** AL code.

## Table of Contents

- [What Is This Skill?](#what-is-this-skill)
- [Compatible Agents](#compatible-agents)
- [Quick Start](#quick-start)
- [What's Included](#whats-included)
- [Skill Structure](#skill-structure)
- [Topics Covered](#topics-covered)
- [Usage Examples](#usage-examples)
- [Contributing](#contributing)
- [License](#license)
- [References](#references)

## What Is This Skill?

This skill teaches AI coding agents how to write well-structured, maintainable, and isolated tests for Business Central AL code. It covers:

- **Unit tests vs Integration tests** - When to use each and how to write fast, isolated unit tests
- **Test doubles** - Dummies, stubs, spies, and mocks for dependency isolation
- **ATDD patterns** - Given-When-Then documentation structure
- **UI handlers** - MessageHandler, ConfirmHandler, PageHandler, and more
- **TestPages** - Simulating user interaction with pages
- **HTTP mocking** - Testing external API integrations
- **Testability** - Designing code with interfaces and dependency injection
- **Microsoft test libraries** - Library Assert, Library Sales, Any, and more

## Compatible Agents

This skill works with any AI coding agent that supports the Agent Skills format:

| Agent                                                 | Configuration Location |
| ----------------------------------------------------- | ---------------------- |
| [GitHub Copilot](https://github.com/features/copilot) | `.github/skills/`      |
| [Claude Code](https://claude.ai/code)                 | `.claude/skills/`      |
| [Cursor](https://cursor.com/)                         | `.cursor/skills/`      |
| [VS Code](https://code.visualstudio.com/)             | `.github/skills/`      |
| [Windsurf](https://codeium.com/windsurf)              | `.windsurf/skills/`    |
| [Cline](https://github.com/cline/cline)               | `.cline/skills/`       |

## Quick Start

### Manual Installation

1. Copy the `bc-testing` folder to your agent's skills directory:

```bash
# For GitHub Copilot / VS Code
cp -r bc-testing .github/skills/

# For Claude Code
cp -r bc-testing .claude/skills/

# For Cursor
cp -r bc-testing .cursor/skills/
```

2. Start using the skill by asking your AI agent about BC testing:

```
How do I write a test for posting a sales order?
```

### Using with npx (if published)

```bash
npx @tech-leads-club/agent-skills install -s bc-testing
```

## What's Included

```
bc-testing/
├── SKILL.md                          # Main skill file with rules and patterns
└── references/                       # Detailed documentation
    ├── TEST-CODEUNITS.md             # Test codeunit setup and attributes
    ├── HANDLERS.md                   # UI handler types and signatures
    ├── TEST-PAGES.md                 # TestPage usage and navigation
    ├── ASSERTIONS.md                 # Assert methods and ASSERTERROR
    ├── DATA-SETUP.md                 # Library codeunits and random data
    ├── ISOLATION.md                  # Transaction models and rollback
    ├── PATTERNS.md                   # ATDD Given-When-Then patterns
    ├── TEST-TYPES.md                 # Unit vs integration tests
    ├── TESTABILITY.md                # Dependency injection and interfaces
    └── HTTP-MOCKING.md               # HttpClientHandler for API testing
```

## Skill Structure

The skill follows the standard Agent Skills format:

- **SKILL.md** - Main instruction file (~250 lines) with:
  - Metadata (name, description, version)
  - When to apply the skill
  - Rule categories by priority
  - Quick reference for all rules
  - Core patterns and examples
  - Best practices summary

- **references/** - Detailed documentation files that the agent loads on-demand when deeper context is needed

## Topics Covered

### Core Testing (Priority: CRITICAL to HIGH)

| Topic          | Description                                                   |
| -------------- | ------------------------------------------------------------- |
| Test Structure | Codeunit setup, `[Test]` attribute, `[HandlerFunctions]`      |
| UI Handlers    | MessageHandler, ConfirmHandler, PageHandler, ModalPageHandler |
| Test Types     | Unit tests (<100ms) vs integration tests (500ms+)             |
| Assertions     | `Assert.AreEqual`, `Assert.IsTrue`, `ASSERTERROR`             |
| Test Pages     | OpenView, OpenEdit, navigation, field access                  |
| Testability    | Interfaces, dependency injection, decoupling                  |

### Data & Isolation (Priority: MEDIUM)

| Topic          | Description                                                   |
| -------------- | ------------------------------------------------------------- |
| Data Setup     | Library - Sales, Library - Random, Library - Variable Storage |
| Test Isolation | TestIsolation property, TransactionModel, AutoRollback        |
| ATDD Patterns  | `[Scenario]`, `[Given]`, `[When]`, `[Then]` comments          |
| HTTP Mocking   | `[HttpClientHandler]`, TestHttpRequestPolicy                  |

### Advanced (Priority: LOW to MEDIUM)

| Topic              | Description                                        |
| ------------------ | -------------------------------------------------- |
| Test Doubles       | Dummies, stubs, spies, mocks                       |
| Event Mocking      | EventSubscriberInstance = Manual, BindSubscription |
| Permission Testing | Library - Lower Permissions                        |

## Usage Examples

### Ask for Test Structure

```
Create a test codeunit for testing customer discount calculations
```

### Ask About Handlers

```
How do I handle a confirm dialog that appears when posting a document?
```

### Ask About Mocking

```
How can I test code that calls an external REST API without making real HTTP calls?
```

### Ask About Best Practices

```
What's the difference between unit tests and integration tests in BC?
```

## Contributing

Contributions are welcome! If you'd like to improve this skill:

1. Fork the repository
2. Create a feature branch (`git checkout -b feat/improvement`)
3. Make your changes
4. Commit with conventional commits (`git commit -m "feat: add new pattern"`)
5. Push to your fork (`git push origin feat/improvement`)
6. Open a Pull Request

### Guidelines

- Keep SKILL.md under 500 lines for optimal agent performance
- Put detailed documentation in `references/` folder
- Include working AL code examples
- Follow the existing formatting style
- Test with at least one AI coding agent

## License

MIT License - see [LICENSE](LICENSE) file for details.

## References

This skill is based on:

- [Microsoft Learn: Testing the Application](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-testing-application)
- [Microsoft BCApps Test Framework](https://github.com/microsoft/BCApps/tree/main/src/Tools/Test%20Framework)
- [Vjeko.com - Testing in Isolation](https://vjeko.com/2020/06/19/testing-in-isolation/)
- [Vjeko.com - Testing, Testability, and All Things Test](https://vjeko.com/2020/06/05/c-side-fridays-testing-testability-and-all-things-test/)
- [Stefan Šošić - Mocking HTTP Calls in AL Tests](https://ssosic.com/development/mocking-http-calls-in-al-tests/)
- [Waldo's Blog - Speeding Up Test Suites](https://www.waldo.be/)

---

Built for the Business Central developer community
