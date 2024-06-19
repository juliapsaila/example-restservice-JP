pipeline {
    agent none

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Building ...'
                sh 'mvn -B -DskipTests clean package'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }

        stage('Postman Tests') {
            agent {
                docker {
                    image 'postman/newman:alpine'
                    args '--entrypoint=""'
                }
            }
            steps {
                echo 'Running Postman API Tests ...'
                sh 'newman run postman-api-tests/postman-collection.json -r junit --reporter-junit-export postman-api-tests/results.xml'
            }
            post {
                always {
                    junit 'postman-api-tests/results.xml'
                }
            }
        }

        stage('Static Code Analysis') {
            agent any
            steps {
                echo 'Running Static Code Analysis ...'
            }
        }

        stage('Security Scanning') {
            agent any
            steps {
                echo 'Running Security Scanning ...'
            }
        }

        stage('Package and Publish Artifacts') {
            when {
                anyOf {
                    branch 'main';
                    buildingTag()
                }
            }
            agent {
                docker {
                    image 'docker:25.0-dind'
                }
            }
            steps {
                echo 'Package ...'
                sh 'docker build ./rest-service -t rest-service:$GIT_COMMIT'
                // Uncomment the line below to push the image to a registry
                // sh 'docker push rest-service:$GIT_COMMIT'
            }
        }

        stage('Dev Environment') {
            when {
                anyOf {
                    branch 'develop';
                    buildingTag()
                }
            }
            stages {
                stage('Dev Deploy') {
                    agent {
                        docker {
                            image 'docker:25.0-dind'
                        }
                    }
                    steps {
                        echo 'Deploying to Dev ...'
                        echo 'Stopping older version of rest-service if it is running'
                        sh 'docker rm $(docker stop $(docker ps -a -q --filter name=rest-service --format="{{.ID}}")) || true'
                        echo "Deploying and starting rest-service:$GIT_COMMIT"
                        sh 'docker container run -d -p 8081:8081 --name rest-service --network=test-automation-demo rest-service:$GIT_COMMIT'
                    }
                }
                stage('Dev Integration Tests') {
                    failFast true
                    parallel {
                        stage('Dev Acceptance Tests') {
                            agent {
                                docker {
                                    image 'postman/newman:alpine'
                                    args '--entrypoint="" --network=test-automation-demo'
                                }
                            }
                            steps {
                                echo 'Running Acceptance Tests on Dev ....'
                                sh 'newman run postman-api-tests/postman-collection.json -r junit --reporter-junit-export postman-api-tests/results.xml'
                            }
                            post {
                                always {
                                    junit 'postman-api-tests/results.xml'
                                }
                            }
                        }
                        stage('Dev Performance Tests') {
                            agent any
                            steps {
                                echo 'Running Performance Tests on Dev ...'
                            }
                        }
                    }
                }
            }
        }

        stage('Prod Environment') {
            when {
                anyOf {
                    branch 'main';
                    buildingTag()
                }
            }
            stages {
                stage('Prod Deploy') {
                    agent any
                    steps {
                        echo 'Deploying to Prod ...'
                    }
                }
                stage('Prod Sanity Check') {
                    agent any
                    steps {
                        echo 'Running Smoke Tests on Prod ...'
                    }
                }
            }
        }
    }
}
