pipeline {
    agent any
    
    environment {
        TOMCAT_CONTAINER = 'webgoat-tomcat'
        // Updated port from 8080 to 8088 to avoid network binding collisions
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
                sh "docker run --rm -v jenkins_home:/var/jenkins_home trufflesecurity/trufflehog:latest github --repo=file:///var/jenkins_home/workspace/WebGoat-DevSecOps-Pipeline --fail"
            }
        }

      //  stage('SCA (OWASP Dependency-Check)') {
        //    steps {
          //      echo '📦 Running Software Composition Analysis directly from container path...'
            //    sh "/opt/dependency-check/bin/dependency-check.sh --project WebGoat --scan ./ --format XML --format HTML --out ./"
           // }
       // }

        stage('Build Artifact') {
            steps {
                echo '🔨 Compiling WebGoat War file with Java 21 targeted configurations...'
                sh "mvn clean package -DskipTests -Dmaven.compiler.release=21 -Djava.version=21"
            }
        }

        stage('Deploy WebGoat Application') {
            steps {
                echo '🚀 Starting WebGoat Application...'
                sh """
                docker stop \$TOMCAT_CONTAINER || true
                docker rm \$TOMCAT_CONTAINER || true
                """
                
                script {
                    def resolvedJarPath = sh(script: "find . -name 'webgoat-server-*.jar' -o -name 'webgoat-*.jar' | grep -v 'original' | head -n 1", returnStdout: true).trim()
                    echo "Resolved target executable path: ${resolvedJarPath}"
                    
                    if (!resolvedJarPath) {
                        error "Could not find a compiled WebGoat .jar file in the workspace target folders!"
                    }
                    
                    // Changed external host port binding from 8080 to 8088
                    sh "docker create -p 8088:8080 --name \$TOMCAT_CONTAINER eclipse-temurin:21-jre-alpine java -jar /webgoat.jar"
                    
                    sh "docker cp ${resolvedJarPath} \$TOMCAT_CONTAINER:/webgoat.jar"
                    
                    sh "docker start \$TOMCAT_CONTAINER"
                }
                
                echo '⏳ Waiting for WebGoat initialization...'
                sh "sleep 35"
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                echo '⚡ Running Dynamic Application Security Testing...'
                sh "docker run --rm -v jenkins_home:/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t \$TARGET_URL -r zap_report.html || true"
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        }
    }
}
