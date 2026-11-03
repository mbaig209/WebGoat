pipeline {
    agent any
    
    tools {
        jdk 'Java-17'     // Points to your updated /opt/java/openjdk configuration
        maven 'Maven-3.x' // Installs Maven automatically inside the container
    }
    
    environment {
        TOMCAT_CONTAINER = 'webgoat-tomcat'
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
                // Changed from 'bat' to 'sh', and changed Windows path mapping to Linux syntax
                sh "docker run --rm -v \$(pwd):/pwd trufflesecurity/trufflehog:latest github --repo=file:///pwd --fail"
            }
        }

     //   stage('SCA (OWASP Dependency-Check)') {
          //  steps {
            //    echo '📦 Scanning third-party dependencies...'
              //  dependencyCheck additionalArguments: '--scan ./ --format XML --format HTML', odcInstallation: 'OWASP-Dependency-Check'
                //dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
           // }
       // }

     stage('SCA (OWASP Dependency-Check)') {
            steps {
                echo '📦 Running Software Composition Analysis directly from container path...'
                sh "/opt/dependency-check/bin/dependency-check.sh --project WebGoat --scan ./ --format XML --format HTML --out ./"
            }
        }
        stage('Build Artifact') {
            steps {
                echo '🔨 Compiling WebGoat War file...'
                // Changed from 'bat' to 'sh'
                sh "mvn clean package -DskipTests"
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                echo '🚀 Deploying WebGoat to Tomcat Container...'
                // Clean up previous test containers using Linux shell context
                sh """
                docker stop \$TOMCAT_CONTAINER || true
                docker rm \$TOMCAT_CONTAINER || true
                """
                
                script {
                    def warFile = findFiles(glob: 'webgoat-server/target/webgoat-server-*.war')
                    echo "Found war file: ${warFile.path}"
                    
                    sh "docker run -d -p 8080:8080 --name \$TOMCAT_CONTAINER -v \$(pwd)/${warFile.path}:/usr/local/tomcat/webapps/webgoat.war tomcat:9.0"
                }
                
                echo '⏳ Waiting for Tomcat initialization...'
                sh "sleep 20"
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                echo '⚡ Running Dynamic Application Security Testing...'
                sh "docker run --rm -v \$(pwd):/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t \$TARGET_URL -r zap_report.html"
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
        }
    }
}
