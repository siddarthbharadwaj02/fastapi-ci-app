pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: docker
spec:
  containers:
    - name: docker
      image: docker:20.10.24-dind
      securityContext:
        privileged: true
      tty: true
      command:
        - sh
        - -c
        - |
          dockerd-entrypoint.sh &
          sleep 10 && tail -f /dev/null
      volumeMounts:
        - name: docker-graph-storage
          mountPath: /var/lib/docker
    - name: jnlp
      image: jenkins/inbound-agent:latest
      args:
        - \$(JENKINS_SECRET)
        - \$(JENKINS_NAME)
  volumes:
    - name: docker-graph-storage
      emptyDir: {}
"""
    }
  }

  environment {
    REGISTRY = "docker.io"  // Give the docker registry
    IMAGE_NAME = "fastapi-ci-app"
  }

  stages {
    stage('Clone') {
      steps {
        git branch: 'main', url: 'https://github.com/your_repository/fastapi-ci-app.git'
      }
    }

    stage('Buildx: Multi-Arch Docker Image') {
      steps {
        container('docker') {
          script {
            def versionTag = "v${new Date().format('yyyyMMddHHmmss')}"
            def imageWithVersion = "${REGISTRY}/${IMAGE_NAME}:${versionTag}"
            def imageLatest = "${REGISTRY}/${IMAGE_NAME}:latest"

            withCredentials([usernamePassword(
              credentialsId: 'docker-reg', // update your correct ID for docker registory created in the Jenkins credentials manager
              usernameVariable: 'DOCKER_USER',
              passwordVariable: 'DOCKER_PASS'
            )]) {
              sh """
                echo "Waiting for Docker to be ready..."
                for i in \$(seq 1 30); do
                  docker info > /dev/null 2>&1 && break
                  echo "Waiting..."
                  sleep 2
                done

                echo "Installing buildx..."
                mkdir -p ~/.docker/cli-plugins
                wget -q https://github.com/docker/buildx/releases/download/v0.13.1/buildx-v0.13.1.linux-amd64 -O ~/.docker/cli-plugins/docker-buildx
                chmod +x ~/.docker/cli-plugins/docker-buildx

                echo "Enable QEMU..."
                docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

                echo "Login to Docker Registry..."
                echo "\$DOCKER_PASS" | docker login ${REGISTRY} -u "\$DOCKER_USER" --password-stdin

                echo "Create buildx builder..."
                docker buildx create --use --name multi-builder || true
                docker buildx inspect --bootstrap

                echo "Build and push multi-arch images..."
                docker buildx build --platform linux/amd64,linux/arm64 \\
                  -t ${imageWithVersion} \\
                  -t ${imageLatest} \\
                  --push app/
              """

              echo "Image Pushed: ${imageWithVersion}"
              echo "Also tagged as: ${imageLatest}"
            }
          }
        }
      }
    }

    stage('Deploy: Run on Host via SSH') {
      steps {
        container('docker') {
          withCredentials([
            usernamePassword(
              credentialsId: 'host-ssh-cred', // update your correct ID for host credentials created in the Jenkins credentials manager
              usernameVariable: 'SSH_USER',
              passwordVariable: 'SSH_PASS'
            ),
            usernamePassword(
              credentialsId: 'docker-reg',
              usernameVariable: 'DOCKER_USER',
              passwordVariable: 'DOCKER_PASS'
            )
          ]) {
            sh '''
              echo "Installing sshpass..."
              apk add --no-cache sshpass openssh-client

              echo "Running deployment script remotely..."
              sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no -p <portnumber> $SSH_USER@<Your_host_name or ip> ' // Provide the hostname or the IP of the machine to ssh & where the docker container need to run.
                export DOCKER_USER="'$DOCKER_USER'"
                export DOCKER_PASS="'$DOCKER_PASS'"
                /home/user/deploy-fastapi.sh
              '
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo 'CI/CD pipeline completed. Host container is running latest image.'
    }
    failure {
      echo 'Bild or Deploy failed. Check logs.'
    }
  }
}
