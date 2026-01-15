pipeline {
  agent any

  environment {
    SSH_CRED = 'mi-ssh-key'

    DEV_HOST = '172.22.50.26'
    UAT_HOST = '172.22.50.136'

    CONTAINER = 'apim-tutorial_mi-runtime_1'
    MI_CARBONAPPS_DIR = '/home/wso2carbon/wso2mi-4.2.0/repository/deployment/server/carbonapps'
    REMOTE_STAGE_DIR = '/tmp/mi-deploy'
    SMOKE_URL = 'http://localhost:8290/HelloWorld/'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Pick CAR') {
      steps {
        script {
          env.CAR_FILE = sh(script: 'ls -1 *.car **/*.car 2>/dev/null | head -n 1', returnStdout: true).trim()
          if (!env.CAR_FILE) error("No .car found in repo. Commit it (e.g. HelloWorldService_1.0.0.car) or build it in pipeline.")
          sh "echo Using CAR_FILE=${env.CAR_FILE}"
        }
      }
    }

    stage('Deploy DEV') {
      when { branch 'dev' }
      steps {
        sshagent(credentials: ["${env.SSH_CRED}"]) {
          sh """
            set -euo pipefail
            echo "Deploying to DEV ${DEV_HOST}"

            ssh -o StrictHostKeyChecking=no ubuntu@${DEV_HOST} "mkdir -p ${REMOTE_STAGE_DIR}"
            scp -o StrictHostKeyChecking=no "${CAR_FILE}" ubuntu@${DEV_HOST}:${REMOTE_STAGE_DIR}/

            ssh -o StrictHostKeyChecking=no ubuntu@${DEV_HOST} 'bash -s' <<'EOS'
              set -euo pipefail

              CAR_NAME="$(ls -1 ${REMOTE_STAGE_DIR}/*.car | xargs -n1 basename | head -n1)"
              echo "Deploying \$CAR_NAME into ${CONTAINER}"

              # remove old artifact with same name to avoid duplicate deploy behavior
              docker exec ${CONTAINER} bash -lc "rm -f ${MI_CARBONAPPS_DIR}/\$CAR_NAME"

              docker cp "${REMOTE_STAGE_DIR}/\$CAR_NAME" "${CONTAINER}:${MI_CARBONAPPS_DIR}/\$CAR_NAME"
              docker restart ${CONTAINER}

              # wait for MI to come back
              for i in {1..45}; do
                if curl -sf ${SMOKE_URL} >/dev/null 2>&1; then
                  break
                fi
                sleep 2
              done

              # smoke test (fails pipeline if not 200)
              curl -sf ${SMOKE_URL} | cat
              echo ""
              echo "Smoke test OK: ${SMOKE_URL}"

              # show deployment evidence
              docker logs --tail 120 ${CONTAINER} | egrep -i "HelloWorld|HelloWorldService|APIDeployer|CappDeployer|ERROR|Exception" || true
            EOS
          """
        }
      }
    }

    stage('Deploy UAT') {
      when { branch 'uat' }
      steps {
        sshagent(credentials: ["${env.SSH_CRED}"]) {
          sh """
            set -euo pipefail
            echo "Deploying to UAT ${UAT_HOST}"

            ssh -o StrictHostKeyChecking=no ubuntu@${UAT_HOST} "mkdir -p ${REMOTE_STAGE_DIR}"
            scp -o StrictHostKeyChecking=no "${CAR_FILE}" ubuntu@${UAT_HOST}:${REMOTE_STAGE_DIR}/

            ssh -o StrictHostKeyChecking=no ubuntu@${UAT_HOST} 'bash -s' <<'EOS'
              set -euo pipefail

              CAR_NAME="$(ls -1 ${REMOTE_STAGE_DIR}/*.car | xargs -n1 basename | head -n1)"
              echo "Deploying \$CAR_NAME into ${CONTAINER}"

              docker exec ${CONTAINER} bash -lc "rm -f ${MI_CARBONAPPS_DIR}/\$CAR_NAME"
              docker cp "${REMOTE_STAGE_DIR}/\$CAR_NAME" "${CONTAINER}:${MI_CARBONAPPS_DIR}/\$CAR_NAME"
              docker restart ${CONTAINER}

              for i in {1..45}; do
                if curl -sf ${SMOKE_URL} >/dev/null 2>&1; then
                  break
                fi
                sleep 2
              done

              curl -sf ${SMOKE_URL} | cat
              echo ""
              echo "Smoke test OK: ${SMOKE_URL}"

              docker logs --tail 120 ${CONTAINER} | egrep -i "HelloWorld|HelloWorldService|APIDeployer|CappDeployer|ERROR|Exception" || true
            EOS
          """
        }
      }
    }
  }
}

