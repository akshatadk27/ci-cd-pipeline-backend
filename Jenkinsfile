pipeline {
    agent any

    environment {
        DEPLOY_PATH = '/home/ubuntu/ci-cd-pipeline/ci-cd-pipeline-backend'
        BACKEND_PORT = '5000'  // Change to your app's port
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
                    echo "Files deployed to ${DEPLOY_PATH}"
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
                    
                    echo "Dependencies installed successfully!"
                """
            }
        }

        stage('Stop Existing Backend') {
            steps {
                sh """
                    echo "Stopping existing backend processes..."
                    
                    # Find and kill process running on port ${BACKEND_PORT}
                    PID=\$(lsof -ti:${BACKEND_PORT} || echo "")
                    if [ ! -z "\$PID" ]; then
                        echo "Killing process \$PID on port ${BACKEND_PORT}"
                        kill -9 \$PID || echo "Process already stopped"
                    else
                        echo "No process found on port ${BACKEND_PORT}"
                    fi
                    
                    # Also kill any python app.py processes
                    pkill -f 'python.*app.py' || echo "No app.py process found"
                """
            }
        }

        stage('Start Backend') {
            steps {
                sh """
                    cd ${DEPLOY_PATH}
                    
                    # Start the backend in background
                    echo "Starting backend application..."
                    source venv/bin/activate
                    nohup python3 app.py > app.log 2>&1 &
                    
                    # Wait a moment for the app to start
                    sleep 3
                    
                    # Check if app is running
                    if ps aux | grep -v grep | grep "python.*app.py" > /dev/null; then
                        echo "✅ Backend started successfully!"
                        echo "Logs: ${DEPLOY_PATH}/app.log"
                        echo "Check logs: tail -f ${DEPLOY_PATH}/app.log"
                    else
                        echo "❌ Failed to start backend. Check logs at ${DEPLOY_PATH}/app.log"
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
                        echo "📋 Backend Version Info:"
                        cat backend_version.txt
                    else
                        echo "No backend_version.txt found"
                        # Create a version file
                        echo "v1.0.0-$(date +'%Y%m%d-%H%M%S')" > backend_version.txt
                        cat backend_version.txt
                    fi
                """
            }
        }
    }

    post {
        success {
            echo '🎉 Backend deployment completed successfully!'
            echo "Application running on: http://localhost:${BACKEND_PORT}"
            
            // Display the version
            sh """
                cd ${DEPLOY_PATH}
                echo "Current version: \$(cat backend_version.txt 2>/dev/null || echo 'Unknown')"
            """
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
