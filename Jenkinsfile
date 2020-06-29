def LABEL_ID = "questcod-${UUID.randomUUID().toString()}"

podTemplate(
    label: LABEL_ID, 
    containers: [
        containerTemplate(args: 'cat', name: 'docker-container', command: '/bin/sh -c', image: 'docker', ttyEnabled: true),
        containerTemplate(args: 'cat', name: 'helm-container', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:latest', ttyEnabled: true)
    ],
    volumes: [
      hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
    ]
){
    def REPOS
    def IMAGE_VERSION
    def IMAGE_NAME = "questcode-backend-user"
    def ENVIRONMENT = "staging"
    def GIT_URL = "git@github.com:sandrocaetano/questcode-backend-user.git"
    def CHARTMUSEUM_URL = "http://chartmuseum-lab-chartmuseum:8080"
    def DEPLOY_NAME = "questcode-backend-user"
    def DEPLOY_CHART = "actarlab/questcode-backend-user"
    def NODE_PORT = "30020"


    node(LABEL_ID) {
        stage('Checkout') {
            echo 'Iniciando Clone do Repositorio'
            REPOS = checkout scm
            GIT_BRANCH = REPOS.GIT_BRANCH

            if(GIT_BRANCH.equals("master")) {
                KUBE_NAMESPACE = "production"
            } else if(GIT_BRANCH.equals("develop")) {
                KUBE_NAMESPACE = "staging"
                NODE_PORT = "31020"
            } else {
                def error = "Nao existe pipeline para a branch ${GIT_BRANCH}"
                echo error
                throw new Exception(error)
            }

            IMAGE_VERSION = sh returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION = IMAGE_VERSION.trim()
        }
        stage('Static Analysis') {
            nodejs('node14') { 
                parallel (
                    SCA: {
                        sh 'export COMMIT_ID=`cat .git/HEAD`'
                        sh 'export DCHIGH=20'
                        sh 'export DCMEDIUM=100'

                        sh 'npm install'
                    },
                    SCA2: {
                        dependencyCheck(additionalArguments: '''
                            -o Dependency-Check''',
                        odcInstallation: 'Default')
                        dependencyCheckPublisher unstableTotalAll: '0'
                    },
                    SAST: {
                        echo  'FindSecBugs'
                    }
                )
            }
        }
        stage('Package') {
            container('docker-container') {
                echo 'Iniciando Empacotamento com Docker'
                withCredentials([usernamePassword(credentialsId: 'dockerhubid', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} ."
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
                }
            }
        }
        stage('Container Analysis') {
            parallel(
                CAC: {
                    echo 'Compliance as Code'                     
                },
                CS: {
                    echo 'Container Scanning'                     
                }
            )
        }
    }
    node(LABEL_ID) {
        stage('Deploy') {
            container('helm-container') {
                echo 'Iniciando o Deploy com Helm'
                sh "helm repo add actarlab ${CHARTMUSEUM_URL}"
                sh 'helm repo update'
                sh "helm upgrade --install ${DEPLOY_NAME} ${DEPLOY_CHART} --set image.tag=${IMAGE_VERSION}  -n ${KUBE_NAMESPACE}"
            }
        }
        stage('Dynamic Analysis') {
            parallel(
                UAT:  {
                    echo 'Vulnerability Assessment'
                },
                DAST: {
                    echo 'Dynamic Application Security Testing'
                },
                VA: {
                    echo 'Vulnerability Assessment'
                }                  
            )
        }     
        stage('WAF') {
            echo 'WAF'
        }     
    }
}
