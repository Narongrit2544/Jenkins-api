pipeline {
    triggers {
        pollSCM('H/1 * * * *') // Check every 5 minutes
    }
    agent { label 'vmtest' }
    environment {
        GITLAB_IMAGE_NAME = "registry.gitlab.com/threeman/boomtestjenkins"
        VMTEST_MAIN_WORKSPACE = "/home/vmtest/workspace/jenkinstestJob"
    }
    stages {
        stage('Deploy Docker Compose') {
            agent { label 'vmtest-test' }
            steps {
                sh "docker compose up -d --build"
            }
        }
        stage("Run Tests") {
            agent { label 'vmtest-test' }
            steps {
                sh '''
                . /home/vmtest/env/bin/activate
                
                cd ${VMTEST_MAIN_WORKSPACE}
                python3 -m unittest unit_test.py -v
                coverage run -m unittest unit_test.py -v
                coverage report -m

                rm -rf robot-aun
                if [ ! -d "robot-aun" ]; then
                    git clone https://github.com/SDPxMTNRWTPKKS/robot-aun.git
                fi
                
                pip install -r requirements.txt 
                cd robot-aun
                robot test-calculate.robot || true
                '''
            }
        }
        stage("Delivery to GitLab Registry") {
            agent { label 'vmtest-test' }
            steps {
                withCredentials(
                    [usernamePassword(
                        credentialsId: 'gitlab-admin',
                        passwordVariable: 'gitlabPassword',
                        usernameVariable: 'gitlabUser'
                    )]
                ) {
                    sh "docker login registry.gitlab.com -u ${gitlabUser} -p ${gitlabPassword}"
                    sh "docker tag ${GITLAB_IMAGE_NAME} ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker push ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker rmi ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }
        stage("Pull from GitLab Registry") {
            agent { label 'vmpreprod' }
            steps {
                withCredentials(
                    [usernamePassword(
                        credentialsId: 'gitlab-admin',
                        passwordVariable: 'gitlabPassword',
                        usernameVariable: 'gitlabUser'
                    )]
                ) {
                    script {
                        def containers = sh(script: "sudo docker ps -q", returnStdout: true).trim()
                        if (containers) {
                            // Use try-catch to handle permission denied errors gracefully
                            try {
                                sh "docker stop ${containers}"
                                echo "Containers stopped successfully."
                            } catch (Exception e) {
                                echo "Failed to stop containers, but continuing: ${e.getMessage()}"
                            }
                        } else {
                            echo "No running containers to stop."
                        }
                    }
                    sh "docker login registry.gitlab.com -u ${gitlabUser} -p ${gitlabPassword}"
                    sh "docker pull ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker run -p 5000:5000 -d ${GITLAB_IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }
    }
}
