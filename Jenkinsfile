
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
        TOMCAT_CONTAINER = 'webgoat-tomcat'
        TARGET_URL = 'http://webgoat-tomcat:8080/WebGoat'
    }

    stages {

        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Git Secrets (TruffleHog)') {
            steps {
                echo 'Scanning workspace for leaked credentials...'

                sh """
                    docker run --rm \
                        -v jenkins_home:/scan:ro \
                        trufflesecurity/trufflehog:latest \
                        filesystem "/scan/\${WORKSPACE#/var/jenkins_home/}" \
                        --fail \
                        --fail-on-scan-errors
                """
            }
        }

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
                        -r "/zap/wrk/\${WORKSPACE#/var/jenkins_home/}/zap_report.html" \
                        -I
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'zap_report.html',
                fingerprint: true
            )
        }
    }
}
