pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-deployer
  containers:
    - name: k8s-tools
      image: alpine/k8s:1.35.3
      command:
        - sleep
      args:
        - 99d
'''
        }
    }

    stages {
        stage('Validate') {
            steps {
                container('k8s-tools') {
                    sh 'helm lint ./nginx-lab'
                }
            }
        }

        stage('Helm Template') {
            steps {
                container('k8s-tools') {
                    sh '''
                        helm template lab-nginx ./nginx-lab \
                          --namespace jenkins-lab
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                container('k8s-tools') {
                    sh '''
                        helm upgrade --install lab-nginx ./nginx-lab \
                          --namespace jenkins-lab \
                          --wait \
                          --timeout 3m
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                container('k8s-tools') {
                    sh '''
                        kubectl rollout status deployment/lab-nginx \
                          --namespace jenkins-lab \
                          --timeout=120s

                        kubectl get deployment,pod,service \
                          --namespace jenkins-lab \
                          --selector app.kubernetes.io/instance=lab-nginx
                    '''
                }
            }
        }
    }
}
