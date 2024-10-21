pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  tools {
    jdk 'JDK 17 OpenJDK'  // Global Tool Configuration'da tanımladığınız JDK ismi
    maven 'Maven 3.9.9'   // Global Tool Configuration'da tanımladığınız Maven ismi
  }
  environment {
    JAVA_HOME = "${tool 'JDK 17 OpenJDK'}"  // JAVA_HOME ortam değişkeni tanımlandı
  }
  stages {
    stage('Check Maven Version') {
      steps {
        sh 'echo $JAVA_HOME'  // JAVA_HOME değerini kontrol et
        sh 'java -version'    // Java sürümünü kontrol et
        sh 'mvn -version'     // Maven sürümünü kontrol et
      }
    }

    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Static Analysis') {  // 'Test' aşaması Static Analysis olarak yeniden adlandırıldı
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {  // SCA aşaması eklendi
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, 
                                artifacts: 'target/dependency-check-report.html', 
                                fingerprint: true,
                                onlyIfSuccessful: true
            }
          }
        }
        stage('OSS License Checker') {  // OSS License Checker aşaması eklendi
          steps {
            container('licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
              /bin/bash --login
              rvm use default
              gem install license_finder
              license_finder
              '''
            }
          }
        }
      }
    }
    stage('SAST') {  // Yeni SAST aşaması Static Analysis'ten sonra eklendi
      steps {
        container('slscan') {
          sh 'scan --type java,depscan --build'
        }
      }
      post {
        success {
          archiveArtifacts allowEmptyArchive: true, 
                            artifacts: 'reports/*', 
                            fingerprint: true, 
                            onlyIfSuccessful: true
        }
      }
    }
    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage('OCI Image BnP') {  // Kaniko ile imaj oluşturma ve yayınlama aşaması
          steps {
            container('kaniko') {
              sh '''
                /kaniko/executor \
                -f `pwd`/Dockerfile \
                -c `pwd` \
                --insecure \
                --skip-tls-verify \
                --cache=true \
                --destination=docker.io/fhkzero/dso-demo
              '''
            }
          }
        }
      }
    }
    stage('Deploy to Dev') {
      steps {
        // TODO
        sh "echo done"
      }
    }
  }
}
