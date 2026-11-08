
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    tools {
        jdk 'Java-17'
        maven 'Maven-3.x'
    }

    environment {
        ENABLE_SEMGREP_GATE = 'true'
        ENABLE_SONARQUBE_GATE = 'true'
        ENABLE_TRUFFLEHOG_GATE = 'true'
        ENABLE_DEPENDENCY_CHECK_GATE = 'true'
        ENABLE_TRIVY_GATE = 'true'
        ENABLE_ZAP_GATE = 'true'
        ENABLE_JIRA_REPORTING = 'true'
        JIRA_BASE_URL = 'https://codefortix.atlassian.net'
        JIRA_ISSUE_KEY = 'SCRUM-5'
        TOMCAT_CONTAINER = 'webgoat-tomcat'
        TARGET_URL = 'http://webgoat-tomcat:8080/WebGoat'
    }

    stages {

        // STAGE 1: Clean the workspace and check out the source code.
stage('Git Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }
        // STAGE 2: Run Semgrep static analysis and save the findings as JSON.
stage('SAST - Semgrep') {
          
            steps {
                echo 'Scanning WebGoat source code with Semgrep...'

                sh '''
                    docker run --rm \
                        -v jenkins_home:/src:rw \
                        semgrep/semgrep \
                        semgrep scan \
                        --config p/java \
                        --metrics=off \
                        --json \
                        --output "/src${WORKSPACE#/var/jenkins_home}/semgrep-sast-report.json" \
                        "/src${WORKSPACE#/var/jenkins_home}"
                '''

                sh 'ls -lh semgrep-sast-report.json'
            }
        }
        // STAGE 3: Scan the source tree for exposed credentials and secrets.
stage('Git Secrets (Trufflehog)') {
            steps {
                echo 'Scanning source code for leaked credentials...'

                sh '''
                    docker run --rm \
                        -v jenkins_home:/scan:ro \
                        trufflesecurity/trufflehog:latest \
                        --json \
                        filesystem "/scan${WORKSPACE#/var/jenkins_home}" \
                        --results=verified,unknown \
                        > trufflehog-report.jsonl
                '''

                sh 'ls -lh trufflehog-report.jsonl'
            }
        }
        // STAGE 4: Compile and package WebGoat for analysis and deployment.
stage('Build Artifact') {
            steps {
                echo 'Building WebGoat using Java 21 settings...'

                sh """
                    mvn clean package \
                        -DskipTests \
                        -Dmaven.compiler.release=21 \
                        -Djava.version=21
                """
            }
        }
        // STAGE 5: Run SonarQube static code analysis on the Maven project.
stage('SAST - SonarQube') {
    steps {
        echo 'Running SonarQube code analysis...'

        withSonarQubeEnv('SonarQube') {
            sh '''
                mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.0.0.4389:sonar \
                    -Dsonar.projectKey=WebGoat \
                    -Dsonar.projectName=WebGoat \
                    -Dsonar.java.binaries=target/classes
            '''
        }
    }
}
        // STAGE 6: Verify authenticated connectivity to the SonarQube API.
stage('SonarQube API Connectivity') {
    steps {
        withCredentials([
            string(
                credentialsId: 'sonarqube-token',
                variable: 'SONAR_TOKEN'
            )
        ]) {
            sh '''
                docker run --rm \
                    --network devsecops-network \
                    -e SONAR_TOKEN="$SONAR_TOKEN" \
                    curlimages/curl:latest \
                    -fsS \
                    -u "$SONAR_TOKEN:" \
                    "http://sonarqube:9000/api/issues/search?componentKeys=WebGoat&ps=1" \
                    -o /dev/null
            '''

            echo 'SonarQube API connectivity verified.'
        }
    }
}
        // STAGE 7: Export SonarQube issues to a JSON report.
stage('SonarQube - Generate JSON Report') {
    steps {
        withCredentials([
            string(
                credentialsId: 'sonarqube-token',
                variable: 'SONAR_TOKEN'
            )
        ]) {
            sh '''
               docker run --rm \
    --user 1000:1000 \
    --network devsecops-network \
                    -v jenkins_home:/data:rw \
                    -e SONAR_TOKEN="$SONAR_TOKEN" \
                    curlimages/curl:latest \
                    -fsS \
                    -u "$SONAR_TOKEN:" \
                    "http://sonarqube:9000/api/issues/search?componentKeys=WebGoat&types=VULNERABILITY&ps=500" \
                    -o "/data${WORKSPACE#/var/jenkins_home}/sonarqube-report.json"
            '''
        }

        sh 'ls -lh sonarqube-report.json'
    }
}
        // STAGE 8: Render the SonarQube JSON export as a readable HTML report.
stage('SonarQube - Generate HTML Report') {
    steps {
        echo 'Generating SonarQube HTML report...'

        sh '''
            docker run --rm \
                --user 1000:1000 \
                -v jenkins_home:/data \
                -w "/data${WORKSPACE#/var/jenkins_home}" \
                python:3.12-alpine \
                python -c '
import json, html
from collections import Counter

with open("sonarqube-report.json") as f:
    data = json.load(f)

issues = data.get("issues", [])
counts = Counter(i.get("severity", "UNKNOWN") for i in issues)

rows = "".join(
    "<tr><td>" + html.escape(str(i.get("severity", ""))) +
    "</td><td>" + html.escape(str(i.get("type", ""))) +
    "</td><td>" + html.escape(str(i.get("component", "")).split(":")[-1]) +
    "</td><td>" + str(i.get("line", "N/A")) +
    "</td><td>" + html.escape(str(i.get("message", ""))) +
    "</td></tr>"
    for i in issues
)

summary = "".join(
    "<li>" + html.escape(k) + ": " + str(v) + "</li>"
    for k, v in sorted(counts.items())
)

report = """<html><head><meta charset="UTF-8">
<title>WebGoat SonarQube Report</title>
<style>
body{font-family:Arial;margin:30px}
table{border-collapse:collapse;width:100%}
th,td{border:1px solid #ddd;padding:9px;text-align:left}
th{background:#263b59;color:white}
tr:nth-child(even){background:#f2f2f2}
</style></head><body>
<h1>WebGoat -SonarQube Security Vulnerability Report</h1>
<h2>Security Vulnerability Summary</h2><ul>""" + summary + """</ul>
<h2>Security Vulnerability Details</h2>
<table><tr><th>Severity</th><th>Type</th><th>File</th>
<th>Line</th><th>Message</th></tr>""" + rows + """</table>
</body></html>"""

with open("sonarqube-report.html", "w") as f:
    f.write(report)

print("HTML report generated:", len(issues), "issues")
'
        '''

        sh 'ls -lh sonarqube-report.html'
    }
}
        // STAGE 9: Generate a CycloneDX Software Bill of Materials for dependency analysis.
stage('Generate CycloneDX SBOM') {
            steps {
                echo 'Generating CycloneDX Software Bill of Materials...'

                sh '''
                    docker run --rm \
                        -v jenkins_home:/workspace \
                        -v jenkins_home:/jenkins-data \
                        -w "/workspace${WORKSPACE#/var/jenkins_home}" \
                        maven:3.9-eclipse-temurin-21 \
                        mvn \
                        "-Dmaven.repo.local=/jenkins-data/.m2/repository" \
                        "org.cyclonedx:cyclonedx-maven-plugin:2.9.1:makeAggregateBom" \
                        "-DoutputFormat=json" \
                        "-DexcludeTestProject=false" \
                        "-DskipTests"
                '''

                sh 'ls -lh target/bom.json'
            }
        }
        // STAGE 10: Scan project dependencies against the NVD vulnerability database.
stage('SCA - OWASP Dependency-Check') {
    steps {
        echo 'Scanning WebGoat dependencies for known vulnerabilities...'

        withCredentials([
            string(
                credentialsId: 'NVD_API_KEY',
                variable: 'NVD_API_KEY'
            )
        ]) {
            sh '''
                docker run --rm \
                    -v jenkins_home:/src:rw \
                    -v dependency-check-data:/usr/share/dependency-check/data \
                    -e NVD_API_KEY="$NVD_API_KEY" \
                    owasp/dependency-check:latest \
                    --scan "/src${WORKSPACE#/var/jenkins_home}" \
                    --format JSON \
                    --out "/src${WORKSPACE#/var/jenkins_home}/dependency-check-report.json" \
                    --nvdApiKey "$NVD_API_KEY"
            '''
        }

        sh 'ls -lh dependency-check-report.json'
    }
}
        // STAGE 11: Scan the CycloneDX SBOM with Trivy.
stage('SCA - Trivy Dependency Scan') {
            steps {
                echo 'Scanning CycloneDX SBOM for dependency vulnerabilities...'

                sh '''
                    docker run --rm \
                        -v jenkins_home:/scan:rw \
                        aquasec/trivy:latest \
                        sbom \
                        --scanners vuln \
                        --format json \
                        --output "/scan${WORKSPACE#/var/jenkins_home}/trivy-sca-report.json" \
                        "/scan${WORKSPACE#/var/jenkins_home}/target/bom.json"
                '''

                sh 'ls -lh trivy-sca-report.json'
            }
        }
        // STAGE 12: Deploy the built WebGoat JAR in the Docker network for dynamic testing.
stage('Deploy WebGoat Application') {
            steps {
                echo 'Starting WebGoat Application...'

                sh """
                    docker stop \$TOMCAT_CONTAINER || true
                    docker rm \$TOMCAT_CONTAINER || true
                """

                script {
                    def resolvedJarPath = sh(
                        script: "find ./target -name 'webgoat-*.jar' | grep -v 'original' | head -n 1",
                        returnStdout: true
                    ).trim()

                    if (!resolvedJarPath) {
                        error 'Could not find the compiled WebGoat JAR!'
                    }

                    echo "Resolved JAR: ${resolvedJarPath}"

                    sh """
                        docker create \
                            --network devsecops-network \
                            -p 8088:8080 \
                            --name \$TOMCAT_CONTAINER \
                            eclipse-temurin:21-jre-alpine \
                            java -jar /webgoat.jar \
                            --server.address=0.0.0.0
                    """

                    sh "docker cp ${resolvedJarPath} \$TOMCAT_CONTAINER:/webgoat.jar"

                    sh "docker start \$TOMCAT_CONTAINER"
                }

                echo 'Waiting for WebGoat initialization...'
                sh 'sleep 35'
            }
        }
        // STAGE 13: Run the OWASP ZAP Baseline DAST scan and save HTML/JSON reports.
stage('DAST - OWASP ZAP') {
            steps {
                echo 'Running OWASP ZAP Baseline Scan...'

                sh """
                    docker run --rm \
                        --user 0:0 \
                        --network devsecops-network \
                        -v jenkins_home:/zap/wrk:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t "\$TARGET_URL" \
                       -r "\${WORKSPACE#/var/jenkins_home}/zap_report.html" \
                        -J "\${WORKSPACE#/var/jenkins_home}/zap_report.json" \
                        -I
                """
            }
        }
    
        // QUALITY GATES: All scanner stages above finish before these gates are evaluated.
        // Each gate is controlled independently by its own ENABLE_*_GATE setting.
        // An enabled gate can fail the build, but cannot prevent the scans above from running.
        


// GATE 1: Evaluate Semgrep findings.
// Fail the gate when High-impact vulnerabilities are detected.

stage('Semgrep Quality Gate') {
    when {
        environment name: 'ENABLE_SEMGREP_GATE',
                  value: 'true'
    }

    steps {
        catchError(
            buildResult: 'FAILURE',
            stageResult: 'FAILURE'
        ) {
            echo 'Evaluating Semgrep security findings...'

            sh '''
                docker run --rm \
                    --user 1000:1000 \
                    -v jenkins_home:/data:ro \
                    -w "/data${WORKSPACE#/var/jenkins_home}" \
                    python:3.12-alpine \
                    python -c '
import json

with open("semgrep-sast-report.json") as f:
    report = json.load(f)

results = report.get("results", [])

error_count = sum(
    1 for item in results
    if item.get("extra", {}).get("severity") == "ERROR"
)

high_count = sum(
    1 for item in results
    if str(
        item.get("extra", {})
            .get("metadata", {})
            .get("impact", "")
    ).upper() == "HIGH"
)

medium_count = sum(
    1 for item in results
    if str(
        item.get("extra", {})
            .get("metadata", {})
            .get("impact", "")
    ).upper() == "MEDIUM"
)

print("Semgrep ERROR findings:", error_count)
print("Semgrep HIGH-impact findings:", high_count)
print("Semgrep MEDIUM-impact findings:", medium_count)

if high_count > 0:
    raise SystemExit(
        "Semgrep Quality Gate FAILED: "
        + str(high_count)
        + " High-impact findings detected!"
    )

print("Semgrep Quality Gate PASSED.")
'
            '''
        }
    }
}


          // GATE 2: Evaluate SonarQube findings. This current gate reports the Blockers and Critical vulnerabilities; Only issues classified as VULNERABILITY should be considered. We are not evaluating general code quality or maintainability.
        
stage('SonarQube Quality Gate') {

    when {
        environment name: 'ENABLE_SONARQUBE_GATE',
                  value: 'true'
    }

    steps {
        catchError(
            buildResult: 'FAILURE',
            stageResult: 'FAILURE'
        ) {

            echo 'Evaluating SonarQube security vulnerabilities...'

            sh '''
                docker run --rm \
                    --user 1000:1000 \
                    -v jenkins_home:/data:ro \
                    -w "/data${WORKSPACE#/var/jenkins_home}" \
                    python:3.12-alpine \
                    python -c '
import json

report_file = "sonarqube-report.json"

with open(report_file) as f:
    report = json.load(f)

issues = report.get("issues", [])

# Consider only security vulnerabilities
vulnerabilities = [
    issue for issue in issues
    if issue.get("type") == "VULNERABILITY"
]

blocker_count = sum(
    1 for issue in vulnerabilities
    if issue.get("severity") == "BLOCKER"
)

critical_count = sum(
    1 for issue in vulnerabilities
    if issue.get("severity") == "CRITICAL"
)

major_count = sum(
    1 for issue in vulnerabilities
    if issue.get("severity") == "MAJOR"
)

minor_count = sum(
    1 for issue in vulnerabilities
    if issue.get("severity") == "MINOR"
)

print("SonarQube Vulnerability Summary")
print("--------------------------------")
print("BLOCKER vulnerabilities:", blocker_count)
print("CRITICAL vulnerabilities:", critical_count)
print("MAJOR vulnerabilities:", major_count)
print("MINOR vulnerabilities:", minor_count)

if blocker_count > 0 or critical_count > 0:
    raise SystemExit(
        "SonarQube Quality Gate FAILED: "
        "Blocker or Critical vulnerabilities detected."
    )

print("SonarQube Quality Gate PASSED.")
'
            '''
        }
    }
}


        // GATE 3: Fail the build if verified secrets are detected.

stage('TruffleHog Quality Gate') {
    when {
        environment name: 'ENABLE_TRUFFLEHOG_GATE', value: 'true'
    }
    steps {
        catchError(
            buildResult: 'FAILURE',
            stageResult: 'FAILURE'
        ) {
            echo 'Evaluating TruffleHog security findings...'

            sh '''
                docker run --rm \
                    --user 1000:1000 \
                    -v jenkins_home:/data:ro \
                    -w "/data${WORKSPACE#/var/jenkins_home}" \
                    python:3.12-alpine \
                    python -c '
import json

report_file = "trufflehog-report.jsonl"

verified_count = 0

with open(report_file) as f:
    for line in f:
        if not line.strip():
            continue

        finding = json.loads(line)

        if finding.get("Verified") is True:
            verified_count += 1

print("TruffleHog Verified secrets:", verified_count)

if verified_count > 0:
    raise SystemExit(
        "TruffleHog Quality Gate FAILED: "
        + str(verified_count)
        + " verified secrets detected!"
    )

print("TruffleHog Quality Gate PASSED.")
'
            '''
        }
    }
}
        // GATE 4: Fail the build if a dependency vulnerability has CVSS 9.0 or higher.

stage('Dependency-Check Quality Gate') {
    when {
        environment name: 'ENABLE_DEPENDENCY_CHECK_GATE', value: 'true'
    }
    steps {
          catchError(
        buildResult: 'FAILURE',
        stageResult: 'FAILURE'
    ) {
        echo 'Evaluating Dependency-Check security findings...'

        sh '''
            docker run --rm \
                --user 1000:1000 \
                -v jenkins_home:/data:ro \
                -w "/data${WORKSPACE#/var/jenkins_home}" \
                python:3.12-alpine \
                python -c '
import json

report_file = "dependency-check-report.json"

with open(report_file) as f:
    report = json.load(f)

critical_count = 0

for dependency in report.get("dependencies", []):
    for vulnerability in dependency.get("vulnerabilities", []):
        cvssv3 = vulnerability.get("cvssv3") or {}
        cvssv2 = vulnerability.get("cvssv2") or {}

        score = cvssv3.get("baseScore") or cvssv2.get("score") or 0

        if float(score) >= 9.0:
            critical_count += 1

print("Dependency-Check Critical vulnerabilities:", critical_count)

if critical_count > 0:
    raise SystemExit(
        "Dependency-Check Quality Gate FAILED: "
        + str(critical_count)
        + " Critical vulnerabilities detected!"
    )

print("Dependency-Check Quality Gate PASSED.")
'
        '''
    }
}
}
        // GATE 5: Fail the build if Trivy reports any CRITICAL vulnerabilities.

stage('Trivy Quality Gate') {
    when {
        environment name: 'ENABLE_TRIVY_GATE', value: 'true'
    }
    steps {
          catchError(
            buildResult: 'FAILURE',
            stageResult: 'FAILURE')
        {
        echo 'Evaluating Trivy security findings...'

        sh '''
            docker run --rm \
                --user 1000:1000 \
                -v jenkins_home:/data:ro \
                -w "/data${WORKSPACE#/var/jenkins_home}" \
                python:3.12-alpine \
                python -c '
import json

report_file = "trivy-sca-report.json"

with open(report_file) as f:
    report = json.load(f)

critical_count = 0

for result in report.get("Results", []):
    for vulnerability in result.get("Vulnerabilities") or []:
        if vulnerability.get("Severity") == "CRITICAL":
            critical_count += 1

print("Trivy Critical vulnerabilities:", critical_count)

if critical_count > 0:
    raise SystemExit(
        "Trivy Quality Gate FAILED: "
        + str(critical_count)
        + " Critical vulnerabilities detected!"
    )

print("Trivy Quality Gate PASSED.")
'
        '''
    }
}
}
        // GATE 6: Fail the build if ZAP reports any High-risk alerts.

stage('ZAP Quality Gate') {
    when {
        environment name: 'ENABLE_ZAP_GATE', value: 'true'
    }
    steps {
          catchError(
        buildResult: 'FAILURE',
        stageResult: 'FAILURE'
    ) {
        echo 'Evaluating OWASP ZAP security findings...'

        sh '''
            docker run --rm \
                --user 1000:1000 \
                -v jenkins_home:/data:ro \
                -w "/data${WORKSPACE#/var/jenkins_home}" \
                python:3.12-alpine \
                python -c '
import json

report_file = "zap_report.json"

with open(report_file) as f:
    report = json.load(f)

high_count = 0

for site in report.get("site", []):
    for alert in site.get("alerts", []):
        if str(alert.get("riskcode", "")) == "3":
            high_count += 1

print("ZAP High-risk alerts:", high_count)

if high_count > 0:
    raise SystemExit(
        "ZAP Quality Gate FAILED: "
        + str(high_count)
        + " High-risk alerts detected!"
    )

print("ZAP Quality Gate PASSED.")
'
        '''
    }
}

    }
        // STAGE 14: Publish scan findings and report links to the Jira tracking issue.
        
        // STAGE 14: Create individual Jira defects for Blocker, Critical and High findings.
        stage('Jira Defect Creation') {
            when {
                environment name: 'ENABLE_JIRA_REPORTING', value: 'true'
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'jira-webgoat-credentials',
                            usernameVariable: 'JIRA_EMAIL',
                            passwordVariable: 'JIRA_API_TOKEN'
                        )
                    ]) {
                        sh '''
                            docker run --rm -i \
                                --network devsecops-network \
                                -v jenkins_home:/data:ro \
                                -w "/data${WORKSPACE#/var/jenkins_home}" \
                                -e JIRA_EMAIL="$JIRA_EMAIL" \
                                -e JIRA_API_TOKEN="$JIRA_API_TOKEN" \
                                -e JIRA_BASE_URL="$JIRA_BASE_URL" \
                                -e BUILD_NUMBER="$BUILD_NUMBER" \
                                -e BUILD_URL="$BUILD_URL" \
                                -e JOB_NAME="$JOB_NAME" \
                                python:3.12-alpine \
                                python - <<'PY'
import os
import json
import base64
import hashlib
import urllib.request
import urllib.error
import urllib.parse
from pathlib import Path

BASE = os.environ["JIRA_BASE_URL"].rstrip("/")
PROJECT = "SCRUM"
ISSUE_TYPE = "Bug"

EMAIL = os.environ["JIRA_EMAIL"]
TOKEN = os.environ["JIRA_API_TOKEN"]

BUILD = os.getenv("BUILD_NUMBER", "unknown")
BUILD_URL = os.getenv("BUILD_URL", "")
JOB = os.getenv("JOB_NAME", "unknown")

AUTH = base64.b64encode(
    (EMAIL + ":" + TOKEN).encode()
).decode()

HEADERS = {
    "Authorization": "Basic " + AUTH,
    "Accept": "application/json",
    "Content-Type": "application/json"
}

findings = {}


def read_json(filename):
    path = Path(filename)
    if not path.exists():
        print("Report unavailable:", filename)
        return None

    with path.open(encoding="utf-8") as f:
        return json.load(f)


def add_finding(scanner, severity, title, details, stable_key):
    if severity not in ("BLOCKER", "CRITICAL", "HIGH"):
        return

    fingerprint = hashlib.sha256(
        (scanner + "|" + stable_key).encode()
    ).hexdigest()[:20]

    findings[fingerprint] = {
        "scanner": scanner,
        "severity": severity,
        "title": str(title)[:180],
        "details": str(details)[:12000],
        "fingerprint": fingerprint
    }


def jira_request(method, endpoint, payload=None):
    url = BASE + endpoint
    data = None

    if payload is not None:
        data = json.dumps(payload).encode()

    request = urllib.request.Request(
        url,
        data=data,
        headers=HEADERS,
        method=method
    )

    with urllib.request.urlopen(request, timeout=45) as response:
        raw = response.read().decode()
        return json.loads(raw) if raw else {}


def existing_issue(fingerprint):
    label = "sec-fp-" + fingerprint

    jql = (
        'project = "' + PROJECT + '" AND labels = "' +
        label + '"'
    )

    result = jira_request(
        "POST",
        "/rest/api/3/search/jql",
        {
            "jql": jql,
            "maxResults": 1,
            "fields": ["key"]
        }
    )

    issues = result.get("issues", [])
    return issues[0]["key"] if issues else None


def create_issue(item):
    label = "sec-fp-" + item["fingerprint"]

    summary = (
        "[" + item["severity"] + "] [" +
        item["scanner"] + "] " + item["title"]
    )[:250]

    description_text = (
        "Automated security finding detected by Jenkins.\n\n"
        "Scanner: " + item["scanner"] + "\n"
        "Severity: " + item["severity"] + "\n"
        "Build: #" + BUILD + "\n"
        "Job: " + JOB + "\n"
        "Build URL: " + BUILD_URL + "\n"
        "Fingerprint: " + item["fingerprint"] + "\n\n"
        "Finding details:\n" + item["details"]
    )

    payload = {
        "fields": {
            "project": {"key": PROJECT},
            "issuetype": {"name": ISSUE_TYPE},
            "summary": summary,
            "description": {
                "type": "doc",
                "version": 1,
                "content": [
                    {
                        "type": "paragraph",
                        "content": [
                            {
                                "type": "text",
                                "text": description_text
                            }
                        ]
                    }
                ]
            },
            "labels": [
                "security",
                "automated",
                "webgoat",
                label
            ]
        }
    }

    result = jira_request(
        "POST",
        "/rest/api/3/issue",
        payload
    )

    return result.get("key", "created-without-key")


# --------------------------------------------------
# 1. SEMGREP
# --------------------------------------------------

data = read_json("semgrep-sast-report.json")

if data:
    for item in data.get("results", []):
        extra = item.get("extra", {})
        metadata = extra.get("metadata", {})

        impact = str(metadata.get("impact", "")).upper()

        if impact != "HIGH":
            continue

        path = item.get("path", "Unknown file")
        line = item.get("start", {}).get("line", "N/A")
        rule = item.get("check_id", "Unknown rule")
        message = extra.get("message", "")

        add_finding(
            "Semgrep",
            "HIGH",
            rule,
            "File: " + path +
            "\nLine: " + str(line) +
            "\nRule: " + rule +
            "\nFinding: " + str(message),
            rule + "|" + path + "|" + str(line)
        )


# --------------------------------------------------
# 2. SONARQUBE
# --------------------------------------------------

data = read_json("sonarqube-report.json")

if data:
    for item in data.get("issues", []):
        if item.get("type") != "VULNERABILITY":
            continue

        severity_map = {
            "BLOCKER": "BLOCKER",
            "CRITICAL": "CRITICAL"
        }

        severity = severity_map.get(item.get("severity"))

        if not severity:
            continue

        rule = item.get("rule", "Unknown rule")
        component = item.get("component", "Unknown component")
        line = item.get("line", "N/A")
        message = item.get("message", "")

        add_finding(
            "SonarQube",
            severity,
            rule,
            "Rule: " + rule +
            "\nComponent: " + component +
            "\nLine: " + str(line) +
            "\nFinding: " + str(message),
            item.get("key", rule + "|" + component + "|" + str(line))
        )


# --------------------------------------------------
# 3. TRIVY
# --------------------------------------------------

data = read_json("trivy-sca-report.json")

if data:
    for result in data.get("Results", []):
        target = result.get("Target", "Unknown target")

        for vuln in result.get("Vulnerabilities") or []:
            severity = str(vuln.get("Severity", "")).upper()

            if severity not in ("CRITICAL", "HIGH"):
                continue

            vuln_id = vuln.get("VulnerabilityID", "Unknown ID")
            package = vuln.get("PkgName", "Unknown package")
            version = vuln.get("InstalledVersion", "Unknown")
            fixed = vuln.get("FixedVersion", "Not specified")

            add_finding(
                "Trivy",
                severity,
                vuln_id + " - " + package,
                "Target: " + target +
                "\nPackage: " + package +
                "\nInstalled version: " + str(version) +
                "\nFixed version: " + str(fixed) +
                "\nVulnerability ID: " + vuln_id,
                target + "|" + vuln_id + "|" + package + "|" + str(version)
            )


# --------------------------------------------------
# 4. DEPENDENCY-CHECK
# --------------------------------------------------

data = read_json("dependency-check-report.json")

if data:
    for dependency in data.get("dependencies", []):
        package = dependency.get("fileName", "Unknown dependency")

        for vuln in dependency.get("vulnerabilities", []):
            cvssv3 = vuln.get("cvssv3") or {}
            cvssv2 = vuln.get("cvssv2") or {}

            score = float(
                cvssv3.get("baseScore") or
                cvssv2.get("score") or 0
            )

            if score >= 9.0:
                severity = "CRITICAL"
            elif score >= 7.0:
                severity = "HIGH"
            else:
                continue

            vuln_id = vuln.get("name", "Unknown CVE")
            description = vuln.get("description", "")

            add_finding(
                "Dependency-Check",
                severity,
                vuln_id + " - " + package,
                "Dependency: " + package +
                "\nVulnerability: " + vuln_id +
                "\nCVSS score: " + str(score) +
                "\nDescription: " + str(description),
                package + "|" + vuln_id
            )


# --------------------------------------------------
# 5. OWASP ZAP
# --------------------------------------------------

data = read_json("zap_report.json")

if data:
    for site in data.get("site", []):
        site_name = site.get("name", "Unknown site")

        for alert in site.get("alerts", []):
            if str(alert.get("riskcode", "")) != "3":
                continue

            alert_name = alert.get("alert", "Unknown alert")
            plugin_id = alert.get("pluginid", "")
            instances = alert.get("instances", [])

            locations = []

            for instance in instances[:10]:
                locations.append(
                    str(instance.get("uri", "")) +
                    " | Method: " + str(instance.get("method", ""))
                )

            add_finding(
                "OWASP ZAP",
                "HIGH",
                alert_name,
                "Site: " + site_name +
                "\nAlert: " + alert_name +
                "\nDescription: " + str(alert.get("desc", "")) +
                "\nSolution: " + str(alert.get("solution", "")) +
                "\nInstances:\n" + "\n".join(locations),
                site_name + "|" + str(plugin_id) + "|" + alert_name
            )


# --------------------------------------------------
# 6. TRUFFLEHOG
# --------------------------------------------------

secret_path = Path("trufflehog-report.jsonl")

if secret_path.exists():
    with secret_path.open(encoding="utf-8") as f:
        for raw in f:
            if not raw.strip():
                continue

            item = json.loads(raw)

            if item.get("Verified") is not True:
                continue

            detector = item.get("DetectorName", "Unknown detector")
            metadata = item.get("SourceMetadata", {})
            filesystem = metadata.get("Data", {}).get("Filesystem", {})

            source_file = filesystem.get("file", "Unknown file")
            line = filesystem.get("line", "N/A")
            commit = item.get("SourceID", "Unknown source")

            # Never include Raw, RawV2 or secret values in Jira.
            add_finding(
                "TruffleHog",
                "HIGH",
                "Verified secret detected - " + detector,
                "Detector: " + detector +
                "\nFile: " + str(source_file) +
                "\nLine: " + str(line) +
                "\nSource reference: " + str(commit) +
                "\nA verified secret was detected. Secret value intentionally excluded.",
                detector + "|" + str(source_file) + "|" + str(line) + "|" + str(commit)
            )


# --------------------------------------------------
# 7. CREATE JIRA ISSUES WITHOUT DUPLICATES
# --------------------------------------------------

print("")
print("======================================")
print("JIRA SECURITY DEFECT SYNCHRONIZATION")
print("======================================")
print("Eligible findings:", len(findings))

created = 0
duplicates = 0

for fingerprint, item in findings.items():
    try:
        old_issue = existing_issue(fingerprint)

        if old_issue:
            print(
                "Existing Jira defect:",
                old_issue,
                "|",
                item["scanner"],
                "|",
                item["severity"]
            )
            duplicates += 1
            continue

        issue_key = create_issue(item)

        print(
            "Created Jira defect:",
            issue_key,
            "|",
            item["scanner"],
            "|",
            item["severity"]
        )

        created += 1

    except urllib.error.HTTPError as exc:
        print(
            "Jira API error for",
            item["scanner"],
            item["fingerprint"],
            "HTTP",
            exc.code
        )
        print(exc.read().decode(errors="replace"))
        raise

print("")
print("======================================")
print("JIRA DEFECT SYNCHRONIZATION COMPLETED")
print("New defects created:", created)
print("Existing defects skipped:", duplicates)
print("======================================")
PY
                        '''
                    }
                }
            }
        }

        stage('Jira Reporting') {
            when {
                environment name: 'ENABLE_JIRA_REPORTING', value: 'true'
            }
            steps {
                // A Jira outage must not change the security scan build result.
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    script {
                        env.JIRA_BUILD_RESULT = currentBuild.currentResult ?: 'UNKNOWN'
                    }

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'jira-webgoat-credentials',
                            usernameVariable: 'JIRA_EMAIL',
                            passwordVariable: 'JIRA_API_TOKEN'
                        )
                    ]) {
                        sh '''
                            docker run --rm -i \
                                --network devsecops-network \
                                -v jenkins_home:/data:ro \
                                -w "/data${WORKSPACE#/var/jenkins_home}" \
                                -e JIRA_EMAIL="$JIRA_EMAIL" \
                                -e JIRA_API_TOKEN="$JIRA_API_TOKEN" \
                                -e JIRA_BASE_URL="$JIRA_BASE_URL" \
                                -e JIRA_ISSUE_KEY="$JIRA_ISSUE_KEY" \
                                -e JIRA_BUILD_RESULT="$JIRA_BUILD_RESULT" \
                                -e BUILD_NUMBER="$BUILD_NUMBER" \
                                -e BUILD_URL="$BUILD_URL" \
                                -e JOB_NAME="$JOB_NAME" \
                                python:3.12-alpine \
                                python - <<'PY'
import base64
import json
import os
import urllib.error
import urllib.request
from pathlib import Path

def read_json(name):
    path = Path(name)
    if not path.exists():
        return None
    with path.open(encoding="utf-8") as f:
        return json.load(f)

def status(failed):
    return "FAILED" if failed else "PASSED"

lines = [
    "WebGoat DevSecOps Security Scan Report",
    "Job: " + os.getenv("JOB_NAME", "unknown"),
    "Build: #" + os.getenv("BUILD_NUMBER", "unknown"),
    "Build result: " + os.getenv("JIRA_BUILD_RESULT", "UNKNOWN"),
    "Build URL: " + os.getenv("BUILD_URL", "unavailable"),
    "",
    "Scanner findings and quality gates:"
]

    data = read_json("semgrep-sast-report.json")

if data is None:
    lines.append("Semgrep: report missing")
else:
    results = data.get("results", [])

    error_count = sum(
        1 for x in results
        if x.get("extra", {}).get("severity") == "ERROR"
    )

    high_count = sum(
        1 for x in results
        if str(
            x.get("extra", {}).get("metadata", {}).get("impact", "")
        ).upper() == "HIGH"
    )

    medium_count = sum(
        1 for x in results
        if str(
            x.get("extra", {}).get("metadata", {}).get("impact", "")
        ).upper() == "MEDIUM"
    )

    lines.append(
        "Semgrep: ERROR=" + str(error_count) +
        ", HIGH impact=" + str(high_count) +
        ", MEDIUM impact=" + str(medium_count) +
        "; gate=" + status(high_count > 0)
    )

data = read_json("sonarqube-report.json")
if data is None:
    lines.append("SonarQube: report missing")
else:
    vulns = [x for x in data.get("issues", [])
             if x.get("type") == "VULNERABILITY"]
    counts = {s: sum(1 for x in vulns if x.get("severity") == s)
              for s in ("BLOCKER", "CRITICAL", "MAJOR", "MINOR")}
    lines.append(
        "SonarQube vulnerabilities: BLOCKER={BLOCKER}, CRITICAL={CRITICAL}, "
        "MAJOR={MAJOR}, MINOR={MINOR}; gate={gate}".format(
            **counts, gate=status(counts["BLOCKER"] > 0 or counts["CRITICAL"] > 0))
    )

secret_path = Path("trufflehog-report.jsonl")
if not secret_path.exists():
    lines.append("TruffleHog: report missing")
else:
    verified = 0
    with secret_path.open(encoding="utf-8") as f:
        for raw in f:
            if raw.strip() and json.loads(raw).get("Verified") is True:
                verified += 1
    lines.append("TruffleHog: " + str(verified) +
                 " verified secrets; gate=" + status(verified > 0))

data = read_json("dependency-check-report.json")
if data is None:
    lines.append("Dependency-Check: report missing")
else:
    count = 0
    for dep in data.get("dependencies", []):
        for vuln in dep.get("vulnerabilities", []):
            v3, v2 = vuln.get("cvssv3") or {}, vuln.get("cvssv2") or {}
            if float(v3.get("baseScore") or v2.get("score") or 0) >= 9.0:
                count += 1
    lines.append("Dependency-Check: " + str(count) +
                 " findings with CVSS >= 9.0; gate=" + status(count > 0))

data = read_json("trivy-sca-report.json")
if data is None:
    lines.append("Trivy: report missing")
else:
    count = sum(1 for r in data.get("Results", [])
                for v in (r.get("Vulnerabilities") or [])
                if v.get("Severity") == "CRITICAL")
    lines.append("Trivy: " + str(count) +
                 " CRITICAL vulnerabilities; gate=" + status(count > 0))

data = read_json("zap_report.json")
if data is None:
    lines.append("OWASP ZAP: report missing")
else:
    count = sum(1 for site in data.get("site", [])
                for alert in site.get("alerts", [])
                if str(alert.get("riskcode", "")) == "3")
    lines.append("OWASP ZAP: " + str(count) +
                 " High-risk alerts; gate=" + status(count > 0))

build_url = os.getenv("BUILD_URL", "")
lines += [
    "",
    "Archived reports (available after build completion):",
    build_url + "artifact/semgrep-sast-report.json",
    build_url + "artifact/sonarqube-report.html",
    build_url + "artifact/trufflehog-report.jsonl",
    build_url + "artifact/dependency-check-report.json",
    build_url + "artifact/trivy-sca-report.json",
    build_url + "artifact/zap_report.html"
]

# Jira Cloud REST API v3 comment bodies use Atlassian Document Format (ADF).
content = []
for line in lines:
    content.append({
        "type": "paragraph",
        "content": ([{"type": "text", "text": line}] if line else [])
    })
payload = {"body": {"type": "doc", "version": 1, "content": content}}

email = os.environ["JIRA_EMAIL"]
token = os.environ["JIRA_API_TOKEN"]
base = os.environ["JIRA_BASE_URL"].rstrip("/")
issue = os.environ["JIRA_ISSUE_KEY"]
url = base + "/rest/api/3/issue/" + issue + "/comment"
auth = base64.b64encode((email + ":" + token).encode()).decode()
request = urllib.request.Request(
    url,
    data=json.dumps(payload).encode(),
    headers={
        "Authorization": "Basic " + auth,
        "Accept": "application/json",
        "Content-Type": "application/json"
    },
    method="POST"
)

try:
    with urllib.request.urlopen(request, timeout=30) as response:
        result = json.loads(response.read().decode())
        print("Jira comment created successfully for", issue)
        print("Jira comment ID:", result.get("id", "not returned"))
except urllib.error.HTTPError as exc:
    print("Jira API returned HTTP", exc.code)
    print(exc.read().decode(errors="replace"))
    raise SystemExit(1)
except Exception as exc:
    print("Jira reporting failed:", str(exc))
    raise SystemExit(1)
PY
                        '''
                    }
                }
            }
        }
    }
post {
    always {
        archiveArtifacts(
            artifacts: 'dependency-check-report.json,semgrep-sast-report.json,sonarqube-report.json,sonarqube-report.html,zap_report.html,zap_report.json,trufflehog-report.jsonl,target/bom.json,trivy-sca-report.json',
            fingerprint: true,
            allowEmptyArchive: true
        )
    }
}
}
