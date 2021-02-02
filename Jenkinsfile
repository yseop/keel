#!/usr/bin/env groovy

@Library('Utilities') _

pipeline {
    agent {
        node {
            label 'ops'
        }
    }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
        timeout time: 30, unit: 'MINUTES'
    }

    environment {
        DOCKER                  = credentials('docker-hub')
        DOCKER_REPO_OPS         = 'yseopops'
        DOCKER_IMAGE            = 'keel'
        DOCKER_BUILD_TAG        = utils.getDockerBuildTag()
        DOCKER_TAG              = utils.getDockerTag()
    }

    stages {
        stage('Abort previous build') {
            steps {
                script {
                    utils.killPreviousBuilds()
                }
            }
        }

        stage ('Configure') {
            steps {
                sh "echo ${env.DOCKER_PSW} | docker login --username ${env.DOCKER_USR} --password-stdin"
            }
        }

        stage('Tests') {
            steps {
                sh 'make test'
            }
        }

        stage('Build') {
            steps {
                sh 'make build'
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_BUILD_TAG} ."
            }
        }

        stage('Scan Docker image') {
            steps {
                script {
                    utils.scanDockerImage("${DOCKER_IMAGE}:${DOCKER_BUILD_TAG}")
                }
            }
        }

        stage('Publish Docker image') {
            steps {
                script {
                    utils.pushDockerImageAs("${DOCKER_IMAGE}:${DOCKER_BUILD_TAG}", "${DOCKER_REPO_OPS}/${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }
    }

    post {
        success {
            slackSend(
                color: 'good',
                message: "SUCCESSFUL: *anna_master* <${env.BUILD_URL}|[${env.BUILD_NUMBER}]> <@${SLACK_NOTIFY}>"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "FAILED: *anna_master* <${env.BUILD_URL}console|[${env.BUILD_NUMBER}]> <@${SLACK_NOTIFY}>"
            )
        }
        cleanup {
            // cleaning docker local images
            sh "docker rmi -f ${DOCKER_IMAGE}:${DOCKER_BUILD_TAG}"

            // clean up our workspace
            cleanWs deleteDirs: true
            deleteDir()
        }
    }
}
