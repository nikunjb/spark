pipeline {
  agent { node { label 'docker-pipeline' } }

  triggers { pollSCM('H/15 * * * *') }

  environment {
    MODULE_NAME = 'spark'
    REGISTRY_ID = '739344182600'
    REGISTRY_REGION = 'us-west-2'
    ECR_IMAGE_NAME = "${REGISTRY_ID}.dkr.ecr.${REGISTRY_REGION}.amazonaws.com/cequence/spark"
    GITLAB_IMAGE_NAME = "registry.gitlab.com/cequence/cqai/images/spark"
    GITLAB_REGISTRY_CREDS = credentials('gitlab-publish-cqai-images')
  }

  stages {
    stage('Version') {
      steps {
        script {
          shortCommit = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
          env.GIT_TIMESTAMP = sh(returnStdout: true, script: "git log --max-count=1 --format=%ci HEAD").trim()
          env.BUILD_VERSION = shortCommit
          currentBuild.displayName = "${env.BUILD_NUMBER}, $shortCommit"
        }
      }
    }

    stage('Build') {
      when { not { branch '*-cq' } }

      agent {
        docker {
          image 'maven:3.5.4-jdk-8'
          label 'docker-pipeline'
        }
      }

      steps {
        sh 'wget -O scala-2.13.0.tgz https://downloads.lightbend.com/scala/2.13.0/scala-2.13.0.tgz'
        sh 'tar xf scala-2.13.0.tgz'
        sh 'export SCALA_HOME=scala-2.13.0'
        sh 'export PATH=$PATH:$SCALA_HOME/bin'
        sh 'build/mvn --batch-mode -Pkubernetes -Phadoop-2.7 -Pyarn -Pflume clean package -DskipTests'

        stash name: 'build'
      }
    }

    stage('Build & Publish') {
      when { branch '*-cq' }

      agent {
        docker {
          image 'maven:3.5.4-jdk-8'
          label 'docker-pipeline'
        }
      }

      options {
        withAWS(region: 'us-west-2')
      }

      steps {
        sh 'wget -O scala-2.13.0.tgz https://downloads.lightbend.com/scala/2.13.0/scala-2.13.0.tgz'
        sh 'tar xf scala-2.13.0.tgz'
        sh 'export SCALA_HOME=scala-2.13.0'
        sh 'export PATH=$PATH:$SCALA_HOME/bin'
        sh 'build/mvn --batch-mode -Pkubernetes -Phadoop-2.7 -Pyarn -Pflume clean deploy -DskipTests'

        stash name: 'build'
      }
    }

    stage('Make Distribution') {
      agent {
        docker {
          image 'maven:3.5.4-jdk-8'
          label 'docker-pipeline'
        }
      }

      steps {
        unstash 'build'
        sh 'dev/make-distribution.sh --tgz -Pkubernetes -Phadoop-2.7 -Pyarn -Pflume'

        stash name: "dist", includes: 'dist/**/*'
        stash name: 'tar', includes: 'spark-*-bin-*.tgz'
      }
    }

    stage('Publish to S3') {
      agent { label 'docker-pipeline' }

      options {
        withAWS(region: 'us-west-2')
      }

      steps {
        unstash 'tar'

        s3Upload(
                path: "build/${MODULE_NAME}/${GIT_BRANCH}/",
                bucket: 'xangent-packages',
                includePathPattern: 'spark-*-bin-*.tgz'
        )
      }
    }
    stage('Publish to Docker Registries') {
      agent { label 'docker-pipeline' }
      steps {
        unstash 'dist'

        script {
          dir('dist') {
            script {
              def GITLAB_BUILD_IMAGE_NAME = "${GITLAB_IMAGE_NAME}/${env.BRANCH_NAME}:latest".toLowerCase()
              if (env.BRANCH_NAME.endsWith("-cq")) {
                latestTag = "-t ${ECR_IMAGE_NAME}:${env.BRANCH_NAME}-latest"
              } else {
                latestTag = ""
              }

              sh """
              docker build . -t ${GITLAB_BUILD_IMAGE_NAME} -t ${ECR_IMAGE_NAME}:${BUILD_VERSION} ${latestTag} -f kubernetes/dockerfiles/spark/Dockerfile             
    
              \$(aws ecr get-login --region ${REGISTRY_REGION} --registry-ids ${REGISTRY_ID} --no-include-email)
              docker login -u ${GITLAB_REGISTRY_CREDS_USR} -p ${GITLAB_REGISTRY_CREDS_PSW} registry.gitlab.com
              docker push ${GITLAB_BUILD_IMAGE_NAME}
              docker push ${ECR_IMAGE_NAME}
              """
            }
          }
        }
      }
    }
  }
  post {
    always {
      script {
        commits = currentBuild.changeSets.size()
        color = (currentBuild.result == 'SUCCESS') ? '#00FF00' : '#FF0000'
      }

      slackSend color: color, message: "${currentBuild.result} [ ${env.JOB_NAME} ] [${env.GIT_BRANCH}] [Commits: ${commits} ]  (<${env.BUILD_URL}|[ ${currentBuild.displayName} ]>) finished in ${currentBuild.durationString.replace(' and counting', '')}"
    }
  }
}
