pipeline {
  environment {
    ARGO_SERVER = '9.223.203.104'
    DEV_URL = 'http://51.12.128.146:8080/'
  }
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
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
    stage('Test') {
      parallel {
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:12.1.0:check -DnvdApiKey=d1654259-85ff-45f1-a53c-3b212b809c5a'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
              // dependencyCheckPublisher pattern: 'report.xml'
            }
          }
        }
        stage('OSS License Checker') {
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
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
      }
    }
    stage('SAST') {
      steps {
        container('slscan') {
          sh 'scan --type java,depscan --build'
        }
      }
      post {
        success {
          archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful: true
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
        stage('OCI Image BnP') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/elwonen/dso-demo'
            }
          }
        }
      }
    }
    stage('Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/elwonen/dso-demo'
            }
          }
        }
        stage('Image Scan') {
          steps {
            container('docker-tools') {
              sh 'trivy image --timeout 20m --exit-code 1 elwonen/dso-demo'
            }
          }
        }
      }
    }
    stage('Deploy to Dev') {
      environment { 
        AUTH_TOKEN = credentials('argocd-jenkins-deployer-token') 
      }
      steps {
        container('docker-tools') {
          sh 'docker run -t schoolofdevops/argocd-cli argocd app sync dso-demo --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
          sh 'docker run -t schoolofdevops/argocd-cli argocd app wait dso-demo --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
        }
        sh "echo done"
      }
    }
    stage('Dynamic Analysis') {
      parallel {  // <-- Fixed typo: was 'parrallel'
        stage('E2E tests') {
          steps {
            sh 'echo "All Tests Passed!!!!"'
          }
        }
        stage('DAST') {
          steps {
            container('docker-tools') {
              sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t $DEV_URL || exit 0'
              //                                      ^ Added space here
            }
          }  // <-- Added missing closing brace
        }  // <-- Added missing closing brace
      }  // <-- Added missing closing brace for parallel
    }  // <-- Added missing closing brace for stage
  }  // <-- Closing stages
}  // <-- Closing pipeline 
