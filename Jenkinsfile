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

     // stage('SCA (OWASP Dependency-Check)') {
             // steps {
             //   echo '📦 Running Software Composition Analysis directly from container path...'
           //     sh "/opt/dependency-check/bin/dependency-check.sh --project WebGoat --scan ./ --format XML --format HTML --out ./"
         //   }
       // }
       // stage('Build Artifact') {
          //  steps {
          //      echo '🔨 Compiling WebGoat War file...'
                // Changed from 'bat' to 'sh'
        //        sh "mvn clean package -DskipTests"
      //      }
       // }
stage('Build Artifact') {
            steps {
                echo '🔨 Compiling WebGoat War file with Java 21 targeted configurations...'
                // Changed from release 17 to release 21 to support advanced switch patterns
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
                    
                    // Swapped base image to an active, reliable Eclipse Temurin Java 21 platform
                    sh "docker run -d -p 8080:8080 --name \$TOMCAT_CONTAINER -v jenkins_home:/var/jenkins_home/workspace/WebGoat-DevSecOps-Pipeline/${resolvedJarPath}:/app/webgoat.jar eclipse-temurin:21-jre-alpine java -jar /app/webgoat.jar"
                }
                
                echo '⏳ Waiting for WebGoat initialization...'
                sh "sleep 30"
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
