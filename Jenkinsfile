pipeline {

    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        PROJECT = "gatsby-blog-test"
        EMAIL_DEVELOPERS = "${EMAIL_DEVELOPERS}"

        SONAR_TOOL = tool "${SONAR_TOOL}"
        SONAR_SERVER = "${SONAR_SERVER}"

        AWS_DEFAULT_REGION = "${AWS_REGION}"
    }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'ref', value: '$.ref']
            ],

            causeString: 'Triggered on $ref',

            token: 'gatsby-blog-test',

            printContributedVariables: true,
            printPostContent: true,

            regexpFilterText: '$ref',
            regexpFilterExpression: 'refs/heads/' + BRANCH_NAME
        )
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        env.AWS_BUCKET_S3 = "s3://prod-${PROJECT}"
                        env.AWS_CDN_DISTRIBUTION = "${AWS_CDN_DISTRIBUTION_PDN}"
                        env.AWS_CREDENTIALS_ID = "${AWS_CREDENTIALS_PDN}"
                    } else if (env.BRANCH_NAME == "develop") {
                        env.AWS_BUCKET_S3 = "s3://dev-${PROJECT}"
                        env.AWS_CDN_DISTRIBUTION = "${AWS_CDN_DISTRIBUTION_DEV}"
                        env.AWS_CREDENTIALS_ID = "${AWS_CREDENTIALS_DEV}"
                    }
                }
            }
        }
        stage('checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: scm.branches,
                    doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
                    extensions: scm.extensions + [[$class: 'SubmoduleOption', parentCredentials: true, reference: '', recursiveSubmodules: true]],
                    submoduleCfg: [],
                    userRemoteConfigs: scm.userRemoteConfigs
                ])
            }
        }
        stage('npm install') {
            steps {
                sh 'npm install'
            }
        }
        stage('npm build') {
            steps {
                sh 'gatsby build'
            }
        }
        stage("SonarQube analysis") {
            steps {
                echo "[EXEC] - Analisis estatico de codigo"
                script {
                    withSonarQubeEnv("${SONAR_SERVER}") {
                        sh "${SONAR_TOOL}/bin/sonar-scanner -Dsonar.projectKey=${PROJECT} -Dsonar.projectName=${PROJECT}"
                    }
                }
            }
        }
        stage('S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh "aws s3 sync ./public/ ${AWS_BUCKET_S3} --delete"
                }
            }
        }
        stage('CloudFront') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDENTIALS_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh "aws cloudfront create-invalidation --distribution-id ${AWS_CDN_DISTRIBUTION} --paths '/*'"
                }
            }
        }
    }

    post {
        // always {
        //     deleteDir()
        // }

        // failure {
        //     mail to: "${EMAIL_DEVELOPERS}",
        //          subject: "[Familia][DevOps] Failed ${PROJECT}",
        //          body: "Error en el proyecto ${env.BUILD_URL}"
        // }
    }
}