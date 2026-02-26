# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains two main components:

1. **SAST Triaging Plugin** (root level): A Claude plugin designed to efficiently triage static application security testing (SAST) scan results, helping security teams distinguish between true positives and false positives in vulnerability reports.

2. **OWASP Benchmark for Java** (`BenchmarkJava-master/`): A reference test suite of intentionally vulnerable Java code used to evaluate the accuracy and coverage of application security tools.

## Repository Structure

```
/
├── README.md                    # Project overview and requirements
├── CLAUDE.md                    # This file
├── BenchmarkJava-master/        # OWASP Benchmark - Java test suite
│   ├── pom.xml                 # Maven build configuration
│   ├── src/main/
│   │   ├── java/               # Benchmark test classes (BenchmarkTestXXXXX.java)
│   │   ├── webapp/             # Web application resources and JSPs
│   │   └── resources/          # Configuration and resource files
│   ├── src/config/             # Tomcat and application configuration
│   ├── results/                # Benchmark test results and scorecard data
│   ├── scripts/                # Utility scripts for running tests
│   └── runBenchmark.sh         # Script to run the benchmark locally
```

## Build and Development

### Prerequisites
- Java 8 (Maven targets Java 8 as per pom.xml)
- Maven 3+ (Project uses Maven for build system)
- Tomcat 9+ (for running the web application)

### Building the Benchmark

```bash
cd BenchmarkJava-master
mvn clean package
```

This creates a WAR file that can be deployed to Tomcat.

### Building with Security Analysis

To run with SpotBugs/FindSecBugs security analysis:

```bash
cd BenchmarkJava-master
mvn clean package -Pfindsecbugs
```

### Running the Benchmark Locally

```bash
cd BenchmarkJava-master
./runBenchmark.sh
```

This starts the Benchmark web application on `https://localhost:8443` (local access only).

For remote access:

```bash
cd BenchmarkJava-master
./runRemoteAccessibleBenchmark.sh
```

This allows the application to be accessed from outside the local machine on port 8443.

### Maven Deployment Profiles

The project includes multiple profiles for deploying with different security testing tools:

- `deploy` - Basic deployment with local infrastructure
- `findsecbugs` - SpotBugs/FindSecBugs static analysis
- `deploywcontrast` - Deploy with Contrast IAST agent
- `deploywseeker` - Deploy with Synopsys Seeker IAST agent
- `deploywcxiast` - Deploy with Checkmarx CxIAST agent
- `deploywhcl` - Deploy with HCL AppScan IAST agent

Example:
```bash
mvn clean package cargo:run -Pdeploy
```

## Key Configuration Files

- **pom.xml** ([BenchmarkJava-master/pom.xml](BenchmarkJava-master/pom.xml)): Maven build configuration, profiles, and dependencies
- **web.xml** ([BenchmarkJava-master/src/config/web.xml](BenchmarkJava-master/src/config/web.xml)): Web application configuration
- **server.xml** ([BenchmarkJava-master/src/config/local/server.xml](BenchmarkJava-master/src/config/local/server.xml)): Tomcat server configuration
- **build.xml** ([BenchmarkJava-master/src/config/build.xml](BenchmarkJava-master/src/config/build.xml)): Ant build configuration for supporting services (LDAP, database)

## Test Code Organization

The Benchmark test classes are generated and organized as:
- **Location**: `BenchmarkJava-master/src/main/java/org/owasp/benchmark/testcode/`
- **Naming**: `BenchmarkTestXXXXX.java` (e.g., BenchmarkTest00001.java, BenchmarkTest02672.java)
- **Purpose**: Each test file contains intentionally vulnerable code patterns to evaluate SAST tool detection capabilities

## Architecture Notes

### Vulnerability Test Design
- Tests are intentionally vulnerable to allow security tools to detect real vulnerabilities
- Each test is designed to be exploitable, ensuring fair evaluation of detection tools
- Tests cover various vulnerability types (injection, XSS, path traversal, etc.)

### Deployment Architecture
- Application runs on Tomcat with HTTPS on port 8443
- Uses embedded LDAP and database servers for testing
- Keystore (`.keystore`) is pre-configured for HTTPS
- Default password for keystore connector is "changeit"

### SAST Tool Integration
The Cargo Maven plugin is configured to deploy the application with optional Java agents for IAST tools. Each profile sets up specific Java agent arguments and configurations for different security scanners.

## Important Build Properties

- `java.target`: 8
- `failOnMissingWebXml`: false
- `maven.war.webxml`: References custom web.xml location
- `runenv`: Defaults to 'local' (can be set to 'remote' via scripts)
- Tomcat JVM arguments: `-Xms1G -Xmx8G` by default

## Typical Development Workflow

1. **Make code changes** to test files in `src/main/java/org/owasp/benchmark/testcode/`
2. **Build the project**: `mvn clean package`
3. **Run locally**: `./runBenchmark.sh`
4. **Test with security tools**: Deploy with appropriate profile and run security scanning tools
5. **Evaluate results**: Check the results/ directory for scorecard reports

## Common Issues and Notes

- **Java version**: Ensure your system is running Java 8+. The project targets Java 8 but modern tools may require Java 11+.
- **Port 8443**: Make sure port 8443 is available before running the benchmark.
- **HTTPS certificates**: The `.keystore` file contains a pre-configured certificate for development/testing use only.
- **Database and LDAP**: The `src/config/build.xml` sets up embedded servers that start as part of the deployment process.

## SAST Triaging Plugin

The project includes a Claude Code plugin for automatically triaging SAST scan results. This plugin helps reduce AppSec team workload by analyzing findings and classifying them as true or false positives.

### Plugin Structure

```
.claude/
├── commands/
│   └── triage-sast.md           # Slash command for triaging SAST results
└── skills/
    └── sast-triage/
        └── SKILL.md             # Auto-triggered skill for SAST analysis
```

### Setup: Generate SAST Results with Semgrep

Before using the plugin, generate SAST scan results from the BenchmarkJava-master codebase:

```bash
# Install Semgrep (if not already installed)
brew install semgrep
# OR: pip install semgrep

# Run Semgrep with Java security rules and output to JSON
cd /Users/devenkumar/Documents/Triaging-Plugin
semgrep --config=p/java BenchmarkJava-master/src/main/java --json > sast-results.json
```

This creates `sast-results.json` at the project root with all identified findings.

### Using the Plugin

#### Option 1: Slash Command (Explicit)

Invoke the `/triage-sast` command with the scan results file and repository context:

```
/triage-sast sast-results.json BenchmarkJava-master/ "Java EE web application, OWASP Benchmark test suite, externally accessible via HTTPS on port 8443"
```

Claude will:
1. Parse the Semgrep JSON results
2. Read relevant source code from the flagged files
3. Analyze each finding for true/false positive status
4. Generate a detailed report and save it to `triage-report.md`

#### Option 2: Auto-Triggered Skill (Conversational)

Simply describe your SAST findings in the chat with phrases like:
- "Triage these SAST findings..."
- "Is this a false positive?"
- "Review my Semgrep results"

Claude will automatically apply the SAST triage skill methodology.

### Report Output

The plugin generates `triage-report.md` with:
- **Summary table** of all findings with verdicts (TP/FP)
- **Executive summary** of key risk areas
- **Detailed analysis** for each finding including:
  - Vulnerability type and severity
  - Verdict (TRUE POSITIVE / FALSE POSITIVE)
  - Confidence level (High/Medium/Low)
  - Explanation with code context
  - Remediation advice (if true positive)

### Triage Methodology

The plugin applies this decision flow for each finding:

1. **Read the code** — examine the flagged line and surrounding context (±25 lines)
2. **Identify vulnerability type** — injection, XSS, path traversal, etc.
3. **Trace data flow** — does untrusted user input reach the vulnerable sink?
4. **Check protections** — encoding, validation, parameterized queries, framework guards
5. **Assign verdict** — TRUE POSITIVE if exploitable, FALSE POSITIVE if not
6. **Assign confidence** — High/Medium/Low based on certainty

## References

- [OWASP Benchmark Project](https://owasp.org/www-project-benchmark/)
- [GitHub Repository](https://github.com/OWASP-Benchmark/BenchmarkJava)
- [BenchmarkUtils Project](https://github.com/OWASP-Benchmark/BenchmarkUtils) - Scorecard generation tools
- [Semgrep Documentation](https://semgrep.dev/docs/) - SAST scanning tool used for result generation
