---
name: static-analysis
description: Set up, configure, or improve static analysis and code quality automation — use when someone asks about linters, SAST, SonarQube, SpotBugs, ruff, mypy, clippy, quality gates, or CI code quality checks.
---

# Static Analysis and Code Quality Automation

## What Static Analysis Does

Static analysis examines source code (or bytecode) **without executing it**. It finds:

- **Bugs**: null dereferences, resource leaks, type errors, unreachable code
- **Security vulnerabilities**: SQL injection, path traversal, hardcoded secrets, unsafe deserialization
- **Code smells**: excessive complexity, duplicated logic, unused variables
- **Style violations**: formatting inconsistencies, naming convention violations

Types of static analysis:

```
Linting          --> style/convention violations (fast, low false-positive rate)
SAST             --> security vulnerabilities (slower, needs tuning)
Complexity       --> cyclomatic complexity, cognitive complexity, coupling metrics
Dependency audit --> known CVEs in third-party libraries
Type checking    --> type errors without running tests
```

**Key principle**: static analysis should be fast enough to run on every commit. If it takes 20 minutes, engineers will skip it.

---

## Java / Kotlin

### SpotBugs + Find Security Bugs

SpotBugs performs bytecode analysis — it catches issues after compilation that source analysis misses.

Find Security Bugs plugin adds OWASP-category checks: SQL injection, XSS, path traversal, XXE, unsafe reflection.

Maven configuration:
```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <dependencies>
        <dependency>
            <groupId>com.h3xstream.findsecbugs</groupId>
            <artifactId>findsecbugs-plugin</artifactId>
            <version>1.13.0</version>
        </dependency>
    </dependencies>
    <configuration>
        <effort>Max</effort>
        <threshold>Low</threshold>
        <failOnError>true</failOnError>
        <excludeFilterFile>spotbugs-exclusions.xml</excludeFilterFile>
        <plugins>
            <plugin>
                <groupId>com.h3xstream.findsecbugs</groupId>
                <artifactId>findsecbugs-plugin</artifactId>
                <version>1.13.0</version>
            </plugin>
        </plugins>
    </configuration>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

Exclusion filter (`spotbugs-exclusions.xml`) — suppress known false positives:
```xml
<FindBugsFilter>
    <!-- Suppress false positive: we validate this externally via Bean Validation -->
    <Match>
        <Class name="com.example.api.UserController" />
        <Bug pattern="SPRING_UNVALIDATED_REDIRECT" />
    </Match>

    <!-- Generated code: suppress all -->
    <Match>
        <Package name="~com\.example\.generated\..*" />
    </Match>
</FindBugsFilter>
```

Run: `mvn spotbugs:check`

### Checkstyle

Enforces coding conventions. Use Google Style as a baseline, then override project-specific rules.

Maven:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <failsOnError>true</failsOnError>
        <includeTestSourceDirectory>true</includeTestSourceDirectory>
    </configuration>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
            <phase>validate</phase>
        </execution>
    </executions>
</plugin>
```

`checkstyle.xml` (subset of Google Style with project overrides):
```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
    <property name="severity" value="error"/>
    <property name="fileExtensions" value="java"/>

    <module name="TreeWalker">
        <!-- naming -->
        <module name="ConstantName"/>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        <module name="MethodName"/>
        <module name="TypeName"/>

        <!-- imports -->
        <module name="AvoidStarImport"/>
        <module name="UnusedImports"/>

        <!-- complexity -->
        <module name="CyclomaticComplexity">
            <property name="max" value="15"/>
        </module>
        <module name="MethodLength">
            <property name="max" value="80"/>
        </module>

        <!-- code quality -->
        <module name="EmptyCatchBlock">
            <property name="exceptionVariableName" value="expected|ignore"/>
        </module>
        <module name="MagicNumber">
            <property name="ignoreNumbers" value="-1, 0, 1, 2"/>
        </module>
        <module name="EqualsHashCode"/>
    </module>

    <!-- file-level -->
    <module name="FileLength">
        <property name="max" value="500"/>
    </module>
    <module name="NewlineAtEndOfFile"/>
</module>
```

### PMD

Finds code smells the compiler doesn't catch.

`pmd.xml` ruleset:
```xml
<?xml version="1.0"?>
<ruleset name="Project Rules"
    xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0
                        https://pmd.sourceforge.net/ruleset_2_0_0.xsd">

    <rule ref="category/java/bestpractices.xml">
        <exclude name="GuardLogStatement"/>  <!-- handled by SLF4J -->
    </rule>
    <rule ref="category/java/errorprone.xml"/>
    <rule ref="category/java/performance.xml"/>
    <rule ref="category/java/security.xml"/>

    <!-- Suppress: generated equals/hashCode from Lombok is fine -->
    <rule ref="category/java/errorprone.xml/MissingSerialVersionUID">
        <properties>
            <property name="violationSuppressXPath"
                value="//ClassDeclaration[../Annotation[@SimpleName='Entity']]"/>
        </properties>
    </rule>
</ruleset>
```

### detekt (Kotlin)

```kotlin
// build.gradle.kts
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.5"
}

detekt {
    config.setFrom(files("$rootDir/detekt.yml"))
    buildUponDefaultConfig = true
    allRules = false
}
```

`detekt.yml` (key rules):
```yaml
complexity:
  CyclomaticComplexMethod:
    threshold: 15
  LongMethod:
    threshold: 80
  LongParameterList:
    functionThreshold: 8

coroutines:
  GlobalCoroutineUsage:
    active: true
  RedundantSuspendModifier:
    active: true

naming:
  FunctionNaming:
    functionPattern: '[a-z][a-zA-Z0-9]*'
  VariableNaming:
    variablePattern: '[a-z][A-Za-z0-9]*'

style:
  MagicNumber:
    active: true
    ignoreNumbers: ['-1', '0', '1', '2']
  UnusedPrivateMember:
    active: true
```

### SonarQube

SonarQube aggregates all the above and adds trend tracking, Quality Gates, and technical debt visualization.

Quality Gate configuration (block PRs that regress quality):
```
New Code Quality Gate — fail if:
  - New Bugs > 0
  - New Vulnerabilities > 0
  - New Security Hotspots Reviewed < 100%
  - New Code Coverage < 80%
  - New Duplicated Lines > 3%
  - New Cognitive Complexity > 15 per method
```

Maven analysis:
```bash
mvn sonar:sonar \
  -Dsonar.projectKey=my-service \
  -Dsonar.host.url=https://sonarqube.company.com \
  -Dsonar.login=$SONAR_TOKEN \
  -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

Key SonarQube metrics to monitor:
- **Reliability Rating**: A (0 bugs) through E (critical bugs)
- **Security Rating**: A (0 vulnerabilities) through E
- **Maintainability Rating**: based on technical debt ratio (debt / dev time)
- **Coverage**: line + branch coverage from JaCoCo
- **Cognitive Complexity**: measures how hard a function is to understand (not just branches)

---

## Python

### ruff — The Single Tool for Python Linting

ruff replaces flake8, isort, pyupgrade, pydocstyle, and dozens of plugins. It is 10-100x faster.

```bash
# install
pip install ruff

# check and auto-fix
ruff check --fix .

# format (replaces black)
ruff format .
```

`pyproject.toml`:
```toml
[tool.ruff]
target-version = "py312"
line-length = 100
src = ["src"]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "B",    # flake8-bugbear (common bugs)
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade (modern Python syntax)
    "S",    # bandit security checks
    "RUF",  # ruff-specific rules
]
ignore = [
    "E501",   # line too long (handled by formatter)
    "S101",   # use of assert (OK in tests)
]
per-file-ignores = { "tests/**/*.py" = ["S", "B"] }

[tool.ruff.lint.isort]
known-first-party = ["mypackage"]
```

### mypy — Static Type Checking

```bash
pip install mypy

# strict mode: catch all type errors
mypy --strict src/
```

`pyproject.toml`:
```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true

# per-module overrides for third-party libraries without stubs
[[tool.mypy.overrides]]
module = ["requests.*", "boto3.*"]
ignore_missing_imports = true
```

Common mypy errors and fixes:
```python
# Error: Argument 1 to "process" has incompatible type "Optional[str]"; expected "str"
def process(value: Optional[str]) -> str:
    return value.upper()  # mypy catches: value could be None

# Fix:
def process(value: Optional[str]) -> str:
    if value is None:
        raise ValueError("value must not be None")
    return value.upper()
```

### Bandit — Security SAST for Python

```bash
pip install bandit

# scan with severity and confidence filters
bandit -r src/ -ll -ii  # low severity or higher, medium confidence or higher

# output formats: text, json, html, csv
bandit -r src/ -f json -o bandit-report.json
```

Key Bandit checks:
- `B201` / `B202`: Flask debug mode enabled
- `B301` / `B302`: pickle / marshal (unsafe deserialization)
- `B501` - `B509`: SSL/TLS misconfiguration
- `B601` - `B612`: shell injection (subprocess with shell=True)
- `B701` - `B703`: Jinja2 / Mako template injection
- `B801` - `B803`: XML vulnerabilities (XXE)

Suppress false positives with inline comments:
```python
# Intentional: this subprocess call uses a hardcoded command, not user input
result = subprocess.run(["git", "status"], capture_output=True)  # noqa: S603
```

### radon — Complexity Analysis

```bash
pip install radon

# cyclomatic complexity (A=1-5, B=6-10, C=11-15, D=16-20, E=21-25, F=25+)
radon cc -s -a src/

# maintainability index
radon mi -s src/

# raw metrics (LOC, SLOC, comments)
radon raw src/
```

Target: average cyclomatic complexity below B (10). Functions rated D or higher warrant refactoring.

---

## Rust

### clippy — The Official Linter

```bash
# fail CI on any warning
cargo clippy -- -D warnings

# enable all pedantic lints (more aggressive)
cargo clippy -- -D warnings -W clippy::pedantic

# check specific categories
cargo clippy -- -D clippy::correctness -D clippy::suspicious -W clippy::perf
```

Lint categories:
- `correctness`: almost certainly wrong code; enabled by default; very low false-positive rate
- `suspicious`: code that is probably wrong or confusing
- `style`: idiomatic Rust; enabled by default
- `complexity`: needlessly complex expressions
- `perf`: code that could be faster with minor changes
- `pedantic`: opinionated; higher false-positive rate; opt-in
- `nursery`: experimental; may have bugs; opt-in

`Cargo.toml` workspace-level clippy configuration:
```toml
[workspace.metadata.clippy]
# List of lints to deny workspace-wide
deny = ["clippy::unwrap_used", "clippy::expect_used", "clippy::panic"]
```

Inline suppression with justification:
```rust
#[allow(clippy::too_many_arguments)]
// This function mirrors the exact signature of the external API we wrap;
// splitting it would require a builder pattern that adds complexity without clarity.
pub fn create_invoice(
    customer_id: Uuid,
    line_items: Vec<LineItem>,
    // ... 8 more params
) -> Result<Invoice, InvoiceError> {
    // ...
}
```

### cargo-audit — Dependency CVE Checking

```bash
cargo install cargo-audit

# check against RustSec advisory database
cargo audit

# output as JSON for CI parsing
cargo audit --json

# fail on specific severity
cargo audit --deny warnings
```

### cargo-deny — Policy Enforcement

```bash
cargo install cargo-deny
```

`deny.toml`:
```toml
[licenses]
allow = ["MIT", "Apache-2.0", "Apache-2.0 WITH LLVM-exception", "BSD-2-Clause", "BSD-3-Clause"]
deny = ["GPL-2.0", "GPL-3.0", "AGPL-3.0"]
copyleft = "deny"
unlicensed = "deny"

[bans]
multiple-versions = "warn"
wildcards = "deny"   # no '*' version specs
deny = [
    { name = "openssl", reason = "use rustls instead" },
]

[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"
```

Run: `cargo deny check`

---

## CI Integration

### Pipeline Stage Order

```
Stage 1: Static Analysis (fastest; catch issues before tests)
  - linting (ruff check, clippy, checkstyle)
  - type checking (mypy, kotlinc)
  - security scan (bandit, find-security-bugs)

Stage 2: Tests (slower; need a clean codebase)
  - unit tests
  - integration tests

Stage 3: Quality Gate (after tests; needs coverage data)
  - SonarQube analysis
  - coverage threshold check
  - dependency audit (cargo-audit, OWASP dependency-check)

Stage 4: Build / Package (only if all above pass)
```

### GitHub Actions — Java (Maven + SpotBugs + Checkstyle)

```yaml
name: Code Quality

on: [push, pull_request]

jobs:
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Checkstyle
        run: mvn checkstyle:check -q

      - name: SpotBugs
        run: mvn spotbugs:check

      - name: PMD
        run: mvn pmd:check

      - name: Build and Test with Coverage
        run: mvn verify -P jacoco

      - name: SonarQube Analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=${{ github.repository_owner }}_${{ github.event.repository.name }} \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

### GitHub Actions — Python (ruff + mypy + bandit)

```yaml
name: Python Code Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip

      - name: Install dependencies
        run: pip install ruff mypy bandit radon pytest pytest-cov

      - name: ruff lint
        run: ruff check .

      - name: ruff format check
        run: ruff format --check .

      - name: mypy type check
        run: mypy --strict src/

      - name: bandit security scan
        run: bandit -r src/ -ll -ii -x tests/

      - name: Complexity check
        run: |
          radon cc -s -a src/ --min B
          # fail if average complexity exceeds B
          radon cc -a src/ | grep -E "Average complexity: [CDEF]" && exit 1 || true

      - name: Tests with coverage
        run: pytest --cov=src --cov-report=xml --cov-fail-under=80
```

### GitHub Actions — Rust (clippy + audit + deny)

```yaml
name: Rust Code Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - name: Cache cargo
        uses: Swatinem/rust-cache@v2

      - name: rustfmt check
        run: cargo fmt --all -- --check

      - name: clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

      - name: cargo-audit
        run: |
          cargo install cargo-audit --locked
          cargo audit --deny warnings

      - name: cargo-deny
        run: |
          cargo install cargo-deny --locked
          cargo deny check

      - name: Tests
        run: cargo test --all-features
```

---

## Baseline Strategy — Don't Fail on Legacy Issues

When introducing static analysis into an existing codebase, failing on all existing issues is often impractical.

**SonarQube "New Code" mode**: only fail the Quality Gate on issues introduced after a specific date or in the current PR. Existing issues become "technical debt" tracked separately.

**Ratchet pattern for other tools**: capture a baseline on the main branch, then fail CI only if the count increases.

```bash
# baseline.sh — run on main branch, commit the output
ruff check . --output-format=json | jq '[.[] | .code] | unique | sort' > .ruff-baseline.json

# check.sh — run in CI, fail if new violations found
current=$(ruff check . --output-format=json | jq '[.[] | .code] | unique | sort')
baseline=$(cat .ruff-baseline.json)
if [ "$current" != "$baseline" ]; then
  echo "New ruff violations introduced"
  exit 1
fi
```

**SpotBugs baseline**: use the `excludeFilterFile` to suppress all existing violations; add new suppressions only when a violation is a confirmed false positive (with a comment explaining why).

---

## Configuration Reference

### Complete `pyproject.toml` (ruff + mypy + pytest)

```toml
[tool.ruff]
target-version = "py312"
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP", "S", "RUF", "N", "ANN"]
ignore = ["E501", "ANN101", "ANN102", "ANN401"]
per-file-ignores = { "tests/**" = ["S101", "ANN", "B"] }

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]  # allow unused imports in __init__.py

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
plugins = ["pydantic.mypy"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=term-missing --cov-fail-under=80"
```

### `.clippy.toml`

```toml
# Maximum allowed cyclomatic complexity
cognitive-complexity-threshold = 20

# Warn when a function has more arguments than this
too-many-arguments-threshold = 7

# Warn when a function has more lines than this
too-many-lines-threshold = 100

# Minimum chars in a string literal to trigger clippy::min_ident_chars
min-ident-chars-threshold = 2
```

---

## Anti-Patterns

**Running Static Analysis Only Before Release**
- Static analysis finds issues cheapest when found at commit time; at release time they require rework under pressure

**Suppressing Everything Without Review**
- Bulk-suppressing all violations to get a green build without reviewing them introduces false security
- Each suppression must have a comment explaining why it is safe to ignore

**Treating All Warnings as Errors From Day One on Legacy Code**
- Causes the team to suppress everything rather than fix root causes
- Use the baseline/ratchet approach to freeze existing issues and fail only on new ones

**Not Running the Security Scanner (Bandit/Find Security Bugs)**
- Security issues are the most expensive static analysis findings to skip; they become CVEs
- Security checks should always be fail-the-build, even if style checks are warn-only

**Ignoring Complexity Metrics**
- Cyclomatic complexity above 20 is a reliable predictor of bug density; address it before tests, not after
