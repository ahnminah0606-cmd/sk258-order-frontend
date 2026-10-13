pipeline {
  agent any

  environment {
    // === 사용자 수정 영역 ===
    GIT_URL                = 'https://github.com/ahnminah0606-cmd/sk258-order-frontend.git'
    GIT_BRANCH             = 'main'            // 또는 main
    GIT_ID                 = 'sk258-github-id'   // GitHub PAT credential ID
    IMAGE_NAME             = 'sk258-order-frontend'
    // =======================
    IMAGE_TAG              = '1.0'
    IMAGE_REGISTRY_URL     = 'skala-registry.skala-ai.com'
    IMAGE_REGISTRY_PROJECT = 'class-8'

    DOCKER_CREDENTIAL_ID   = 'skala-image-registry-id'  // Harbor 인증 정보 ID
    K8S_NAMESPACE          = 'class-8'
    GITOPS_BRANCH          = 'gitops'
    GITOPS_MANIFEST_DIR    = 'capstone/frontend'
  }

  triggers { pollSCM('H/2 * * * *') }

  options {
    disableConcurrentBuilds()
    timestamps()
  }

  stages {
    stage('Clone Repository') {
      steps {
        echo 'Clone Repository'
        git branch: "${GIT_BRANCH}", url: "${GIT_URL}", credentialsId: "${GIT_ID}"
        sh 'ls -al'
      }
    }

    // 빌드+푸시 (docker buildx 대신 BuildKit(buildctl)을 직접 사용, docker 데몬 불필요)
    stage('Image Build & Push (BuildKit)') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKER_CREDENTIAL_ID}",
          usernameVariable: 'REG_USER',
          passwordVariable: 'REG_PASS'
        )]) {
        script {
          // 이미지 태그 = IMAGE_TAG-빌드번호-커밋해시(7자리), GitOps 스테이지에서도 동일 값 사용
          def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
          env.FINAL_IMAGE_TAG = "${env.IMAGE_TAG}-frontend-bk${env.BUILD_NUMBER}-${shortCommit}"
        }
        sh '''
            set +x
            set -eu
            IMAGE_REF="${IMAGE_REGISTRY_URL}/${IMAGE_REGISTRY_PROJECT}/${IMAGE_NAME}:${FINAL_IMAGE_TAG}"

            # 1. 인증: username/password로 config.json 생성 (buildkit은 DOCKER_CONFIG의 config.json으로 레지스트리 인증)
            DOCKER_CONFIG=$(mktemp -d)
            export DOCKER_CONFIG
            trap 'rm -rf "$DOCKER_CONFIG"' EXIT
            AUTH=$(printf '%s:%s' "${REG_USER}" "${REG_PASS}" | base64 | tr -d '\n')
            cat > "$DOCKER_CONFIG/config.json" << EOF
{"auths":{"${IMAGE_REGISTRY_URL}":{"auth":"${AUTH}"}}}
EOF

            # 2. 빌드 + 푸시 (buildctl-daemonless.sh: rootless buildkitd를 즉석에서 띄워 한 번에 수행)
            # overlayfs 스냅샷터는 현재 파일시스템에서 trusted.overlay xattr을 지원하지 않아
            # "failed to get xattr" 에러 발생 → xattr에 의존하지 않는 native 스냅샷터 사용
            export BUILDKITD_FLAGS="--oci-worker-snapshotter=native --root /tmp/sk258-frontend-buildkit-$BUILD_NUMBER"
            buildctl-daemonless.sh build \
              --frontend dockerfile.v0 \
              --local context=. \
              --local dockerfile=. \
              --opt platform=linux/amd64 \
              --output type=image,name="${IMAGE_REF}",push=true
            # TLS 오류 발생 시 --output 뒤에 ,registry.insecure=true 추가
        '''
        }
      }
    }

    // GitOps 업데이트 (ArgoCD CD 트리거) ────────────────
    // kustomize/overlays/dev 이미지 태그 수정 후 gitops 브랜치에 push
    stage('GitOps Update') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${GIT_ID}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
          sh '''
            set +x
            set -eu
            SOURCE_DIR="$PWD"
            DEPLOY_DIR=$(mktemp -d)
            git config user.name 'skala-gitops'
            git config user.email 'skala@skala-ai.com'
            ASKPASS=$(mktemp)
            trap 'rm -f "$ASKPASS"; rm -rf "$DEPLOY_DIR"' EXIT
            cat > "$ASKPASS" <<'ASK'
#!/bin/sh
case "$1" in *Username*) printf '%s\n' "$GIT_USER" ;; *) printf '%s\n' "$GIT_PASS" ;; esac
ASK
            chmod 700 "$ASKPASS"
            export GIT_ASKPASS="$ASKPASS" GIT_TERMINAL_PROMPT=0
            git -c credential.helper= clone https://github.com/ahnminah0606-cmd/myfirst-api-server.git "$DEPLOY_DIR"
            cd "$DEPLOY_DIR"
            git config user.name skala-gitops
            git config user.email skala@skala-ai.com
            git -c credential.helper= fetch origin
            if git show-ref --verify --quiet "refs/remotes/origin/${GITOPS_BRANCH}"; then
              git checkout -B "$GITOPS_BRANCH" "origin/$GITOPS_BRANCH"
            else
              git checkout -B "$GITOPS_BRANCH"
            fi
            IMAGE_BASE="${IMAGE_REGISTRY_URL}/${IMAGE_REGISTRY_PROJECT}/${IMAGE_NAME}"
            mkdir -p "$GITOPS_MANIFEST_DIR"
            cp "$SOURCE_DIR/manifest.yaml" "$SOURCE_DIR/kustomization.yaml" "$GITOPS_MANIFEST_DIR/"
            (cd "$GITOPS_MANIFEST_DIR" && kustomize edit set image "$IMAGE_BASE=$IMAGE_BASE:$FINAL_IMAGE_TAG")
            git add "$GITOPS_MANIFEST_DIR"
            git commit -m "[AUTO] image: ${IMAGE_NAME}:${FINAL_IMAGE_TAG}"
            git -c credential.helper= push origin "$GITOPS_BRANCH"
            echo "GitOps updated: $FINAL_IMAGE_TAG"
          '''
        }
      }
    }

  }

  post {
    always {
      deleteDir()  // 워크스페이스 정리 (git clone, target/, 빌드 아티팩트)
    }
  }
}
