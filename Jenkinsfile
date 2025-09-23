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
        powershell '''
          python --version
          python -m venv .venv
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
        always { junit 'report.xml' }
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