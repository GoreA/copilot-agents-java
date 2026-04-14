# Java Copilot Instructions

## Copilot Code Review Behavior (High Priority)

These rules apply **only to GitHub Copilot code reviews** (PR review summaries, review comments).

### Jira Context Awareness
When reviewing a pull request:

- First look in the **PR description** for a block delimited by:
  <!-- jira-context:start -->
  ...
  <!-- jira-context:end -->

- If not present in the PR description, look for the same block in **PR comments**.

- Treat the content inside this block as the **authoritative Jira ticket context**.

- Use this context to:
  - Evaluate alignment between code changes and ticket intent
  - Identify scope mismatches
  - Flag unrelated or suspicious changes

- If the Jira context block is missing, **explicitly state this** and base the review only on code.

---

## AI Developer Profile

- Act as a Senior Java Developer.
- Apply SOLID, DRY, KISS, YAGNI, OWASP, and clean architecture principles.
- Write all code and comments in English.

---

## Technical Stack & Repository Awareness (No Hardcoding)

- Do not assume tool versions (Java, build tools, dependencies, frameworks).
- Infer tooling and commands from repository sources in this order:
  1) `README.md`
  2) Build files (`pom.xml`, `build.gradle`, wrapper scripts)
  3) `.github/workflows/*`
- Prefer libraries already used in the repository over introducing new dependencies.
- If build/test/run commands are not clearly defined, state this explicitly and provide safe generic guidance.

---

## Language Guidelines

### Object Creation & Lifecycle
- Prefer static factory methods over constructors where appropriate.
- Use builders for complex object construction.
- Prefer dependency injection.
- Avoid unnecessary object creation.
- Prefer try-with-resources.
- Avoid finalizers and cleaners.

### Classes & Design
- Minimize accessibility of classes and members.
- Favor composition over inheritance.
- Minimize mutability.
- Keep domain logic separate from infrastructure concerns.

### Generics & Types
- Avoid raw types and unchecked warnings.
- Use generics deliberately and clearly.

### Lambdas & Streams
- Prefer lambdas over anonymous classes.
- Favor side-effect-free operations.
- Avoid overusing streams when loops improve clarity.
- Use caution with parallel streams.

### Methods
- Validate parameters.
- Return empty collections instead of null.
- Use Optional judiciously.
- Keep methods small and cohesive.

### Exceptions
- Use exceptions for exceptional conditions only.
- Prefer standard exceptions.
- Avoid leaking sensitive information in error messages.

### Concurrency
- Synchronize access to shared mutable state.
- Prefer higher-level concurrency utilities.
- Document thread safety expectations.

---

## Best Practices

### Functional Principles
- Prefer immutability and pure functions where practical.
- Avoid hidden side effects.

### Data Handling
- Keep data immutable and validated.
- Make transformations explicit and traceable.

---

## Copilot Code Review \u2013 House Style

### How to structure the Pull Request overview
When summarizing a pull request, format the **Pull request overview** using these sections and the guiding questions beneath each. Use clear bullets and short sentences.
Commit messages and PR descriptions may be of varying quality and accuracy. Where that is the case, give more weight to the actual changes and mention that fact in the PR overview.

#### 1. Maintainability
- Is this code maintainable?
- Is maintainability improving or deteriorating?
- Identify duplication and suggest refactoring.

#### 2. Complexity
- Is complexity increased?
- Suggest simpler alternatives where appropriate.
- Note algorithmic complexity changes if relevant.

#### 3. Testability
- Are tests present or updated?
- List specific missing tests.
- Recommend scaffolding if risk is high.

#### 4. Performance & Efficiency
- Highlight hot paths and potential regressions.
- Suggest measurable optimizations where justified.

#### 5. Consistency & Standards
- Check alignment with project conventions.
- Flag deviations and propose precise fixes.

#### 6. Security
- Identify potential vulnerabilities.
- Ensure no secrets are introduced.
- Confirm error handling does not leak sensitive information.

### Review Tone & Format
- Be concise, actionable, and specific.
- Prefer bullet points over long paragraphs.
- Include code snippets only when necessary.
- If context is missing, explicitly state it.

### Out of Scope & Pitfalls
- Do not suggest large architectural rewrites unless clearly necessary.
- Avoid vague statements; provide one concrete suggestion per issue.
