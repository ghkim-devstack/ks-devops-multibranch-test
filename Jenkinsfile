pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: ks-devops-host-deployer
  securityContext:
    fsGroup: 1000
  containers:
  - name: kubectl
    image: alpine/k8s:1.31.2
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
        container('kubectl') {
          sh '''
            echo "=== Multi-branch Pipeline ==="
            echo "JOB_NAME=$JOB_NAME"
            echo "BRANCH_NAME=$BRANCH_NAME"
            echo "GIT_BRANCH=$GIT_BRANCH"
            echo "WORKSPACE=$WORKSPACE"
          '''
        }
      }
    }

    stage('Feature branch check only') {
      when {
        not {
          branch 'main'
        }
      }
      steps {
        container('kubectl') {
          sh '''
            echo "This is a feature branch."
            echo "No deployment will be executed."
          '''
        }
      }
    }

    stage('Prepare member kubeconfig') {
      when {
        branch 'main'
      }
      steps {
        container('kubectl') {
          withCredentials([usernamePassword(
            credentialsId: 'member-devops-deployer-kubeconfig-b64',
            usernameVariable: 'DUMMY_USER',
            passwordVariable: 'MEMBER_KUBECONFIG_B64'
          )]) {
            sh '''
              echo "=== create host in-cluster kubeconfig ==="

              HOST_TOKEN="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
              HOST_CA="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"

              kubectl config set-cluster host \
                --server=https://kubernetes.default.svc \
                --certificate-authority="${HOST_CA}" \
                --embed-certs=true \
                --kubeconfig=host-kubeconfig.yaml

              kubectl config set-credentials ks-devops-host-deployer \
                --token="${HOST_TOKEN}" \
                --kubeconfig=host-kubeconfig.yaml

              kubectl config set-context host \
                --cluster=host \
                --user=ks-devops-host-deployer \
                --namespace=devops-nginx-host-test \
                --kubeconfig=host-kubeconfig.yaml

              kubectl config use-context host \
                --kubeconfig=host-kubeconfig.yaml

              echo "=== decode member kubeconfig ==="
              set +x
              printf "%s" "$MEMBER_KUBECONFIG_B64" | base64 -d > member-kubeconfig.yaml
              chmod 600 member-kubeconfig.yaml
              set -x

              echo "=== check permissions ==="
              kubectl --kubeconfig=host-kubeconfig.yaml auth can-i create deployments -n devops-nginx-host-test
              kubectl --kubeconfig=member-kubeconfig.yaml auth can-i create deployments -n devops-nginx-member-test
            '''
          }
        }
      }
    }

    stage('Deploy nginx to host cluster') {
      when {
        branch 'main'
      }
      steps {
        container('kubectl') {
          sh '''
            cat <<EOF | kubectl --kubeconfig=host-kubeconfig.yaml apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: devops-nginx-host-test
  labels:
    app: nginx
    target-cluster: host
    managed-by: kubesphere-multibranch-pipeline
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      target-cluster: host
  template:
    metadata:
      labels:
        app: nginx
        target-cluster: host
        managed-by: kubesphere-multibranch-pipeline
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: devops-nginx-host-test
spec:
  type: ClusterIP
  selector:
    app: nginx
    target-cluster: host
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF

            kubectl --kubeconfig=host-kubeconfig.yaml -n devops-nginx-host-test rollout status deployment/nginx --timeout=120s
            kubectl --kubeconfig=host-kubeconfig.yaml -n devops-nginx-host-test get deploy,rs,pod,svc -o wide
          '''
        }
      }
    }

    stage('Deploy nginx to member cluster') {
      when {
        branch 'main'
      }
      steps {
        container('kubectl') {
          sh '''
            cat <<EOF | kubectl --kubeconfig=member-kubeconfig.yaml apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: devops-nginx-member-test
  labels:
    app: nginx
    target-cluster: member
    managed-by: kubesphere-multibranch-pipeline
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      target-cluster: member
  template:
    metadata:
      labels:
        app: nginx
        target-cluster: member
        managed-by: kubesphere-multibranch-pipeline
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: devops-nginx-member-test
spec:
  type: ClusterIP
  selector:
    app: nginx
    target-cluster: member
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF

            kubectl --kubeconfig=member-kubeconfig.yaml -n devops-nginx-member-test rollout status deployment/nginx --timeout=120s
            kubectl --kubeconfig=member-kubeconfig.yaml -n devops-nginx-member-test get deploy,rs,pod,svc -o wide
          '''
        }
      }
    }
  }
}
