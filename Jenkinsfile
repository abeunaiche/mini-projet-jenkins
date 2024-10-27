/* import shared library */
@Library('slack-share-library')_

pipeline {
    environment {
        IMAGE_NAME = "staticwebsite"
        IMAGE_TAG = "latest"
        DOCKERHUB_ID = "abeunaiche"
        DOCKERHUB_PASSWORD = credentials('dockerhub_password')
        APP_CONTAINER_PORT = "5000"
        APP_EXPOSED_PORT = "80"
        STAGING = "static-web-site-staging"
        PRODUCTION = "static-web-site-prod"
    }
    agent none
    stages {
        stage('Build image') {
            agent any
            steps {
                script {
                    sh 'docker build -t ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG .'
                }
            }
        }
        
        stage('Run container using built image') {
            agent any
            steps {
                script {
                    sh '''
                    echo "Cleaning existing container if exist"
                    docker ps -a | grep -i $IMAGE_NAME && docker stop $IMAGE_NAME && docker rm $IMAGE_NAME
                    docker run --name $IMAGE_NAME -d -p $APP_EXPOSED_PORT:$APP_CONTAINER_PORT -e PORT=$APP_CONTAINER_PORT ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
                    sleep 5
                    '''
                }
            }
        }
        
        stage('Test image') {
            agent any
            steps {
                script {
                    sh '''
                    curl 172.17.0.1 | grep -i "Dimension"
                    '''
                }
            }
        }
        
        stage('Clean container') {
            agent any
            steps {
                script {
                    sh '''
                    docker stop $IMAGE_NAME
                    docker rm $IMAGE_NAME
                    '''
                }
            }
        }

        stage ('Push image on DockerHub') {
            agent any
            steps {
                script {
                    sh '''
                    echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_ID --password-stdin
                    docker push ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy app in staging on Heroku') {
            when {
                expression { GIT_BRANCH == 'origin/main' }
            }
	        agent {
        	    docker { image 'franela/dind' }
	        }
            environment {
                HEROKU_API_KEY = credentials('heroku_api_key')
            }
            steps {
                script {
                    sh '''
                    apk --no-cache add npm
                    npm install -g heroku
                    heroku container:login
                    heroku create $STAGING || echo "projets already exist"
                    heroku container:push -a $STAGING web
                    heroku container:release -a $STAGING web
                    '''
                }
            }
        }
        
        stage('Deploy app in production on Heroku') {
            when {
                expression { GIT_BRANCH == 'origin/main' }
            }
	        agent {
        	    docker { image 'franela/dind' }
	        }
            environment {
                HEROKU_API_KEY = credentials('heroku_api_key')
            }
            steps {
                script {
                    sh '''
                    apk --no-cache add npm
                    npm install -g heroku
                    heroku container:login
                    heroku create $PRODUCTION || echo "projets already exist"
                    heroku container:push -a $PRODUCTION web
                    heroku container:release -a $PRODUCTION web
                    '''
                }
            }
        }
    }
    post {
        always {
            script {
                slackNotifier currentBuild.result
            }
        } 
    }
}