# Spring Boot AI Assisted Testing

AI-driven wizards for test generation and SonarQube issue resolution in Spring Boot applications.

## How to Use

1. Copy the `.claude/` folder into your Spring Boot project root
2. Open the project in Claude Code
3. Invoke the wizard you need:
   - `/generate-tests` — for test generation
   - `/fix-sonar` — for SonarQube issue fixing
4. Follow the quick start guide below

---

## Test Generation Wizard

Type **`GENERATE_TESTS`** to launch the wizard that will:

1. 🔍 **Verify** your testing infrastructure is ready
2. 🎯 **Guide** you through selecting what to test
3. 📝 **Generate** test scenarios in a numbered table for your approval
4. 🚀 **Create** unit, integration, or controller tests following your project's patterns
5. ✅ **Validate** all tests pass

### What You Get
- **Unit Tests** — Service layer isolation testing with BDD-style mocking
- **Integration Tests** — Full HTTP endpoint testing with database and WireMock
- **Controller Tests** — Input validation and error response testing
- **Complete Setup** — Dependencies, annotations, factories, and assertions

---

## SonarQube Fix Wizard

Type **`FIX_SONAR`** to launch the wizard that will:

1. 🔍 **Verify** SonarQube configuration is ready
2. ⚙️ **Run** analysis on the full project
3. 📊 **Display** all issues in a dashboard grouped by severity
4. 🔧 **Fix** selected issues or **ignore** them via `sonar-project.properties`
5. ✅ **Re-analyze** to confirm resolution

### Key Features
- Fix code smells, bugs, and vulnerabilities with guided approval
- Ignore rules via `sonar-project.properties` (never annotations)
- Scoped ignores: global, test-only, specific package, or specific file
- Before/after comparison after fixes

---

## Documentation Structure

| File | Purpose |
|------|---------|
| `.claude/commands/generate-tests.md` | Test generation wizard rules |
| `.claude/commands/fix-sonar.md` | SonarQube fix wizard rules |
| `.claude/docs/INITIAL_TEST_PREQUISITES.md` | Infrastructure setup and implementations |
| `.claude/docs/UNIT_TESTING.md` | Unit test patterns and usage |
| `.claude/docs/INTEGRATION_TESTING.md` | Integration test patterns and usage |
| `.claude/docs/CONTROLLER_TESTING.md` | Controller test patterns and usage |

## Features

✅ Interactive wizard-driven workflows
✅ Infrastructure verification and auto-setup
✅ User approval before any code changes
✅ Project-specific conventions enforcement
✅ Multi-project portable
✅ Clear table-based previews of all actions