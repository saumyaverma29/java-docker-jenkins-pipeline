pipeline {
  agent any

  environment {
    DOCKERHUB_USER = "saumy123"
    IMAGE_REPO     = "basic-java-hello"

    IMAGE_TAG      = "${env.BUILD_NUMBER}"
    FULL_IMAGE     = "docker.io/${DOCKERHUB_USER}/${IMAGE_REPO}:${IMAGE_TAG}"
    LATEST_IMAGE   = "docker.io/${DOCKERHUB_USER}/${IMAGE_REPO}:latest"

    // Your Podman machine root connection (from your earlier logs)
    PODMAN_HOST     = "ssh://root@127.0.0.1:64000/run/podman/podman.sock"
    PODMAN_IDENTITY = "C:\\Users\\saumyver\\.local\\share\\containers\\podman\\machine\\machine"
  }

  options { timestamps() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build image') {
      steps {
        bat """
          set CONTAINER_HOST=%PODMAN_HOST%
          set CONTAINER_SSHKEY=%PODMAN_IDENTITY%

          podman version
          podman build --pull=always -t %FULL_IMAGE% .
          podman tag %FULL_IMAGE% %LATEST_IMAGE%
          podman image inspect %FULL_IMAGE%
        """
      }
    }

    stage('Smoke test (no pull)') {
      steps {
        powershell '''
          $ErrorActionPreference = "Stop"
          $env:CONTAINER_HOST   = $env:PODMAN_HOST
          $env:CONTAINER_SSHKEY = $env:PODMAN_IDENTITY

          # Ensure local-only
          podman image exists $env:FULL_IMAGE
          if ($LASTEXITCODE -ne 0) { throw "Image not found locally: $env:FULL_IMAGE" }

          # Run the jar container; it prints and exits
          podman run --pull=never --rm $env:FULL_IMAGE | Out-Host
        '''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DH_USER',
                                          passwordVariable: 'DH_PASS')]) {
          bat """
            set CONTAINER_HOST=%PODMAN_HOST%
            set CONTAINER_SSHKEY=%PODMAN_IDENTITY%

            echo %DH_PASS% | podman login docker.io -u %DH_USER% --password-stdin
            podman push %FULL_IMAGE%
            podman push %LATEST_IMAGE%
            podman logout docker.io
          """
        }
      }
    }
  }

  post {
    always {
      bat """
        set CONTAINER_HOST=%PODMAN_HOST%
        set CONTAINER_SSHKEY=%PODMAN_IDENTITY%
        podman image prune -f
      """
    }
  }
}