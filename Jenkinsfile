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
          docker version
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

    stage('Build images') {
      steps {
        sh '''
          set -e

          test -f movie-service/Dockerfile
          test -f cast-service/Dockerfile

          docker build -t $DOCKERHUB_USER/$MOVIE_IMAGE:$IMAGE_TAG ./movie-service
          docker build -t $DOCKERHUB_USER/$CAST_IMAGE:$IMAGE_TAG ./cast-service
        '''
      }
    }

    stage('Push images') {
      steps {
        sh '''
          set -e
          docker push $DOCKERHUB_USER/$MOVIE_IMAGE:$IMAGE_TAG
          docker push $DOCKERHUB_USER/$CAST_IMAGE:$IMAGE_TAG
        '''
      }
    }
  }

  post {
    always {
      sh 'docker logout || true'
    }
  }
}
