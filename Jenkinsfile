pipeline {
    triggers {
        pollSCM('H/1 * * * *') // Check every 5 minutes
    }
    agent {label 'vmtest'}
    environment {
        GITLAB_IMAGE_NAME = "registry.gitlab.com/threeman/boomtestjenkins"
        VMTEST_MAIN_WORKSPACE = "/home/vmtest/workspace/jenkinstestJob"
    }
    stages {
        stage('Deploy Docker Compose') {
            agent {label 'vmtest-test'}
            steps {
                sh "docker compose up -d --build"
            }
        }
        stage("Run Tests") {
            agent {label 'vmtest-test'}
            steps {
                sh '''
                . /home/vmtest/env/bin/activate
                
                cd ${VMTEST_MAIN_WORKSPACE}
                python3 -m unittest unit_test.py -v
                coverage run -m unittest unit_test.py -v
                coverage report -m

                # Check if the directory already exists
                rm -rf robot-aun
                if [ ! -d "robot-aun" ]; then
                    git clone https://github.com/SDPxMTNRWTPKKS/robot-aun.git
                fi
                
                # Install dependencies before running tests
                pip install -r requirements.txt 
                cd robot-aun
                robot test-calculate.robot || true
                '''
            }
        }
        stage("Delivery to GitLab Registry") {
            agent {label 'vmtest-test'}
            steps {
                withCredentials(
                    [usernamePassword(
                        credentialsId: 'gitlab-admin',
                        passwordVariable: 'gitlabPassword',
                        usernameVariable: 'gitlabUser'
                    )]
                ){
                    sh "sudo docker login registry.gitlab.com -u ${gitlabUser} -p ${gitlabPassword}"
                    sh "sudo docker tag ${GITLAB_IMAGE_NAME} ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "sudo docker push ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "sudo docker rmi ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }
        stage("Pull from GitLab Registry") {
            agent {label 'vmpreprod'}
            steps {
                withCredentials(
                    [usernamePassword(
                        credentialsId: 'gitlab-admin',
                        passwordVariable: 'gitlabPassword',
                        usernameVariable: 'gitlabUser'
                    )]
                ) {
                    script {
                        def containers = sh(script: "docker ps -q", returnStdout: true).trim()
                        if (containers) {
                            sh "sudo docker stop ${containers} || true "
                        } else {
                            echo "No running containers to stop."
                        }
                    }
                    sh "sudo docker login registry.gitlab.com -u ${gitlabUser} -p ${gitlabPassword}"
                    sh "sudo docker pull ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "sudo docker run -p 5000:5000 -d ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }
    }
}