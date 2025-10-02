pipeline {
  agent {
    docker {
      image 'docker:25.0.3-cli'
      // ล้าง entrypoint, รันเป็น root, และเมาท์ docker.sock ให้ชัดเจน
      args '--entrypoint="" -u 0:0 -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'mkdir -p security-reports'
      }
    }

    stage('Semgrep (OWASP)') {
      steps {
        sh '''
          set -e
          # ใช้โหมดไม่พึ่ง git เพื่อกัน "0 files"
          docker run --rm --volumes-from jenkins -w "$PWD" python:3.11-slim bash -lc '
            pip install --no-cache-dir -q semgrep &&
            semgrep scan --no-git \
              --include "**/*.py" \
              --config=p/owasp-top-ten \
              --config=p/python \
              --severity ERROR \
              --sarif --output security-reports/semgrep.sarif \
              --error
          '
        '''
      }
    }

    stage('Bandit (Python SAST)') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$PWD" python:3.11-slim bash -lc '
            pip install --no-cache-dir -q bandit==1.* &&
            bandit -r . -ll -f json -o security-reports/bandit.json || true
          '
        '''
      }
    }

    stage('pip-audit (Dependencies)') {
      steps {
        sh '''
          # ระบุไฟล์ให้ชัดเจน อย่าให้ -r ว่าง
          docker run --rm --volumes-from jenkins -w "$PWD" python:3.11-slim bash -lc '
            pip install --no-cache-dir -q pip-audit &&
            pip-audit -r requirements.txt -f json -o security-reports/pip-audit_requirements.json || true
          '
        '''
      }
    }

    stage('Trivy FS (Secrets & Misconfig)') {
      steps {
        sh '''
          set +e
          docker run --rm --volumes-from jenkins -w "$PWD" aquasec/trivy:latest \
            fs . \
            --scanners vuln,secret,misconfig \
            --severity HIGH,CRITICAL \
            --format sarif --output security-reports/trivy.sarif \
            --exit-code 1
          TRIVY_RC=$?
          set -e
          [ $TRIVY_RC -eq 0 ] || true
        '''
      }
    }

    stage('Publish Reports') {
      steps {
        recordIssues(
          tools: [sarif(pattern: 'security-reports/*.sarif')]
        )
        archiveArtifacts artifacts: 'security-reports/*', fingerprint: true
      }
    }
  }

  post {
    always {
      echo 'Scan completed. Reports archived in security-reports/'
    }
    success {
      echo 'Build succeeded. No blocking security findings.'
    }
    failure {
      echo 'Build failed due to security findings or scanner errors. See artifacts.'
    }
  }
}
