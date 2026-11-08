
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
        ENABLE_TRUFFLEHOG_GATE = 'true'
        ENABLE_DEPENDENCY_CHECK_GATE = 'true'
        ENABLE_TRIVY_GATE = 'true'
        ENABLE_ZAP_GATE = 'true'
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
                        -r "/zap/wrk\${WORKSPACE#/var/jenkins_home}/zap_report.html" \
                        -J "/zap/wrk\${WORKSPACE#/var/jenkins_home}/zap_report.json" \
                        -I
                """
            }
        }
    
        // QUALITY GATES: All scanner stages above finish before these gates are evaluated.
        // Each gate is controlled independently by its own ENABLE_*_GATE setting.
        // An enabled gate can fail the build, but cannot prevent the scans above from running.
        // GATE 1: Evaluate Semgrep findings. This current gate reports the count only; it is non-blocking.

stage('Semgrep Quality Gate') {
    when {
        environment name: 'ENABLE_SEMGREP_GATE', value: 'true'
    }
    steps {
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

print("Semgrep ERROR findings:", error_count)
print("Semgrep security report processed successfully.")
'
        '''
    }
}

        // GATE 2: Fail the build if verified secrets are detected.
stage('TruffleHog Quality Gate') {
    when {
                environment name: 'ENABLE_TRUFFLEHOG_GATE', value: 'true'
            }
    steps {
        script {
            def verifiedSecrets = 0
            def reportFile = 'trufflehog-report.jsonl'

            if (!fileExists(reportFile)) {
                error 'TruffleHog report is missing!'
            }

            def report = readFile(reportFile)

            if (report.trim()) {
                report.readLines().each { line ->
                    if (line.trim()) {
                        def finding =
                            new groovy.json.JsonSlurperClassic()
                                .parseText(line)

                        if (finding.Verified == true) {
                            verifiedSecrets++
                        }
                    }
                }
            }

            echo "Verified secrets found: ${verifiedSecrets}"

            if (verifiedSecrets > 0) {
                error "TruffleHog Quality Gate FAILED: ${verifiedSecrets} verified secrets detected!"
            }

            echo 'TruffleHog Quality Gate PASSED.'
        }
    }
}
        // GATE 3: Fail the build if a dependency vulnerability has CVSS 9.0 or higher.
stage('Dependency-Check Quality Gate') {
    when {
                environment name: 'ENABLE_DEPENDENCY_CHECK_GATE', value: 'true'
            }
    steps {
        script {
            def reportFile = 'dependency-check-report.json'

            if (!fileExists(reportFile)) {
                error 'Dependency-Check JSON report is missing!'
            }

            def report = new groovy.json.JsonSlurperClassic()
                .parseText(readFile(reportFile))

            def criticalCount = 0

            report.dependencies?.each { dependency ->
                dependency.vulnerabilities?.each { vulnerability ->
                    def score = vulnerability.cvssv3?.baseScore ?:
                                vulnerability.cvssv2?.score ?: 0

                    if (score >= 9.0) {
                        criticalCount++
                    }
                }
            }

            echo "Dependency-Check Critical vulnerabilities: ${criticalCount}"

            if (criticalCount > 0) {
                error "Dependency-Check Quality Gate FAILED: ${criticalCount} Critical vulnerabilities detected!"
            }

            echo 'Dependency-Check Quality Gate PASSED.'
        }
    }
}
        // GATE 4: Fail the build if Trivy reports any CRITICAL vulnerabilities.
stage('Trivy Quality Gate') {
            when {
                environment name: 'ENABLE_TRIVY_GATE', value: 'true'
            }
    steps {
        script {
            def reportFile = 'trivy-sca-report.json'

            if (!fileExists(reportFile)) {
                error 'Trivy JSON report is missing!'
            }

            def report = new groovy.json.JsonSlurperClassic()
                .parseText(readFile(reportFile))

            def criticalCount = 0

            report.Results?.each { result ->
                result.Vulnerabilities?.each { vulnerability ->
                    if (vulnerability.Severity == 'CRITICAL') {
                        criticalCount++
                    }
                }
            }

            echo "Trivy Critical vulnerabilities: ${criticalCount}"

            if (criticalCount > 0) {
                error "Trivy Quality Gate FAILED: ${criticalCount} Critical vulnerabilities detected!"
            }

            echo 'Trivy Quality Gate PASSED.'
        }
    }
}
        // GATE 5: Fail the build if ZAP reports any High-risk alerts.
stage('ZAP Quality Gate') {
    when {
                environment name: 'ENABLE_ZAP_GATE', value: 'true'
            }
    steps {
        script {
            def reportFile = 'zap_report.json'

            if (!fileExists(reportFile)) {
                error 'ZAP JSON report is missing!'
            }

            def report = new groovy.json.JsonSlurperClassic()
                .parseText(readFile(reportFile))

            def highCount = 0

            report.site?.each { site ->
                site.alerts?.each { alert ->
                    if (alert.riskcode?.toString() == '3') {
                        highCount++
                    }
                }
            }

            echo "ZAP High-risk alerts: ${highCount}"

            if (highCount > 0) {
                error "ZAP Quality Gate FAILED: ${highCount} High-risk alerts detected!"
            }

            echo 'ZAP Quality Gate PASSED.'
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
