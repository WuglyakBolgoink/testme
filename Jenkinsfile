pipeline {
  agent any

  options {
          disableConcurrentBuilds()
      }

  stages {

  stage('Check global module') {
        steps {
          script {
            sh "java -version"
            sh "cordova --version"
            sh "android --version"
            sh "node --version"
            sh "npm --version"
          }
          }
          }



    stage('Initialize') {
      steps {
        script {
                            // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - Set ENV variables
                            env.LOGICAL_BRANCH_NAME = env.BRANCH_NAME
                            env.BUILD_TIER = "integ"

                            if (env.BRANCH_NAME == 'release/release') {
                                env.BUILD_TIER = "stage"
                                env.LOGICAL_BRANCH_NAME = 'staging'
                            } else if (env.BRANCH_NAME == 'master') {
                                env.BUILD_TIER = "prod"
                            }

                            env.SERVICE_ID = "test-me"
                            env.SERVICE_VERSION = "${env.LOGICAL_BRANCH_NAME}.${env.BUILD_NUMBER}"

                            // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - Outputs

                            echo "BRANCH_NAME: $env.BRANCH_NAME"
                            echo "LOGICAL_BRANCH_NAME: $env.LOGICAL_BRANCH_NAME"
                            echo "BUILD_TIER: $env.BUILD_TIER"
                            echo "SERVICE_ID: $env.SERVICE_ID"
                            echo "SERVICE_VERSION: $env.SERVICE_VERSION"

                            // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - GIT
                            sh "pwd"
                            sh "ls -la"
                            // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - GIT
                            sh "git clean -f && git reset --hard origin/$env.BRANCH_NAME"
                        }
      }
    }


    stage('Cordova init') {
                steps {
                    sh "cordova prepare"
                }
            }


  }

   post {
          success {
              emailext(
                  subject: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]'",
                  body: """<p>SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]':</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}@[${
                      env.LOGICAL_BRANCH_NAME
                  }]]</a>&QUOT;</p>""",
                  recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                  mimeType: 'text/html'
              )
          }
          failure {
              emailext(
                  subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]'",
                  body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${
                      env.LOGICAL_BRANCH_NAME
                  }]':</p><hr><p>Build trigger: $cause</p><hr>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}@[${
                      env.LOGICAL_BRANCH_NAME
                  }]]</a>&QUOT;</p>""",
                  recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                  mimeType: 'text/html'
              )
          }
          unstable {
              emailext(
                  subject: "UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]'",
                  body: """<p>UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]':</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}@[${
                      env.LOGICAL_BRANCH_NAME
                  }]]</a>&QUOT;</p>""",
                  recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                  mimeType: 'text/html'
              )
          }
      }
}
