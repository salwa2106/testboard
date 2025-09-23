pipeline {
  agent any
  options { timestamps() }

  environment {
    // CHANGE this to the URL Jenkins can reach your FastAPI at:
    // If Jenkins runs on the same Windows machine as your API: http://localhost:8000
    // If Jenkins runs in Docker on the same machine (Windows/Mac): http://host.docker.internal:8000
    BACKEND_BASE = 'http://localhost:8000'

    PROJECT_ID   = '1'  // your project id in the API
    API_USER     = credentials('testboard_user_email')      // Jenkins Secret Text: e.g. qa@example.com
    API_PASS     = credentials('testboard_user_password')   // Jenkins Secret Text: e.g. Secret123!
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Create venv & install deps') {
      steps {
        powershell '''
          py -m venv .venv
          .\\.venv\\Scripts\\Activate.ps1
          python -m pip install --upgrade pip
          pip install -r requirements.txt
        '''
      }
    }

    stage('Run pytest (produce JUnit)') {
      steps {
        powershell '''
          .\\.venv\\Scripts\\Activate.ps1
          pytest --junitxml=report.xml
          if ($LASTEXITCODE -ne 0) {
            Write-Host "pytest had failures; continuing the pipeline"; $global:BUILD_FAILED=$true
          }
        '''
      }
      post {
        always { junit 'report.xml' }   // publish JUnit in Jenkins UI
      }
    }

    stage('Get API token') {
      steps {
        powershell '''
          $body = @{ email = "$env:API_USER"; password = "$env:API_PASS" } | ConvertTo-Json
          $resp = Invoke-RestMethod -Method POST -Uri "$env:BACKEND_BASE/api/auth/login" -ContentType "application/json" -Body $body
          $resp.access_token | Out-File -Encoding ascii token.txt
        '''
      }
    }

    stage('Upload JUnit to TestBoard') {
      steps {
        powershell '''
          $TOKEN = Get-Content token.txt -Raw
          $headers = @{ Authorization = "Bearer $TOKEN" }
          $form = @{ file = Get-Item report.xml }
          Invoke-WebRequest -Method POST -Uri "$env:BACKEND_BASE/api/ingest/junit?project_id=$env:PROJECT_ID" -Headers $headers -Form $form | Out-Null
        '''
      }
    }
  }
}
