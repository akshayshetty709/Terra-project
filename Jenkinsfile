pipeline {
    agent { label 'android-build' }   // pin to a dedicated/labeled node with enough RAM+SSD
                                        // (fall back to `agent any` only if no labeled node exists)

    options {
        timeout(time: 45, unit: 'MINUTES')   // hard ceiling for the whole pipeline
        disableConcurrentBuilds()             // avoid two builds fighting for the same Gradle cache
    }

    environment {
        S3_BUCKET        = 'android-apk11'
        APK_PATH         = 'android/app/build/outputs/apk/release/app-release.apk'
        JAVA_HOME        = '/usr/lib/jvm/java-17-openjdk-amd64'
        GRADLE_USER_HOME = "${WORKSPACE}/.gradle"   // persist Gradle cache across builds on this node
    }

    stages {

        stage('1. Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/akshayshetty709/personal-finance-mobile.git'
            }
        }

        stage('2. npm install') {
            steps {
                sh "npm ci"   // clean install from lockfile — no need to rm -rf node_modules first
            }
        }

        stage('3. prebuild') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh "npx expo install --fix"
                    sh "npx expo prebuild"
                }
            }
        }

        stage('4. build apk file') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {   // fail fast instead of silently running to 1h
                    sh '''
                        cd android &&
                        chmod +x gradlew &&
                        ./gradlew assembleRelease \
                            --build-cache \
                            --parallel \
                            -Dorg.gradle.jvmargs="-Xmx4g -XX:+UseParallelGC"
                    '''
                }
            }
        }

        stage('5. Upload APK to S3') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    withCredentials([
                        aws(credentialsId: 'AWS-Cred',
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            aws s3 cp ${APK_PATH} s3://${S3_BUCKET}/releases/app-release-${BUILD_NUMBER}.apk
                            aws s3 cp ${APK_PATH} s3://${S3_BUCKET}/latest/app-release.apk
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            // Optional: archive the APK as a Jenkins build artifact too, useful if S3 upload fails
            archiveArtifacts artifacts: "${env.APK_PATH}", allowEmptyArchive: true
        }
        failure {
            echo "Build failed — check which stage timed out above. Common causes: cold Gradle cache, agent resource starvation, or a hung dependency download."
        }
    }
}
