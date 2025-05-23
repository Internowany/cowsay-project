pipeline {
    options {
        timestamps()
    }
    agent any
    environment {
        AWS_REGION = 'eu-central-1'
        ECR_REPO = '017820661172.dkr.ecr.eu-central-1.amazonaws.com/seba/cowsay_project'
        IMAGE_TAG = 'latest'
        VERSION = ''
        COMMIT = ''
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    COMMIT = sh(returnStdout: true, script:'git log --pretty=format:"%s" | head -1')
                    VERSION = sh(returnStdout: true, script:'git tag --sort=-creatordate | head -1')
                    def versionParts = VERSION.tokenize('.')
                    def major = versionParts[0] as int
                    def minor = versionParts[1] as int
                    def patch = versionParts[2] as int

                    if ("${COMMIT}".contains('[major]')) {
                        sh "echo 'New MAJOR release'"
                        major += 1
                        minor = 0
                        patch = 0
                        Release = 'True'
                    }
                    else if ("${COMMIT}".contains('[minor]')) {
                        sh "echo 'New MINOR release'"
                        minor += 1
                        patch = 0
                        Release = 'True'
                    }
                    else if ("${COMMIT}".contains('[patch]')) {
                        sh "echo 'New PATCH release'"
                        patch += 1
                        Release = 'True'
                    }
                    else {
                        sh "echo 'Not for release'"
                        Release = 'False'
                    }
                    def newVersion = "${major}.${minor}.${patch}"
                    sh "echo 'New version: ${newVersion}'"
                    VERSION = "${newVersion}"
                    sh "echo '${VERSION}'"
                }
            }
        }
        stage('Build') {
            steps {
                echo 'Building...'
                sh "./build"
                sh "echo '${VERSION}'"
            }
        }
        stage('Test') {
            steps {
                echo 'Testing...'
                sh "./test"
            }
        }
        stage('Deploy to ECR') {
            when {
                expression { "${Release}" == 'True' }
            }
            steps {
                echo 'Deploying to ECR...'
                sh "echo '${VERSION}'"
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                        docker tag cowsay_project-app:latest ${ECR_REPO}:${VERSION}
                        docker tag cowsay_project-app:latest ${ECR_REPO}:latest
                        docker push ${ECR_REPO}:${VERSION}
                        docker push ${ECR_REPO}:latest
                        docker rmi ${ECR_REPO}:${VERSION}
                        docker rmi ${ECR_REPO}:latest
                    """
                }
            }
        }
        stage('Push tag to git') {
            when {
                expression { "${Release}" == 'True' }
            }
            steps {
                sh "git tag ${VERSION}"
                withCredentials([string(credentialsId: 'Internowany', variable: 'TOKEN')]) {
                    sh "git push https://Internowany:$TOKEN@github.com/Internowany/cowsay-project.git tag ${VERSION}"
                }
            }
        }
        stage('Update ArgoCD') {
            when {
                expression { "${Release}" == 'True' }
            }
            steps {
                echo 'Updatingsss version in manifest...'
                sh 'cd ..'
                git branch: 'master', credentialsId: 'internowany-at-github', url: 'git@github.com:Internowany/demo-crm.git', changelog: false, poll: false
                sh """
                    sed -i 's/tag:.*/tag: "${VERSION}"/g' app-democrm/values.yaml
                    echo 'Git Config'
                    git checkout master
                    git config --global user.email "Jenkins@internowany.click"
                    git config --global user.name "Jenkins-ci"
                    git add .
                    git commit -m "Update Image tag to ${VERSION}"
                """
                withCredentials([string(credentialsId: 'Internowany', variable: 'TOKEN')]) {
                    sh "git push https://Internowany:$TOKEN@github.com/Internowany/demo-crm.git"
                }
            }
        }
        stage('Cleanup') {
            steps {
                echo 'Cleaning garbage...'
                sh '''
                    docker rm -f app
                    docker rmi cowsay_project-app:latest
                '''
            }
        }
    }
    post {
        success {
            script {
                emailext(
                    to: '$DEFAULT_RECIPIENTS',
                    subject: "Jenkins Builds Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: "Good news! The build was successful.\n\nJob: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nCheck it here: ${env.BUILD_URL}"
                )
            }
        }
        failure {
            script {
                emailext(
                    to: '$DEFAULT_RECIPIENTS',
                    subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: "Unfortunately, the build failed.\n\nJob: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nCheck the console output to view the details: ${env.BUILD_URL}"
                )
            }
        }
    }
}
