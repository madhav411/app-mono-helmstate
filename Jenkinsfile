def INSTALL = ""
pipeline {
    // If you are running jenkins in a container use "agent { docker { image 'dtzar/helm-kubectl:2.11.0' }}"
	agent {
		docker {
			image 'dtzar/helm-kubectl:2.11.0'
		}
	}

    environment {
        KUBECONFIG = 'kubeconfig'
        HELM_CHARTS_GIT_REPO = 'github.com/madhav411/app-mono-helmcharts.git'
        HELM_CHARTS_REPO = 'app-mono-helmcharts'
        HELM_CHARTS_BRANCH = 'master'
        GITHUB_HOOK_SECRET = "acfd0fbb53fb81bab63efbb2c49c60af53ced2b8"
        K8S_SERVER = credentials('K8S_SERVER')
        // K8S_TILLER_TOKEN = credentials('k8s-tiller-token')
        K8S_CA_BASE64 = credentials('K8S_CA')
        GIT = credentials('madhav_github_credentials')
        
    }

    stages {
        stage('configure webook') {
            steps {
                script {
                    setupWebhook()
	        }
	    }
        }

        stage('Create kubeconfig') {
            steps {
                container('deploy') {
                    createKubeconfig()
                }
            }
        }

        stage('Checkout scm') {
            steps {
                checkout scm
            }
        }

        stage('Find app name to deploy') {
            steps {
                script {
                    if (REF != "") {
                        env.APP = VALUESFILE.split('/')[0]
                    }
                }
                echo "application to deploy:${APP}"
            }
        }

        stage('Get helm-charts') {
            steps {
                getHelmCharts()
            }
        }

        stage('Deploy helm chart') {
            steps {
                container('deploy') {
                    script {
                        INSTALL = sh (
                            script: "helm status ${APP}",
                            returnStatus: true
                        )
                    }
                    echo "To install? ${INSTALL}"
                    helmInstall(INSTALL)
                }
            }
        }
    }
    post {
        cleanup {
            deleteDir()
        }
    }
}

def createKubeconfig() {
    sh '''
    kubectl config set-cluster k8s-cluster --server=$K8S_SERVER --certificate-authority=$K8S_CA_BASE64
    sed -i 's/certificate-authority/certificate-authority-data/g' $KUBECONFIG
    kubectl config set-credentials tiller --token=$K8S_TILLER_TOKEN
    kubectl config set-context tiller --cluster=k8s-cluster --user=tiller
    kubectl config use-context tiller
    '''
}

def getHelmCharts() {
    sh '''
    git clone https://${GIT_USR}:${GIT_PSW}@${HELM_CHARTS_GIT_REPO}
    cd ${HELM_CHARTS_REPO}
    git checkout ${HELM_CHARTS_BRANCH}
    '''
    }

def helmInstall(install) {
    sh 'echo deploying application:$APP'
    sh 'helm init --client-only --home /tmp'
    if (install) {
        sh 'helm install --home /tmp --name $APP -f $APP/values.yaml ${HELM_CHARTS_REPO}/charts/$APP/'
    } else {
        sh 'helm upgrade --home /tmp $APP -f $APP/values.yaml ${HELM_CHARTS_REPO}/charts/$APP/'
    }
}


def setupWebhook() {
    properties([
        pipelineTriggers([
            [$class: 'GenericTrigger',
                genericVariables: [
                    [key: 'REF', value: '$.ref'],
                    [key: 'VALUESFILE', value: '$.commits[0].modified[0]'],
                ],
                causeString: 'Triggered on github push',
                token: env.GITHUB_HOOK_SECRET,
                printContributedVariables: true,
                printPostContent: true,
                regexpFilterText: '$REF',
                regexpFilterExpression: 'refs/heads/master',
                regexpFilterText: '$VALUESFILE',
                regexpFilterExpression: '.*.yaml'
            ]
        ])
    ])
}
