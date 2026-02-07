pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = 'johndavidmcfrank'
    MOVIE_IMAGE    = 'movie-service'
    CAST_IMAGE     = 'cast-service'
    // kubeconfig déjà copié chez jenkins, mais on force pour être sûr
    KUBECONFIG     = '/var/lib/jenkins/.kube/config'
    HELM_CHART_DIR = './charts'
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Compute env / tag') {
      steps {
        script {
          def b = env.BRANCH_NAME ?: 'unknown'
          // mapping branches -> namespaces
          if (b == 'dev')      { env.TARGET_NS = 'dev' }
          else if (b == 'qa')  { env.TARGET_NS = 'qa' }
          else if (b == 'staging') { env.TARGET_NS = 'staging' }
          else if (b == 'master')  { env.TARGET_NS = 'prod' }
          else { env.TARGET_NS = '' }

          // tag Docker par branche + build number (traçabilité)
          env.IMAGE_TAG = "${b}-${env.BUILD_NUMBER}"
          echo "BRANCH=${b} TARGET_NS=${env.TARGET_NS} IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Tools check') {
      steps {
        sh '''
          set -e
          whoami
          git --version
          docker version
          kubectl version --client
          helm version
          kubectl get ns
        '''
      }
    }

    stage('Docker login') {
      steps {
        withCredentials([string(credentialsId: 'dockerhub-token', variable: 'DOCKER_TOKEN')]) {
          sh '''
            set -e
            echo "$DOCKER_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
          '''
        }
      }
    }

    stage('Build & Push images') {
      steps {
        sh '''
          set -e
          docker buildx create --name jenkins-builder --use >/dev/null 2>&1 || docker buildx use jenkins-builder
          docker buildx inspect --bootstrap

          docker buildx build --platform linux/amd64 -t "$DOCKERHUB_USER/$MOVIE_IMAGE:$IMAGE_TAG" --push ./movie-service
          docker buildx build --platform linux/amd64 -t "$DOCKERHUB_USER/$CAST_IMAGE:$IMAGE_TAG"  --push ./cast-service
        '''
      }
    }

    stage('Deploy dev/qa/staging') {
      when {
        anyOf { branch 'dev'; branch 'qa'; branch 'staging' }
      }
      steps {
        sh '''
          set -e
          NS="$TARGET_NS"

          echo "=== Apply deps in $NS ==="
          kubectl -n "$NS" apply -f k8s/movie-db.yaml
          kubectl -n "$NS" apply -f k8s/cast-db.yaml
          kubectl -n "$NS" apply -f k8s/cast-service.yaml

          kubectl -n "$NS" rollout status deploy/movie-db --timeout=180s
          kubectl -n "$NS" rollout status deploy/cast-db  --timeout=180s
          kubectl -n "$NS" rollout status deploy/cast-service --timeout=180s || true

          echo "=== Helm upgrade movie in $NS ==="
          helm upgrade --install movie "$HELM_CHART_DIR" -n "$NS" \
            -f "values-${NS}-movie.yaml" \
            --set image.repository="$DOCKERHUB_USER/$MOVIE_IMAGE" \
            --set image.tag="$IMAGE_TAG"

          kubectl -n "$NS" rollout status deploy/movie-fastapiapp --timeout=180s

          echo "=== Smoke test (port-forward) ==="
          kubectl -n "$NS" port-forward svc/movie-fastapiapp 18080:80 >/tmp/pf.log 2>&1 &
          PF_PID=$!
          sleep 2
          curl -fsS "http://127.0.0.1:18080/api/v1/movies/openapi.json" >/dev/null
          curl -fsS "http://127.0.0.1:18080/api/v1/movies/" >/dev/null || true
          kill $PF_PID || true
        '''
      }
    }

    stage('Deploy prod (manual, master only)') {
      when { branch 'master' }
      steps {
        script {
          // Ici : input visible dans l’UI du build, au moment où Jenkins arrive ici.
          input message: "Déployer en PRODUCTION (namespace prod) ?"
        }

        sh '''
          set -e
          NS="prod"

          echo "=== Apply deps in $NS ==="
          kubectl -n "$NS" apply -f k8s/movie-db.yaml
          kubectl -n "$NS" apply -f k8s/cast-db.yaml
          kubectl -n "$NS" apply -f k8s/cast-service.yaml

          kubectl -n "$NS" rollout status deploy/movie-db --timeout=180s
          kubectl -n "$NS" rollout status deploy/cast-db  --timeout=180s
          kubectl -n "$NS" rollout status deploy/cast-service --timeout=180s || true

          echo "=== Helm upgrade movie in $NS ==="
          helm upgrade --install movie "$HELM_CHART_DIR" -n "$NS" \
            -f "values-prod-movie.yaml" \
            --set image.repository="$DOCKERHUB_USER/$MOVIE_IMAGE" \
            --set image.tag="$IMAGE_TAG"

          kubectl -n "$NS" rollout status deploy/movie-fastapiapp --timeout=180s
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
