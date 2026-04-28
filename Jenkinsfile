pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  securityContext:
    fsGroup: 1000
  containers:
  - name: base
    image: alpine:3.20
    command:
    - cat
    tty: true
    securityContext:
      runAsUser: 0
    volumeMounts:
    - name: workspace-volume
      mountPath: /home/jenkins/agent
'''
    }
  }

  stages {
    stage('Branch Info') {
      steps {
        container('base') {
          sh '''
            echo "=== Multi-branch Pipeline Test ==="
            echo "JOB_NAME=$JOB_NAME"
            echo "BRANCH_NAME=$BRANCH_NAME"
            echo "GIT_BRANCH=$GIT_BRANCH"
            echo "CHANGE_ID=$CHANGE_ID"
            echo "WORKSPACE=$WORKSPACE"
            pwd
            ls -al
          '''
        }
      }
    }

    stage('Main Branch Only Check') {
      when {
        branch 'main'
      }
      steps {
        container('base') {
          sh '''
            echo "This stage runs only on main branch."
          '''
        }
      }
    }

    stage('Non-main Branch Check') {
      when {
        not {
          branch 'main'
        }
      }
      steps {
        container('base') {
          sh '''
            echo "This stage runs only on non-main branches."
          '''
        }
      }
    }
  }
}
