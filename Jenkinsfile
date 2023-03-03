pipeline {
  environment {
    APP_VER = "v1.0.${BUILD_ID}"
    HARBOR_URL = "imagerepo.big.go.id"
    HARBOR_CREDS = credentials('harbor')
    IMAGE_NAME = "${HARBOR_URL}/big/samples/spring-petclinic:${APP_VER}"
    DEPLOY_GITREPO_USER = "cusex-btech"    
    DEPLOY_GITREPO_URL = "github.com/${DEPLOY_GITREPO_USER}/spring-petclinic-helmchart.git"
    DEPLOY_GITREPO_BRANCH = "main"
    DEPLOY_GITREPO_TOKEN = credentials('btech-dummy-github-token')
  }
  tools {
    maven 'maven-poc'
  }
  agent any
  stages {
    stage('Checkout code') {
      steps {
        checkout scm
      }
    }
    stage('Build') {
      steps {
        echo sh(script: 'env|sort', returnStdout: true)
        sh """
          mvn -B -ntp -T 2 package -DskipTests -DAPP_VERSION=${APP_VER}
          """
      }
    }
    stage('Test') {
      parallel {
        stage(' Unit/Integration Tests') {
          steps {
              sh """
                mvn -B -ntp -T 2 test -DAPP_VERSION=${APP_VER}
              """
            jacoco ( 
              execPattern: 'target/*.exec',
              classPattern: 'target/classes',
              sourcePattern: 'src/main/java',
              exclusionPattern: 'src/test*'
            )
          }
          post {
            always {
              archiveArtifacts artifacts: 'target/**/*.jar', fingerprint: true
              junit 'target/surefire-reports/**/*.xml'
            }
          } 
        }
      }
    }
    stage('Containerize') {
      steps {
        sh """
        docker login ${env.HARBOR_URL} -u="${HARBOR_CREDS_USR}" -p="${HARBOR_CREDS_PSW}"
        docker build -t ${IMAGE_NAME} .
        docker push ${IMAGE_NAME}
        """
      }
    }
    stage('Approval') {
      input {
        message "Proceed to deploy?"
        ok "YES"
      }
      steps {
        echo "Update helm chart to trigger GitOps-based deployment..."
      }
    }    
    stage('GitOps-based Deploy') {
      steps {
        sh """
          rm -rf deploy/
          git config --global user.name $env.GIT_AUTHOR_NAME
          git config --global user.email $env.GIT_AUTHOR_EMAIL
          git clone https://$env.DEPLOY_GITREPO_USER:$env.DEPLOY_GITREPO_TOKEN@$env.DEPLOY_GITREPO_URL --branch=$env.DEPLOY_GITREPO_BRANCH deploy
          # After cloning
          cd deploy
          # update values.yaml
          sed -i -r 's,repository: (.+),repository: ${env.HARBOR_URL}/samples/spring-petclinic,' values.yaml
          sed -i 's/tag: v1.0.*/tag: v1.0.${env.BUILD_ID}/' values.yaml
          cat values.yaml
          git commit -am 'bump up version number'
          git remote set-url origin https://$env.DEPLOY_GITREPO_USER:$env.DEPLOY_GITREPO_TOKEN@$env.DEPLOY_GITREPO_URL
          git push origin main
        """
      }
    }   
  }
}
