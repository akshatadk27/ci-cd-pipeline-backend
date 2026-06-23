pipeline {
    agent any

    environment {
        DEPLOY_PATH = '/var/jenkins_home/backend-app'
        BACKEND_PORT = '5000'
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out backend code from GitHub...'
                checkout scm
            }
        }

        stage('Deploy to Backend Server') {
            steps {
                sh """
                    echo "Current directory: \$(pwd)"
                    echo "Deploying to: ${DEPLOY_PATH}"
                    
                    # Create deployment directory if it doesn't exist
                    mkdir -p ${DEPLOY_PATH}
                    
                    # Copy all files to deploy directory
                    cp -r ./* ${DEPLOY_PATH}/ 2>/dev/null || true
                    
                    cd ${DEPLOY_PATH}
                    echo "Files deployed:"
                    ls -la
                """
            }
        }

        stage('Setup Virtual Environment & Install Dependencies') {
            steps {
                sh """
                    cd ${DEPLOY_PATH}
                    
                    # Create virtual environment if it doesn't exist
                    if [ ! -d "venv" ]; then
                        echo "Creating virtual environment..."
                        python3 -m venv venv
                    fi
                    
                    # Activate and install dependencies
                    source venv/bin/activate
                    pip install --upgrade pip
                    
                    if [ -f "requirements.txt" ]; then
                        echo "Installing dependencies from requirements.txt..."
                        pip install -r requirements.txt
                    else
                        echo "No requirements.txt found, installing Flask..."
                        pip install flask flask-cors
                    fi
                    
                    echo "✅ Dependencies installed successfully!"
                """
            }
        }

        stage('Stop Existing Backend') {
            steps {
                sh """
                    echo "Stopping existing backend processes..."
                    
                    # Find and kill process running on port ${BACKEND_PORT}
                    PID=\$(lsof -ti:${BACKEND_PORT} 2>/dev/null || echo "")
                    if [ ! -z "\$PID" ]; then
                        echo "Killing process \$PID on port ${BACKEND_PORT}"
                        kill -9 \$PID 2>/dev/null || echo "Process already stopped"
                    else
                        echo "No process found on port ${BACKEND_PORT}"
                    fi
                    
                    # Also kill any python app.py processes
                    pkill -f 'python.*app.py' 2>/dev/null || echo "No app.py process found"
                    
                    sleep 2
                """
            }
        }

        stage('Start Backend') {
            steps {
                sh """
                    cd ${DEPLOY_PATH}
                    
                    echo "Starting backend application..."
                    source venv/bin/activate
                    
                    # Start the backend in background
                    nohup python3 app.py > app.log 2>&1 &
                    BACKEND_PID=\$!
                    
                    echo "Backend started with PID: \$BACKEND_PID"
                    
                    # Wait a moment for the app to start
                    sleep 5
                    
                    # Check if app is running
                    if ps -p \$BACKEND_PID > /dev/null 2>&1; then
                        echo "✅ Backend started successfully!"
                        echo "PID: \$BACKEND_PID"
                        echo "Logs: ${DEPLOY_PATH}/app.log"
                    else
                        echo "❌ Failed to start backend. Check logs:"
                        cat ${DEPLOY_PATH}/app.log
                        exit 1
                    fi
                """
            }
        }

        stage('Read Backend Version') {
            steps {
                sh """
                    cd ${DEPLOY_PATH}
                    
                    if [ -f "backend_version.txt" ]; then
                        echo "📋 Backend Version:"
                        cat backend_version.txt
                    else
                        echo "No backend_version.txt found"
                        echo "v1.0.0" > backend_version.txt
                        echo "Created version file: v1.0.0"
                    fi
                """
            }
        }
    }

    post {
        success {
            echo '🎉 Backend deployment completed successfully!'
        }
        failure {
            echo '❌ Backend deployment failed!'
            sh """
                cd ${DEPLOY_PATH}
                echo "Last 20 lines of app.log:"
                tail -20 app.log 2>/dev/null || echo "No log file found"
            """
        }
    }
}
