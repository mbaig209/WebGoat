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
                    
                    // Changed external host port binding from 8080 to 8088
                   // sh "docker create -p 8088:8080 --name \$TOMCAT_CONTAINER eclipse-temurin:21-jre-alpine java -jar /webgoat.jar"
                   // sh "docker create --network devsecops-network -p 8088:8080 --name \$TOMCAT_CONTAINER eclipse-temurin:21-jre-alpine java -jar /webgoat.jar"
                    sh "docker create --network devsecops-network -p 8088:8080 --name \$TOMCAT_CONTAINER eclipse-temurin:21-jre-alpine java -jar /webgoat.jar --server.address=0.0.0.0"
                    sh "docker cp ${resolvedJarPath} \$TOMCAT_CONTAINER:/webgoat.jar"
                    
                    sh "docker start \$TOMCAT_CONTAINER"
                }
                
                echo '⏳ Waiting for WebGoat initialization...'
                sh "sleep 35"
            }
        }

       // stage('DAST (OWASP ZAP)') {
       //     steps {
       //         echo '⚡ Running Dynamic Application Security Testing...'
       //         sh "docker run --rm -v jenkins_home:/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t \$TARGET_URL -r zap_report.html || true"
       //     }
       // }
        stage('DAST - OWASP ZAP') {
    steps {
        sh """
            docker run --rm --user 0:0 --network devsecops-network \
                -v "\$(pwd):/zap/wrk/:rw" \
                ghcr.io/zaproxy/zaproxy:stable \
                zap-baseline.py \
                -t http://webgoat-tomcat:8080/WebGoat \
                -r zap_report.html \
                -I
        """
    }
}
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        }
    }
}
