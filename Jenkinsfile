pipeline {
  agent any
  options { timestamps() }

  environment {
    // Add Python to PATH
    PATH = "C:\\Users\\nasal\\AppData\\Local\\Programs\\Python\\Python313;C:\\Users\\nasal\\AppData\\Local\\Programs\\Python\\Python313\\Scripts;${PATH}"

    BACKEND_BASE = 'http://localhost:8000'
    PROJECT_ID   = '1'
    API_USER     = credentials('testboard_user_email')
    API_PASS     = credentials('testboard_user_password')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Create venv & install deps') {
      steps {
        bat '''
          python --version
          python -m venv .venv
          call .venv\\Scripts\\activate.bat
          python -m pip install --upgrade pip
          pip install -r requirements.txt
        '''
      }
    }

    stage('Run pytest (produce JUnit)') {
      steps {
        script {
          def result = bat(script: '''
            call .venv\\Scripts\\activate.bat
            pytest --junitxml=report.xml
            exit 0
          ''', returnStatus: true)

          if (result != 0) {
            echo "pytest had failures; continuing the pipeline"
            currentBuild.result = 'UNSTABLE'
          }
        }
      }
      post {
        always { junit 'report.xml' }
      }
    }

    stage('Get API token') {
      steps {
        bat '''
          call .venv\\Scripts\\activate.bat
          python -c "import requests, json, os; resp = requests.post('%BACKEND_BASE%/api/auth/login', json={'email': '%API_USER%', 'password': '%API_PASS%'}); open('token.txt', 'w').write(resp.json()['access_token'])"
        '''
      }
    }

    stage('Upload JUnit to TestBoard') {
      steps {
        bat '''
          call .venv\\Scripts\\activate.bat
          python -c "import requests; token = open('token.txt').read().strip(); requests.post('%BACKEND_BASE%/api/ingest/junit?project_id=%PROJECT_ID%', headers={'Authorization': f'Bearer {token}'}, files={'file': open('report.xml', 'rb')})"
        '''
      }
    }
  }
}