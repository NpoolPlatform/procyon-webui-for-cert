pipeline {
  agent any
  stages {
    stage('Clone') {
      steps {
        git(url: scm.userRemoteConfigs[0].url, branch: '$BRANCH_NAME', changelog: true, credentialsId: 'KK-github-key', poll: true)
      }
    }

    stage('Compile') {
      when {
        expression { BUILD_TARGET == 'true' }
      }
      steps {
        sh (returnStdout: false, script: '''
          /root/.nvm/versions/node/v16.0.0/bin/npm install --global yarn
          /root/.nvm/versions/node/v16.0.0/bin/yarn
          /usr/local/bin/quasar build
        '''.stripIndent())
      }
    }

    stage('Switch to current cluster') {
      when {
        anyOf {
          expression { BUILD_TARGET == 'true' }
          expression { DEPLOY_TARGET == 'true' }
        }
      }
      steps {
        sh 'cd /etc/kubeasz; ./ezctl checkout $TARGET_ENV'
      }
    }

    stage('Generate docker image for development') {
      when {
        expression { BUILD_TARGET == 'true' }
      }
      steps {
        sh 'docker build -t $DOCKER_REGISTRY/entropypool/procyon-webui-cert:latest .'
      }
    }

    stage('Generate docker image for testing or production') {
      when {
        expression { BUILD_TARGET == 'true' }
      }
      steps {
        sh(returnStdout: true, script: '''
          revlist=`git rev-list --tags --max-count=1`
          tag=`git describe --tags $revlist`
          git reset --hard
          git checkout $tag
          /root/.nvm/versions/node/v16.0.0/bin/npm install --global yarn
          /root/.nvm/versions/node/v16.0.0/bin/yarn
          /usr/local/bin/quasar build
          docker build -t $DOCKER_REGISTRY/entropypool/procyon-webui-cert:$tag .
        '''.stripIndent())
      }
    }

    stage('Release docker image for development') {
      when {
        expression { RELEASE_TARGET == 'true' }
      }
      steps {
        sh 'docker push $DOCKER_REGISTRY/entropypool/procyon-webui-cert:latest'
        sh(returnStdout: true, script: '''
          images=`docker images | grep entropypool | grep procyon-webui-cert | grep none | awk '{ print $3 }'`
          for image in $images; do
            docker rmi $image -f
          done
        '''.stripIndent())
      }
    }

    stage('Release docker image for testing') {
      when {
        expression { RELEASE_TARGET == 'true' }
      }
      steps {
        sh(returnStdout: false, script: '''
          revlist=`git rev-list --tags --max-count=1`
          tag=`git describe --tags $revlist`

          set +e
          docker images | grep procyon-webui-cert | grep $tag
          rc=$?
          set -e
          if [ 0 -eq $rc ]; then
            docker push $DOCKER_REGISTRY/entropypool/procyon-webui-cert:$tag
          fi
        '''.stripIndent())
      }
    }

    stage('Release docker image for production') {
      when {
        expression { RELEASE_TARGET == 'true' }
      }
      steps {
        sh(returnStdout: false, script: '''
          revlist=`git rev-list --tags --max-count=1`
          tag=`git describe --tags $revlist`

          major=`echo $tag | awk -F '.' '{ print $1 }'`
          minor=`echo $tag | awk -F '.' '{ print $2 }'`
          patch=`echo $tag | awk -F '.' '{ print $3 }'`

          patch=$(( $patch - $patch % 2 ))
          tag=$major.$minor.$patch

          set +e
          docker images | grep procyon-webui-cert | grep $tag
          rc=$?
          set -e
          if [ 0 -eq $rc ]; then
            docker push $DOCKER_REGISTRY/entropypool/procyon-webui-cert:$tag
          fi
        '''.stripIndent())
      }
    }

    stage('Deploy for development') {
      when {
        expression { DEPLOY_TARGET == 'true' }
        expression { TARGET_ENV ==~ /.*development.*/ }
      }
      steps {
        sh 'sed -i "s/uhub.service.ucloud.cn/$DOCKER_REGISTRY/g" k8s/01-procyon-webui-cert.yaml'
        sh 'kubectl apply -k k8s'
      }
    }

    stage('Deploy for testing') {
      when {
        expression { DEPLOY_TARGET == 'true' }
        expression { TARGET_ENV ==~ /.*testing.*/ }
      }
      steps {
        sh(returnStdout: true, script: '''
          revlist=`git rev-list --tags --max-count=1`
          tag=`git describe --tags $revlist`

          git reset --hard
          git checkout $tag
          sed -i "s/procyon-webui-cert:latest/procyon-webui-cert:$tag/g" k8s/01-procyon-webui.yaml
          sed -i "s/uhub.service.ucloud.cn/$DOCKER_REGISTRY/g" k8s/01-procyon-webui.yaml
          kubectl apply -k k8s
        '''.stripIndent())
      }
    }

    stage('Deploy for production') {
      when {
        expression { DEPLOY_TARGET == 'true' }
        expression { TARGET_ENV ==~ /.*production.*/ }
      }
      steps {
        sh(returnStdout: true, script: '''
          revlist=`git rev-list --tags --max-count=1`
          tag=`git describe --tags $revlist`

          major=`echo $tag | awk -F '.' '{ print $1 }'`
          minor=`echo $tag | awk -F '.' '{ print $2 }'`
          patch=`echo $tag | awk -F '.' '{ print $3 }'`
          patch=$(( $patch - $patch % 2 ))
          tag=$major.$minor.$patch

          git reset --hard
          git checkout $tag
          sed -i "s/procyon-webui-cert:latest/procyon-webui-cert:$tag/g" k8s/01-procyon-webui.yaml
          sed -i "s/uhub.service.ucloud.cn/$DOCKER_REGISTRY/g" k8s/01-procyon-webui.yaml
          kubectl apply -k k8s
        '''.stripIndent())
      }
    }

    stage('Post') {
      steps {
        // Assemble vet and lint info.
        // warnings parserConfigurations: [
        //   [pattern: 'govet.txt', parserName: 'Go Vet'],
        //   [pattern: 'golint.txt', parserName: 'Go Lint']
        // ]

        // sh 'go2xunit -fail -input gotest.txt -output gotest.xml'
        // junit "gotest.xml"
        sh 'echo Posting'
      }
    }
  }
  post('Report') {
    fixed {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh fixed')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    success {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh successful')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    failure {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh failure')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    aborted {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh aborted')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
  }
}
