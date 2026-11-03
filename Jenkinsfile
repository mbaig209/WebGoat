pipeline {
    agent any
    
    tools {
        jdk 'Java-17'     // Matches the JDK name you configured earlier
        maven 'Maven-3.x' // Matches the Maven name you just configured
    }
    
    environment {
        TOMCAT_CONTAINER = 'webgoat-tomcat'
        // host.docker.internal allows the ZAP container to reach the Tomcat container on the host network
        TARGET_URL       = 'http://docker.internal'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Git Secrets (Trufflehog)') {
            steps {
                echo '🕵️‍♂️ Scanning for leaked credentials...'
                // Scans the codebase for keys/secrets. Assumes git history exists in workspace.
                bat "docker run --rm -v %WORKSPACE%:/pwd trufflesecurity/trufflehog:latest github --repo=file:///pwd --fail"
            }
        }

        stage('SCA (OWASP Dependency-Check)') {
            steps {
                echo '📦 Scanning third-party dependencies...'
                // Scans the pom.xml ecosystem for known vulnerabilities
                dependencyCheck additionalArguments: '--scan ./ --format XML --format HTML', odcInstallation: 'OWASP-Dependency-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                echo '🔨 Compiling WebGoat War file...'
                // WebGoat requires Java 17+. Skip unit tests to speed up the security pipeline build.
                bat "mvn clean package -DskipTests"
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WebGoat to Tomcat Container...'
                // Clean up previous test containers to prevent port collisions
                bat """
                docker stop %TOMCAT_CONTAINER% || true
                docker rm %TOMCAT_CONTAINER% || true
                """
                
                // Mount WebGoat's compiled .war file directly into a Tomcat 9 container instance
                // Note: WebGoat builds the artifact in webgoat-server/target/webgoat-*.war
                bat "docker run -d -p 8080:8080 --name %TOMCAT_CONTAINER% -v %WORKSPACE%/webgoat-server/target/webgoat-server-8.2.2.war:/usr/local/tomcat/webapps/webgoat.war tomcat:9.0"
                
                echo '⏳ Waiting for Tomcat initialization...'
                bat "timeout /t 20"
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                echo '⚡ Running Dynamic Application Security Testing...'
                // Using the baseline script to intercept vulnerabilities on the newly deployed app endpoint
                bat "docker run --rm -v %WORKSPACE%:/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t %TARGET_URL% -r zap_report.html"
            }
        }
    }
    
    post {
        always {
            // Publishes the web application scan findings directly to your Jenkins dashboard
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
        }
    }
}
