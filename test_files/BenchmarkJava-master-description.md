// filepath: test_files/BenchmarkJava-master-description.md
# OWASP Benchmark for Java - Repository Description

## Overview

The **OWASP Benchmark for Java** is a reference test suite consisting of intentionally vulnerable Java code designed to evaluate the accuracy, coverage, and effectiveness of application security testing tools. This repository serves as a standardized benchmark for measuring the capabilities of Static Application Security Testing (SAST), Interactive Application Security Testing (IAST), and Dynamic Application Security Testing (DAST) tools.

## Purpose and Use Cases

### Primary Purpose
- **Security Tool Evaluation**: Provides a controlled environment to assess how well security scanners detect real vulnerabilities
- **False Positive Analysis**: Helps distinguish between tools that accurately identify vulnerabilities versus those with high false positive rates
- **Coverage Assessment**: Enables measurement of which vulnerability types different tools can and cannot detect
- **Baseline Comparison**: Offers a standardized benchmark for comparing security tool performance across vendors

### Target Audience
- Security tool vendors evaluating their products
- Security teams selecting SAST/IAST tools for their organization
- Researchers studying vulnerability detection techniques
- AppSec professionals training on vulnerability patterns

## Repository Structure

```
BenchmarkJava-master/
├── pom.xml                         # Maven build configuration with multiple deployment profiles
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── org/owasp/benchmark/
│   │   │       ├── testcode/       # Test case classes (BenchmarkTestXXXXX.java)
│   │   │       ├── helpers/        # Utility classes for test execution
│   │   │       └── score/          # Scorecard generation logic
│   │   ├── webapp/                 # Web application resources
│   │   │   ├── WEB-INF/           # Web configuration
│   │   │   └── *.jsp              # JSP pages for web interface
│   │   └── resources/             # Configuration files and data
│   └── config/
│       ├── web.xml                # Servlet configuration
│       ├── build.xml              # Ant build for supporting services
│       ├── local/
│       │   └── server.xml         # Tomcat configuration (local access)
│       └── remote/
│           └── server.xml         # Tomcat configuration (remote access)
├── results/                       # Test execution results and scorecards
├── scripts/                       # Utility scripts for benchmark execution
├── .keystore                      # Pre-configured SSL certificate (dev/test only)
├── runBenchmark.sh               # Shell script to run benchmark locally
├── runBenchmark.bat              # Windows batch script to run benchmark
├── runRemoteAccessibleBenchmark.sh # Script for remote-accessible deployment
├── createScorecards.sh           # Generate scorecard reports from results
├── expectedresults-1.2.csv       # Expected vulnerability detection outcomes
└── README.md                     # Project documentation
```

## Test Code Organization

### Test Case Structure

The benchmark contains thousands of test cases organized as individual Java servlet classes:

- **Location**: `src/main/java/org/owasp/benchmark/testcode/`
- **Naming Convention**: `BenchmarkTestXXXXX.java` (e.g., `BenchmarkTest00001.java`, `BenchmarkTest00684.java`)
- **Test Count**: Over 2,600 individual test cases covering multiple vulnerability categories

### Vulnerability Categories Tested

Based on the SAST results analyzed, the benchmark tests include:

1. **Path Traversal** - Testing file access control bypasses
2. **Cryptographic Issues**:
   - Deprecated DES cipher usage
   - Deprecated 3DES (DESede) cipher usage
   - Weak hashing algorithms (MD5, SHA-1)
3. **Cross-Site Scripting (XSS)** - Direct response writer vulnerabilities
4. **SQL Injection** - Tainted SQL from HTTP requests
5. **XPath Injection** - Tainted XPath expressions
6. **Command Injection** - OS command execution vulnerabilities
7. **Session Management** - Tainted session data handling

### Test Case Design Philosophy

Each test case is designed to be:
- **Intentionally Vulnerable**: Contains exploitable security flaws by design
- **Realistic**: Mimics real-world code patterns found in production applications
- **Self-Contained**: Each test is an independent servlet that can be tested individually
- **Documented**: Expected outcomes are tracked in `expectedresults-1.2.csv`

## Build System and Technology Stack

### Core Technologies

- **Language**: Java 8 (target compatibility)
- **Build Tool**: Maven 3+
- **Web Framework**: Java Servlets / Java EE
- **Application Server**: Apache Tomcat 9+
- **Security Protocol**: HTTPS on port 8443
- **Supporting Services**:
  - Embedded LDAP server for authentication testing
  - Embedded database for SQL injection testing

### Maven Configuration

The `pom.xml` includes multiple profiles for different deployment scenarios:

#### Basic Deployment
```bash
# Standard build
mvn clean package

# Build with SpotBugs/FindSecBugs static analysis
mvn clean package -Pfindsecbugs
```

#### IAST Agent Deployment Profiles

The project supports deployment with various IAST (Interactive Application Security Testing) agents:

- **`deploywcontrast`**: Deploy with Contrast Security IAST agent
- **`deploywseeker`**: Deploy with Synopsys Seeker IAST agent
- **`deploywcxiast`**: Deploy with Checkmarx CxIAST agent
- **`deploywhcl`**: Deploy with HCL AppScan IAST agent

Example:
```bash
mvn clean package cargo:run -Pdeploywcontrast
```

### JVM Configuration

The project uses the Cargo Maven plugin with the following JVM settings:
- **Minimum Heap**: 1GB (`-Xms1G`)
- **Maximum Heap**: 8GB (`-Xmx8G`)
- **Additional**: Agent-specific JVM arguments when using IAST profiles

## Running the Benchmark

### Local Execution (Default)

```bash
cd BenchmarkJava-master
./runBenchmark.sh
```

**Characteristics**:
- Accessible only from localhost
- Uses HTTPS on port 8443
- URL: `https://localhost:8443`
- Uses self-signed certificate from `.keystore` file

### Remote Access Execution

```bash
cd BenchmarkJava-master
./runRemoteAccessibleBenchmark.sh
```

**Characteristics**:
- Accessible from external machines
- Still uses HTTPS on port 8443
- Suitable for scanning with remote security tools

### Manual Deployment

```bash
# Deploy with Cargo
mvn clean package cargo:run -Pdeploy

# Deploy with IAST agent (example: Contrast)
mvn clean package cargo:run -Pdeploywcontrast
```

## Architecture and Design

### Web Application Architecture

1. **Servlet-Based**: Each test case is implemented as a Java servlet responding to HTTP requests
2. **Request Handling**: Tests extract user input from:
   - HTTP parameters
   - Headers
   - Cookies
   - Session attributes
3. **Vulnerability Simulation**: Input flows through intentionally vulnerable code paths to vulnerable sinks
4. **Response Generation**: Most tests return results via HTTP response writers or JSP pages

### Intentional Vulnerability Design

Each test case follows this pattern:

```
User Input → Insufficient Validation → Vulnerable Operation → Exploitable Outcome
```

Examples:
- **Path Traversal**: Request parameter → File path construction → File operation
- **SQL Injection**: Request parameter → String concatenation → SQL query execution
- **XSS**: Request parameter → Direct response writer → Unescaped HTML output
- **Crypto**: Hardcoded algorithm selection → DES/3DES cipher → Weak encryption

### Supporting Infrastructure

The `src/config/build.xml` Ant script sets up:
- Embedded Apache Directory Server for LDAP testing
- Embedded HSQLDB database for SQL injection testing
- Test data population

## Security Tool Integration

### SAST Tool Integration

The benchmark is designed to be scanned by any SAST tool. Example with Semgrep:

```bash
# Install Semgrep
brew install semgrep
# OR: pip install semgrep

# Scan the benchmark and generate JSON results
semgrep --config=p/java BenchmarkJava-master/src/main/java --json > sast-results.json
```

### IAST Tool Integration

IAST tools are integrated via Java agents configured in Maven profiles. The Cargo plugin automatically:
1. Downloads the IAST agent JAR
2. Configures JVM arguments to load the agent
3. Starts Tomcat with the agent attached
4. The agent instruments the application at runtime

### DAST Tool Integration

DAST tools can scan the running benchmark via:
- Direct HTTP/HTTPS requests to port 8443
- Web crawling from the root URL
- Test case enumeration (each servlet has a predictable URL pattern)

## Scorecard Generation

### Purpose
After running security tools against the benchmark, scorecards quantify tool performance:
- **True Positive Rate**: Percentage of real vulnerabilities correctly identified
- **False Positive Rate**: Percentage of findings that are not actual vulnerabilities
- **Coverage**: Which vulnerability categories the tool can detect

### Generating Scorecards

```bash
cd BenchmarkJava-master
./createScorecards.sh
```

This script:
1. Reads tool output files from the `results/` directory
2. Compares findings against `expectedresults-1.2.csv` (ground truth)
3. Generates HTML scorecard reports
4. Requires the separate BenchmarkUtils project for scorecard generation

### Expected Results Format

The `expectedresults-1.2.csv` file contains:
- Test case identifier (e.g., `BenchmarkTest00001`)
- Vulnerability category (e.g., `pathtraver`, `crypto`, `xss`)
- Whether a vulnerability truly exists (`true`/`false`)

This ground truth enables automated accuracy measurement.

## SSL/TLS Configuration

### Keystore Details
- **File**: `.keystore` (pre-configured in repository)
- **Password**: `changeit` (default, hardcoded in `server.xml`)
- **Purpose**: Enables HTTPS for the benchmark web application
- **⚠️ Warning**: This is a development/testing certificate only, not for production use

### Port Configuration
- **HTTP**: Disabled (security requirement)
- **HTTPS**: Port 8443 (non-standard to avoid conflicts with production servers)

## Common Issues and Solutions

### Port 8443 Already in Use
```bash
# Check what's using the port
lsof -i :8443

# Kill the process if needed
kill -9 <PID>
```

### Java Version Compatibility
- **Minimum**: Java 8 (project target)
- **Recommended**: Java 11 or higher for modern security tools
- **Check Version**: `java -version`

### Maven Build Failures
```bash
# Clean and rebuild
mvn clean install

# Skip tests if needed
mvn clean package -DskipTests
```

### HTTPS Certificate Warnings
- Expected behavior due to self-signed certificate
- Browser warnings can be safely bypassed for testing
- Tools may need certificate validation disabled

## Integration with SAST Triage Plugin

This benchmark repository is used in conjunction with the **SAST Triaging Plugin** (parent directory) to:

1. **Generate Test Data**: Semgrep scans produce `sast-results.json` with 1,909+ findings
2. **Triage Practice**: The plugin analyzes findings to distinguish true vs. false positives
3. **Methodology Validation**: Since vulnerabilities are intentional, triage accuracy can be measured
4. **Training Data**: Provides realistic examples for developing triage algorithms

### Example Workflow

```bash
# 1. Scan the benchmark
semgrep --config=p/java BenchmarkJava-master/src/main/java --json > sast-results.json

# 2. Triage findings with Claude plugin
/triage-sast sast-results.json BenchmarkJava-master/ "Java EE web application, OWASP Benchmark test suite, intentionally vulnerable code" 20

# 3. Review triage report
cat triage-report.md
```

## Research and Educational Value

### For Security Researchers
- **Controlled Environment**: Known vulnerabilities enable accurate tool evaluation
- **Diverse Coverage**: Wide range of vulnerability types in one codebase
- **Reproducible Results**: Consistent test cases for comparative studies
- **Open Source**: Full code access for analysis and modification

### For Security Tool Vendors
- **Benchmarking Standard**: Industry-recognized performance baseline
- **Gap Analysis**: Identify which vulnerability types need better detection
- **Marketing Validation**: Provide objective performance data to customers
- **Continuous Improvement**: Track detection improvements across tool versions

### For Security Educators
- **Teaching Resource**: Real-world vulnerable code examples
- **Vulnerability Patterns**: Illustrates common security anti-patterns
- **Remediation Practice**: Students can practice fixing vulnerabilities
- **Tool Training**: Learn how to configure and interpret SAST/IAST tools

## Related Projects

### BenchmarkUtils
- **Repository**: https://github.com/OWASP-Benchmark/BenchmarkUtils
- **Purpose**: Scorecard generation and result parsing
- **Functionality**: Converts tool output to standardized metrics

### OWASP Benchmark Project
- **Website**: https://owasp.org/www-project-benchmark/
- **Community**: Active OWASP community project
- **Documentation**: Official benchmark methodology and standards

## Contribution and Maintenance

### Project Status
The OWASP Benchmark is an active OWASP project maintained by security researchers and tool vendors.

### Key Maintainers
Refer to the project's GitHub repository and OWASP project page for current maintainer information.

### Version History
- **Current Version**: 1.2 (as indicated by `expectedresults-1.2.csv`)
- **Evolution**: Regular updates to include new vulnerability types and patterns

## Conclusion

The **OWASP Benchmark for Java** serves as a critical resource for:
- Objectively measuring security tool effectiveness
- Training security professionals on vulnerability patterns
- Researching automated vulnerability detection techniques
- Evaluating and improving SAST/IAST/DAST tools

Its intentionally vulnerable codebase, standardized structure, and comprehensive test coverage make it the industry standard for security tool benchmarking in the Java ecosystem.