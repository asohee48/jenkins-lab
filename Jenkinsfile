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

    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: '존재하지 않는 nginx 이미지 태그를 사용해 Deploy Stage 실패를 재현합니다.'
        )
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
                    script {
                        def imageOverride = params.SIMULATE_FAILURE ? '--set-string image.tag=step-11-image-does-not-exist' : ''

                        sh """
                            helm upgrade --install lab-nginx ./nginx-lab \\
                              --namespace jenkins-lab \\
                              --reset-values \\
                              ${imageOverride} \\
                              --wait \\
                              --timeout 3m
                        """
                    }
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

    post {
        failure {
            container('k8s-tools') {
                sh '''
                    echo '=== Helm release status ==='
                    helm status lab-nginx --namespace jenkins-lab || true

                    echo '=== Kubernetes workloads ==='
                    kubectl get deployment,pod \
                      --namespace jenkins-lab \
                      --selector app.kubernetes.io/instance=lab-nginx \
                      -o wide || true

                    echo '=== Pod details ==='
                    kubectl describe pod \
                      --namespace jenkins-lab \
                      --selector app.kubernetes.io/instance=lab-nginx || true

                    echo '=== Recent namespace events ==='
                    kubectl get events \
                      --namespace jenkins-lab \
                      --sort-by=.lastTimestamp | tail -n 30 || true
                '''
            }
        }
    }
}
