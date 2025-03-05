## How to run the playbook to connect on Google Cloud VM

```
ansible-playbook --inventory inventory/vm-setup-playbook/hosts vm-setup-playbook.yml 
```



pipeline {
    agent any
    
    options {
        buildDiscarder logRotator(artifactDaysToKeepStr: "", artifactNumToKeepStr: "", daysToKeepStr: "7", numToKeepStr: "2")
    }
    
    environment {
        GITLAB_AUTH = credentials("GITLAB_AUTH")
        DOCKER_AUTH = credentials("DOCKER_AUTH")
        GITLAB_URL = "https://gitlab.com"
        GITLAB_GROUP = "epic-hot-1"
        GITLAB_REPO = "sivir-be"
        DOCKER_IMAGE = "${DOCKER_AUTH_USR}/${GITLAB_REPO}"
        DOCKER_TAG = "1.${BUILD_NUMBER}.0"
    }
    
    // parameters {
        // string(defaultValue: "test", name: "CONTAINER_NAME", trim: true)
        // string(defaultValue: "hello-world", name: "IMAGE_NAME", trim: true)
        // string(defaultValue: "8080", name: "INTERNAL_PORT", trim: true)
        // string(defaultValue: "8080", name: "EXTERNAL_PORT", trim: true)
    // }
    
    tools {
        nodejs "NodeJS"
    }

    stages {
        stage("Checkout") {
            steps {
                checkout changelog: false, poll: false,
                scm: scmGit(
                    branches: [[name: "*/jenkins-deployment"]],
                    extensions: [],
                    userRemoteConfigs: [[credentialsId: "GITLAB_AUTH", url: "${GITLAB_URL}/${GITLAB_GROUP}/${GITLAB_REPO}.git"]]
                )
            }
        }
        stage("Test") {
            steps {
                sh "npm install"
                sh "npm run test"
            }
        }
        stage("Docker build") {
            steps {
                sh "docker run --privileged --rm tonistiigi/binfmt --install amd64"
                sh "docker buildx create --name multiarch-builder --use || true"
                sh "docker buildx build --platform linux/amd64 -t ${DOCKER_IMAGE}:${DOCKER_TAG} --load ."
            }
        }
        stage("Docker push") {
            steps {
                sh "echo ${DOCKER_AUTH_PSW} | docker login -u ${DOCKER_AUTH_USR} --password-stdin"
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                sh "docker push ${DOCKER_IMAGE}:latest"
            }
        }
        // stage("Execute playbook") {
            // steps {
                // ansiColor("xterm") {
                    // ansiblePlaybook colorized: true,
                    // installation: "Ansible",
                    // inventory: "/var/jenkins_home/ansible-jenkins/ansible/inventory/vm-setup-playbook/hosts",
                    // playbook: "/var/jenkins_home/ansible-jenkins/ansible/vm-setup-playbook.yml",
                    // vaultTmpPath: "",
                    // extras: """
                        // -e CONTAINER_NAME=${CONTAINER_NAME}
                        // -e IMAGE_NAME=${IMAGE_NAME}
                        // -e INTERNAL_PORT=${INTERNAL_PORT}
                        // -e EXTERNAL_PORT=${EXTERNAL_PORT}
                    // """
                // }
            // }
        // }
    }
}
