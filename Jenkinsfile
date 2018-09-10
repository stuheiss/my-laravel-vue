pipeline {
    agent {
        label "jenkins-php"
    }
    environment {
      ORG               = 'stuheiss'
      APP_NAME          = 'my-laravel-vue'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('php') {
            sh "python -m unittest"

            sh 'export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml'


            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }

          dir ('./charts/preview') {
           container('php') {
             sh "make preview"
             sh "jx preview --app $APP_NAME --dir ../.."
           }
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          container('php') {
            // ensure we're not on a detached head
            sh "git checkout master"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
          }
          dir ('./charts/my-laravel-vue') {
            container('php') {
              sh "make tag"
            }
          }
          container('php') {
            sh "python -m unittest"

            sh 'export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml'

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/my-laravel-vue') {
            container('php') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'

              // release the helm chart
              sh 'jx step helm release'

              // promote through all 'Auto' promotion Environments
              sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
            }
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
    }
  }
