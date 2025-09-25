pipeline {
  agent any
  options { timestamps() }

  environment {
    // Python on PATH (adjust if needed)
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

    stage('Start DB (Docker)') {
      steps {
        bat '''
          docker compose -f backend\\docker-compose.db.yml up -d
          docker ps
        '''
      }
    }

    stage('Run migrations') {
      steps {
        bat '''
          call .venv\\Scripts\\activate.bat
          cd backend
          alembic upgrade head
        '''
      }
    }

    stage('Start API') {
      steps {
        // start uvicorn in background for this build
        bat '''
          cd backend
          start "" cmd /c "call ..\\..\\.venv\\Scripts\\activate.bat && python -m uvicorn app.main:app --host 0.0.0.0 --port 8000"
        '''
      }
    }

    stage('Wait for API') {
      steps {
        // poll until docs responds (replace with /health if you have one)
        powershell '''
          $ok=$false
          for($i=0;$i -lt 40;$i++){
            try{ Invoke-WebRequest -UseBasicParsing http://localhost:8000/docs | Out-Null; $ok=$true; break }catch{ Start-Sleep -Seconds 1 }
          }
          if(-not $ok){ throw "API did not become ready" }
        '''
      }
    }

    stage('Run pytest (produce JUnit)') {
      steps {
        script {
          def result = bat(script: '''
            call .venv\\Scripts\\activate.bat
            pytest --junitxml=report.xml
          ''', returnStatus: true)
          if (result != 0) {
            echo "pytest had failures; marking UNSTABLE but continuing"
            currentBuild.result = 'UNSTABLE'
          }
        }
      }
      post {
        always { junit 'report.xml' }
      }
    }

    stage('Get API token') {
      when { expression { currentBuild.result != 'FAILURE' } }
      steps {
        // write a tiny python script to avoid quoting hell
        bat '''
          echo import os,requests,sys> get_token.py
          echo u=os.getenv('API_USER'); p=os.getenv('API_PASS')>> get_token.py
          echo r=requests.post(os.getenv('BACKEND_BASE') + '/api/auth/login', json={'email':u,'password':p})>> get_token.py
          echo print('STATUS', r.status_code)>> get_token.py
          echo print('BODY', r.text)>> get_token.py
          echo r.raise_for_status()>> get_token.py
          echo tok=r.json().get('access_token')>> get_token.py
          echo assert tok, 'No access_token in response'>> get_token.py
          echo open('token.txt','w').write(tok)>> get_token.py

          call .venv\\Scripts\\activate.bat
          python get_token.py
        '''
      }
    }

    stage('Upload JUnit to TestBoard') {
      when { expression { currentBuild.result != 'FAILURE' } }
      steps {
        bat '''
          echo import requests,os>> upload.py
          echo t=open('token.txt').read().strip()>> upload.py
          echo u=os.getenv('BACKEND_BASE') + '/api/ingest/junit?project_id=' + os.getenv('PROJECT_ID')>> upload.py
          echo r=requests.post(u, headers={'Authorization': f'Bearer {t}'}, files={'file': open('report.xml','rb')})>> upload.py
          echo print('STATUS', r.status_code); print('BODY', r.text)>> upload.py
          echo r.raise_for_status()>> upload.py

          call .venv\\Scripts\\activate.bat
          python upload.py
        '''
      }
    }
  }

  post {
    always {
      // stop uvicorn if still running
      bat 'taskkill /F /IM python.exe /FI "WINDOWTITLE eq cmd.exe - python -m uvicorn*" >NUL 2>&1 || ver >NUL'
    }
  }
}
