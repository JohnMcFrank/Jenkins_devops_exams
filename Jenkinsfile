pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    DOCKERHUB_USER = "johndavidmcfrank"
    MOVIE_IMAGE    = "movie-service"
    CAST_IMAGE     = "cast-service"
    IMAGE_TAG      = "dev-${BUILD_NUMBER}"

    // Active BuildKit côté CLI (utile même si on utilise buildx)
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh '''
          set -e
          echo "### Workspace:"
          pwd
          ls -lah
          echo "### Dockerfiles:"
          find . -maxdepth 3 -name Dockerfile -print
        '''
      }
    }

    stage('Tools check') {
      steps {
        sh '''
          set -e
          whoami
          git --version || true
          docker version
          docker info
          echo "### Buildx:"
          docker buildx version
          kubectl version --client || true
          helm version || true
        '''
      }
    }

    stage('Docker login') {
      environment {
        DOCKER_TOKEN = credentials('DOCKER_HUB_TOKEN')
      }
      steps {
        sh '''
          set -e
          echo "$DOCKER_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
        '''
      }
    }

    stage('Build & Push images (buildx)') {
      steps {
        sh '''
          set -e

          test -f movie-service/Dockerfile
          test -f cast-service/Dockerfile

          # S'assure d'avoir un builder usable (idempotent)
          docker buildx create --name jenkins-builder --use >/dev/null 2>&1 || docker buildx use jenkins-builder
          docker buildx inspect --bootstrap

          # Build+push en une commande (obligatoire avec buildx si on veut pousser proprement)
          docker buildx build \
            --platform linux/amd64 \
            -t "$DOCKERHUB_USER/$MOVIE_IMAGE:$IMAGE_TAG" \
            --push \
            ./movie-service

          docker buildx build \
            --platform linux/amd64 \
            -t "$DOCKERHUB_USER/$CAST_IMAGE:$IMAGE_TAG" \
            --push \
            ./cast-service
        '''
      }
    }
  }

  post {
    always {
      sh '''
        docker logout || true
        # Nettoyage léger : ne casse rien si absent
        docker buildx rm jenkins-builder >/dev/null 2>&1 || true
      '''
    }
  }
}
