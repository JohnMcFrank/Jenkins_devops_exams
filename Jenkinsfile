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

    // Tag = branche + build number (ex: staging-42, qa-12, master-7)
    IMAGE_TAG      = "${BRANCH_NAME}-${BUILD_NUMBER}"

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
          echo "### Branch: $BRANCH_NAME"
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
          kubectl version --client
          helm version
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

          docker buildx create --name jenkins-builder --use >/dev/null 2>&1 || docker buildx use jenkins-builder
          docker buildx inspect --bootstrap

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

          echo "Pushed:"
          echo " - $DOCKERHUB_USER/$MOVIE_IMAGE:$IMAGE_TAG"
          echo " - $DOCKERHUB_USER/$CAST_IMAGE:$IMAGE_TAG"
        '''
      }
    }

    stage('Deploy (dev/qa/staging)') {
      when {
        anyOf {
          branch 'dev'
          branch 'qa'
          branch 'staging'
        }
      }
      steps {
        sh '''
          set -e

          NS="$BRANCH_NAME"
          echo "### Deploying to namespace: $NS"

          kubectl get ns "$NS" >/dev/null 2>&1 || kubectl create ns "$NS"

          helm -n "$NS" upgrade --install movie charts \
            --set image.repository="$DOCKERHUB_USER/$CAST_IMAGE" \
            --set image.tag="$IMAGE_TAG" \
            --set replicaCount=1

          echo "### Status"
          kubectl -n "$NS" get deploy,pod,svc -o wide
          kubectl -n "$NS" logs deploy/cast-service --tail=30 || true
        '''
      }
    }

    stage('Approve PROD (master only)') {
      when { branch 'master' }
      steps {
        input message: "Déployer en PRODUCTION (namespace prod) à partir de master ?"
      }
    }

    stage('Deploy PROD (master only)') {
      when { branch 'master' }
      steps {
        sh '''
          set -e

          NS="prod"
          echo "### Deploying to namespace: $NS (MANUAL gate passed)"

          kubectl get ns "$NS" >/dev/null 2>&1 || kubectl create ns "$NS"

          helm -n "$NS" upgrade --install movie charts \
            --set image.repository="$DOCKERHUB_USER/$CAST_IMAGE" \
            --set image.tag="$IMAGE_TAG" \
            --set replicaCount=1

          echo "### Status"
          kubectl -n "$NS" get deploy,pod,svc -o wide
          kubectl -n "$NS" logs deploy/cast-service --tail=30 || true
        '''
      }
    }
  }

  post {
    always {
      sh '''
        docker logout || true
        docker buildx rm jenkins-builder >/dev/null 2>&1 || true
      '''
    }
  }
}
