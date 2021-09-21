#!/usr/bin/env groovy
@Library(['piper-lib', 'piper-lib-os']) _

def qualityBadge = addEmbeddableBadgeConfiguration(id: "quality", subject: "Quality")
def testsBadge = addEmbeddableBadgeConfiguration(id: "tests", subject: "Tests")
def sonarQubeBadge = addEmbeddableBadgeConfiguration(id: "sonarqube", subject: "SonarQube")
def xMakeBuildBadge = addEmbeddableBadgeConfiguration(id: "xmake", subject: "xMake Build")
def checkmarxBadge = addEmbeddableBadgeConfiguration(id: "checkmarx", subject: "Checkmarx")
def whiteSourceBadge = addEmbeddableBadgeConfiguration(id: "whitesource", subject: "WhiteSource")
def ppmsBadge = addEmbeddableBadgeConfiguration(id: "ppms", subject: "PPMS")

def FAILED_STAGE = 'Unknown'

pipeline {
    agent { label 'slave-d' }
    options { skipDefaultCheckout() }

    stages {
        stage('Quality') {
            failFast false
            stages {
                stage('Prepare') {
                    steps {
                        checkout scm
                        setupPipelineEnvironment script: this
                        sh "mkdir -p ${WORKSPACE}/coverage"
                        sh "mkdir -p ${WORKSPACE}/reports"
                        sh 'mkdir -p ${WORKSPACE}/whitesource'
                        sh 'mkdir -p ${WORKSPACE}/ppms'
                        sh 'mkdir -p ${WORKSPACE}/xmake'
                    }
                }
                stage('ESLint') {
                    steps {
                        script {
                            qualityBadge.setStatus('running')
                        }
                        durationMeasure(script: this, measurementName: 'eslint_duration') {
                            dockerExecute(script: this, dockerImage: 'docker.wdf.sap.corp:50001/com.sap.cai/node-dev:10.15.1-alpine-build-1') {
                                sh "npm install"
                                sh "npm run lint"
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: '**/eslint.jslint.xml', allowEmptyArchive: true
                             checksPublishResults script: this, eslint: [
                                active: true,
                                archive: true,
                                pattern: '**/eslint.jslint.xml',
                                thresholds: [fail: [ high: 0]]
                            ]
                        }
                    }
                }
                stage('npm audit') {
                    steps {
                        durationMeasure(script: this, measurementName: 'npmaudit_duration') {
                            dockerExecute(script: this, dockerImage: 'docker.wdf.sap.corp:50001/com.sap.cai/node-dev:10.15.1-alpine-build-1') {
                                withEnv(["NPM_CONFIG_PREFIX=${env.WORKSPACE}/.npm-global"]) {
                                    sh "npm i -g npm-audit-html"
                                    sh "npm i -g npm@6.12.0"
                                    sh ".npm-global/bin/npm --version"
                                    //Should be removed when the 'uglifyjs-webpack-plugin' is updated with safer version of 'serialize-javascript' dependency
                                    sh "npx npm-force-resolutions"
                                    sh ".npm-global/bin/npm audit --json --production >> npm-audit.json"
                                }
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "**/npm-audit.json", allowEmptyArchive: true
                            testsPublishResults script: this, html: [
                                active: true,
                                allowEmpty: true,
                                archive: true,
                                file: "**/npm-audit.json",
                                name: 'npm-audit'
                            ]
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        FAILED_STAGE = env.STAGE_NAME
                    }
                }
                aborted {
                    script {
                        qualityBadge.setStatus('aborted')
                    }
                }
                failure {
                    script {
                        qualityBadge.setStatus('failing')
                    }
                }
                success {
                    script {
                        qualityBadge.setStatus('passing')
                    }
                }
                unstable {
                    script {
                        qualityBadge.setStatus('unstable')
                    }
                }
            }
        }
        stage('Tests') {
            steps {
               lock(resource: "${env.JOB_NAME}/10", inversePrecedence: true) {
                   milestone 10
                   script {
                       testsBadge.setStatus('running')
                   }
                   checkout scm
                   setupPipelineEnvironment script: this
                   durationMeasure(script: this, measurementName: 'tests_duration') {
                       sh 'npm run testHtml && npm run coverage:clover && npm run coverage:cobertura && npm run coverage:lcov && npm run coverage:html'
                   }
               }
            }
            post {
               always {
                   testsPublishResults(
                       script: this,
                       junit: [
                           active:true,
                           allowEmptyResults:true,
                           archive: true,
                           pattern: '**/reports/mocha.xml',
                           updateResults: true
                       ],
                       cobertura: [
                           active:true,
                           allowEmptyResults:true,
                           archive:true,
                           pattern: '**/coverage/cobertura/cobertura-coverage.xml'
                       ],
                       html: [
                           active:true,
                           allowEmptyResults:true,
                           archive:true,
                           name: 'NYC/Mocha',
                           path: '**/coverage/html'
                       ],
                       lcov: [
                           active:true,
                           allowEmptyResults:true,
                           archive:true,
                           name: 'LCOV Coverage',
                           path: '**/coverage/lcov/lcov-report'
                       ]
                   )
                   junit 'reports/mocha.xml'
                   cobertura coberturaReportFile: 'coverage/cobertura/cobertura-coverage.xml'
                   script {
                       FAILED_STAGE = env.STAGE_NAME
                   }
               }
               aborted {
                   script {
                       testsBadge.setStatus('aborted')
                   }
               }
               failure {
                   script {
                       testsBadge.setStatus('failing')
                   }
               }
               success {
                   script {
                       testsBadge.setStatus('passing')
                   }
               }
               unstable {
                   script {
                       testsBadge.setStatus('unstable')
                   }
               }
            }
        }
        stage('SonarQube') {
            when {
                anyOf {
                    branch 'PR-*'
                    branch 'master'
                    branch 'fix_use-new-piper'
                }
            }
            steps {
                lock(resource: "${env.JOB_NAME}/20", inversePrecedence: true) {
                    milestone 20
                    checkout scm
                    setupPipelineEnvironment script: this
                    script {
                        sonarQubeBadge.setStatus('running')
                        def package_json = readJSON file: 'package.json'
                        env.VERSION_TXT = package_json['version']
                    }
                    durationMeasure(script: this, measurementName: 'sonarqube_pr_voter_duration') {
                        artifactPrepareVersion script:this, commitVersion: false
                        pipelineStashFiles(script: this) {
                            sonarExecuteScan(script: this, isVoter: true, projectVersion: "${env.VERSION_TXT}", instance: 'SAP_EE_sonar', githubTokenCredentialsId: 'sonarqube_voter')
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        FAILED_STAGE = env.STAGE_NAME
                    }
                }
                aborted {
                    script {
                        sonarQubeBadge.setStatus('aborted')
                    }
                }
                failure {
                    script {
                        sonarQubeBadge.setStatus('failing')
                    }
                }
                success {
                    script {
                        sonarQubeBadge.setStatus('passing')
                    }
                }
                unstable {
                    script {
                        sonarQubeBadge.setStatus('unstable')
                    }
                }
            }
        }
        stage('xMake Build') {
            when {
                // Don't run in nightly builds
                expression { return !params.STATIC_CODE_ANALYSIS }

                // Only run for master, release, or hotfix
                anyOf {
                    //branch 'PR-*'
                    branch 'master'
                }
            }
            steps {
                lock(resource: "${env.JOB_NAME}/30", inversePrecedence: true) {
                    milestone 30
                    dir('xmake') {
                      script {
                        xMakeBuildBadge.setStatus('running')
                      }
                      checkout scm
                      setupPipelineEnvironment script: this
                      durationMeasure(script: this, measurementName: 'xmake_build_duration') {
                        script {
                            if (env.BRANCH_NAME == 'master') {
                                env.XMAKEBUILDQUALITY = 'Milestone'
                                artifactPrepareVersion script: this, versioningTemplate: '${version}-${timestamp}'
                            }
                        }
                        pipelineStashFiles(script: this, stashIncludes: [buildDescriptor: '**/**']) {
                            executeBuild script: this, gitCommitId: commonPipelineEnvironment.gitCommitId, buildType: 'xMakeStage', xMakeBuildQuality: env.XMAKEBUILDQUALITY
                            executeBuild script: this, gitCommitId: commonPipelineEnvironment.gitCommitId, buildType: 'xMakePromote', xMakeBuildQuality: env.XMAKEBUILDQUALITY
                        }
                      }
                    }
                }
            }
            post {
                always {
                    script {
                        FAILED_STAGE = env.STAGE_NAME
                    }
                }
                aborted {
                    script {
                        xMakeBuildBadge.setStatus('aborted')
                    }
                }
                failure {
                    script {
                        xMakeBuildBadge.setStatus('failing')
                    }
                }
                success {
                    script {
                        xMakeBuildBadge.setStatus('passing')
                    }
                }
                unstable {
                    script {
                        xMakeBuildBadge.setStatus('unstable')
                    }
                }
            }
        }
        //stage('Docker Build') {
//    agent { label 'master' }
//    when {
//        // Don't run in nightly builds
//        expression { return !params.STATIC_CODE_ANALYSIS }

        // Only run for master, release, or hotfix
//        anyOf {
//            branch 'master'
//        }
//    }
//    steps {
//        lock(resource: "${env.JOB_NAME}/40", inversePrecedence: true) {
//            milestone 40
//            script {
//                dockerBuildBadge.setStatus('running')
//            }
//            checkout scm
//            setupPipelineEnvironment script: this
//            durationMeasure(script: this, measurementName: 'docker_build_duration') {
//                build job: '/SAP-Conversational-AI-Docker/ML-CAI-assistantui-web/master', parameters: [string(name: 'ARTIFACT_VERSION', value: globalPipelineEnvironment.getArtifactVersion()), string(name: 'XMAKE_BUILD_QUALITY', value: env.XMAKEBUILDQUALITY)], wait: false
//            }
//        }
//    }
//    post {
//        always {
//            script {
//                FAILED_STAGE = env.STAGE_NAME
//            }
//        }
//        aborted {
//            script {
//                dockerBuildBadge.setStatus('aborted')
//            }
//        }
//        failure {
//            script {
//                dockerBuildBadge.setStatus('failing')
//            }
//        }
//        success {
//            script {
//                dockerBuildBadge.setStatus('passing')
//            }
//        }
//        unstable {
//            script {
//                dockerBuildBadge.setStatus('unstable')
//            }
//        }
//    }
//}
        stage('Checkmarx') {
            when {
                anyOf {
                    //branch 'PR-*'
                    branch 'master'
                    branch 'fix_use-new-piper'
                }
            }
            steps {
                lock(resource: "${env.JOB_NAME}/50", inversePrecedence: true) {
                    milestone 50
                    script {
                        checkmarxBadge.setStatus('running')
                    }
                    checkout scm
                    setupPipelineEnvironment script: this
                    durationMeasure(script: this, measurementName: 'checkmarx_duration') {
                        checkmarxExecuteScan script: this
                    }
                }
            }
            post {
                always {
                    script {
                        FAILED_STAGE = env.STAGE_NAME
                    }
                }
                aborted {
                    script {
                        checkmarxBadge.setStatus('aborted')
                    }
                }
                failure {
                    script {
                        checkmarxBadge.setStatus('failing')
                    }
                }
                success {
                    script {
                        checkmarxBadge.setStatus('passing')
                    }
                }
                unstable {
                    script {
                        checkmarxBadge.setStatus('unstable')
                    }
                }
            }
        }
        stage('WhiteSource') {
            when {
                anyOf {
                    branch 'master'
                    branch 'PR-*'
                    branch 'fix_use-new-piper'
                }
            }
            steps {
                lock(resource: "${env.JOB_NAME}/60", inversePrecedence: true) {
                    milestone 60
                    sh 'mkdir -p ${WORKSPACE}/whitesource'
                    dir('whitesource') {
                        checkout scm
                        setupPipelineEnvironment script: this
                        script {
                            whiteSourceBadge.setStatus('running')
                        }
                        pipelineStashFiles(script: this, stashIncludes: [buildDescriptor: '**/**']) {
                            durationMeasure(script: this, measurementName: 'whitesource_duration') {
                                whitesourceExecuteScan script: this, scanType: 'npm', whitesourceProjectNames: ['ml-cai-webchat - current']
                            }
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        FAILED_STAGE = env.STAGE_NAME
                    }
                }
                aborted {
                    script {
                        whiteSourceBadge.setStatus('aborted')
                    }
                }
                failure {
                    script {
                        whiteSourceBadge.setStatus('failing')
                    }
                }
                success {
                    script {
                        whiteSourceBadge.setStatus('passing')
                    }
                }
                unstable {
                    script {
                        whiteSourceBadge.setStatus('unstable')
                    }
                }
            }
        }
        stage('PPMS Check') {
            when {
                anyOf {
                    // Always run for release and hotfix
                    branch 'master'
                    branch 'PR-*'
                    branch 'fix_use-new-piper'
                }
            }
            steps {
                lock(resource: "${env.JOB_NAME}/80", inversePrecedence: true) {
                    milestone 80
                    sh 'mkdir -p ${WORKSPACE}/ppms'
                    dir('ppms') {
                        checkout scm
                        setupPipelineEnvironment script: this
                        script {
                          ppmsBadge.setStatus('running')
                        }
                        pipelineStashFiles(script: this, stashIncludes: [buildDescriptor: '**/**']) {
                            durationMeasure(script: this, measurementName: 'ppms_duration') {
                                sapCheckPPMSCompliance script: this, scanType: 'whitesource', whitesourceProjectNames: ['ml-cai-webchat - current']
                            }

                          }
                    }
                }
            }
            post {
                always {
                    script {
                        FAILED_STAGE = env.STAGE_NAME
                    }
                }
                aborted {
                    script {
                        ppmsBadge.setStatus('aborted')
                    }
                }
                failure {
                    script {
                        ppmsBadge.setStatus('failing')
                    }
                }
                success {
                    script {
                        ppmsBadge.setStatus('passing')
                    }
                }
                unstable {
                    script {
                        ppmsBadge.setStatus('unstable')
                    }
                }
            }
        }
    }
    post {
        always {
          script {
            try {
              sh 'mkdir ./down'
              dir('./down') {
                checkout scm
                sh 'make down'
              }
            } catch(e) {
                echo e.getMessage()
              }
          }
          mailSendNotification script: this
          cleanWs()
        }
        failure {
            script {
                if(env.BRANCH_NAME in ['master']){
                    slackSend (
                        color: '#D50000',
                        message: "Failed Job: ${env.JOB_NAME} (${env.BUILD_NUMBER}) [Stage: ${FAILED_STAGE}]: ${env.BUILD_URL}"
                    )
                }
            }
        }
        success {
            script {
                if(env.BRANCH_NAME in ['master']){
                    slackSend (
                        color: '#64DD17',
                        message: "Successful Job: ${env.JOB_NAME} (${env.BUILD_NUMBER}): ${env.BUILD_URL}"
                    )
                }
            }
        }
    }
}
