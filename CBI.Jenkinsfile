pipeline {
  agent {
    kubernetes {
      label 'xtext-build-pod'
      defaultContainer 'xtext-buildenv'
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: 'eclipsecbi/jenkins-jnlp-agent'
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    volumeMounts:
    - mountPath: /home/jenkins/.ssh
      name: volume-known-hosts
  - name: xtext-buildenv
    image: docker.io/smoht/xtext-buildenv:0.7
    tty: true
    volumeMounts:
    - name: settings-xml
      mountPath: /home/jenkins/.m2/settings.xml
      subPath: settings.xml
      readOnly: true
    - name: m2-repo
      mountPath: /home/jenkins/.m2/repository
  volumes:
  - name: volume-known-hosts
    configMap:
      name: known-hosts
  - name: settings-xml
    configMap: 
      name: m2-dir
      items:
      - key: settings.xml
        path: settings.xml
  - name: m2-repo
    emptyDir: {}
    '''
    }
  }
  
  parameters {
    choice(name: 'target_platform', choices: ['oxygen', 'latest', 'r201812', 'r201809', 'r201903', 'photon'], description: 'Which Target Platform should be used?')
  }

  options {
    buildDiscarder(logRotator(numToKeepStr:'5'))
    disableConcurrentBuilds()
  }

  // https://jenkins.io/doc/book/pipeline/syntax/#triggers
  triggers {
    upstream(upstreamProjects: 'xtext-extras/milestone_2.18.0.M3', threshold: hudson.model.Result.SUCCESS)
    pollSCM('H/5 * * * *')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        
        script {
          if (params.target_platform == 'latest') {
            currentBuild.displayName = "#${BUILD_NUMBER}(4.12)"
          } else if (params.target_platform == 'r201903') {
            currentBuild.displayName = "#${BUILD_NUMBER}(4.11)"
          }  else if (params.target_platform == 'r201812') {
            currentBuild.displayName = "#${BUILD_NUMBER}(4.10)"
          } else if (params.target_platform == 'r201809') {
            currentBuild.displayName = "#${BUILD_NUMBER}(4.9)"
          } else if (params.target_platform == 'photon') {
            currentBuild.displayName = "#${BUILD_NUMBER}(4.8)"
          } else {
            currentBuild.displayName = "#${BUILD_NUMBER}(4.7)"
          }
        }

        sh '''
            sed_inplace() {
                if [[ "$OSTYPE" == "darwin"* ]]; then
                    sed -i '' "$@"
                else
                    sed -i "$@"
                fi
            }
            
            targetfiles="$(find releng -type f -iname '*.target')"
            for targetfile in $targetfiles
            do
                echo "Redirecting target platforms in $targetfile to $JENKINS_URL"
                sed_inplace "s?<repository location=\\".*/job/\\([^/]*\\)/job/\\([^/]*\\)/?<repository location=\\"$JENKINS_URL/job/\\1/job/\\2/?" $targetfile
            done
        '''
      }
    }

    stage('Build') {
      steps {
          sh """
            /home/vnc/.vnc/xstartup.sh
            ./1-maven-build.sh -s /home/jenkins/.m2/settings.xml --tp=${params.target_platform} --local-repository=/home/jenkins/.m2/repository
          """
          step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/*.xml'])
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'build/**'
    }
    failure {
      archiveArtifacts artifacts: '**/target/work/data/.metadata/.log, **/hs_err_pid*.log'
    }
    changed {
      script {
        def envName = ''
        if (env.JENKINS_URL.contains('ci.eclipse.org/xtext')) {
          envName = ' (JIPP)'
        } else if (env.JENKINS_URL.contains('jenkins.eclipse.org/xtext')) {
          envName = ' (CBI)'
        } else if (env.JENKINS_URL.contains('typefox.io')) {
          envName = ' (TF)'
        }
        
        def curResult = currentBuild.currentResult
        def color = '#00FF00'
        if (curResult == 'SUCCESS' && currentBuild.previousBuild != null) {
          curResult = 'FIXED'
        } else if (curResult == 'UNSTABLE') {
          color = '#FFFF00'
        } else if (curResult == 'FAILURE') {
          color = '#FF0000'
        }
        
        slackSend message: "${curResult}: <${env.BUILD_URL}|${env.JOB_NAME}#${env.BUILD_NUMBER}${envName}>", botUser: true, channel: 'xtext-builds', color: "${color}"
      }
    }
  }
}
