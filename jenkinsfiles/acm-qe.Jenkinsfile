pipeline {
    agent {
        docker {
            image 'quay.io/stolostron/acm-qe:submariner-fedora36-nodejs18'
            registryUrl 'https://quay.io/stolostron/acm-qe'
            registryCredentialsId '0089f10c-7a3a-4d16-b5b0-3a2c9abedaa2'
            args '--network host'
        }
    }
    options {
        ansiColor('xterm')
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        timeout(time: 8, unit: 'HOURS')
    }
    parameters {
        string(name: 'OC_CLUSTER_API', defaultValue: '', description: 'ACM Hub API URL')
        string(name: 'OC_CLUSTER_USER', defaultValue: '', description: 'ACM Hub username')
        string(name: 'OC_CLUSTER_PASS', defaultValue: '', description: 'ACM Hub password')
        extendedChoice(name: 'PLATFORM', description: 'The managed clusters platform that should be tested',
            value: 'aws,gcp,azure,vsphere', defaultValue: 'aws,gcp,azure,vsphere', multiSelectDelimiter: ',', type: 'PT_CHECKBOX')
        booleanParam(name: 'GLOBALNET', defaultValue: true, description: 'Deploy Globalnet on Submariner')
        booleanParam(name: 'DOWNSTREAM', defaultValue: true, description: 'Deploy downstream version of Submariner')
        booleanParam(name: 'SUBMARINER_GATEWAY_RANDOM', defaultValue: true, description: 'Deploy two submariner gateways on one of the clusters')
        string(name:'TEST_TAGS', defaultValue: '', description: 'A tag to control job execution')
        string(name: 'BROWSER', defaultValue: '', description: 'Specify the browser for cypress testing (by default uses chrome)')
        booleanParam(name: 'POLARION', defaultValue: true, description: 'Publish tests results to Polarion')
        choice(name: 'JOB_STAGES', choices: ['all', 'deploy', 'test'], description: 'Select stage that should be executed by the job')
    }
    environment {
        EXECUTE_JOB = false
        OC_CLUSTER_API = "${params.OC_CLUSTER_API}"
        OC_CLUSTER_USER = "${params.OC_CLUSTER_USER}"
        OC_CLUSTER_PASS = "${params.OC_CLUSTER_PASS}"
        // Parameter will be used to disable globalnet in
        // ACM version below 2.5.0 as it's not supported
        GLOBALNET_TRIGGER = true
        // The secret contains polarion authentication
        // and other details for report publish
        POLARION_SECRET = credentials('submariner-polarion-secret')
    }
    stages {
        // This stage will validate the environment for the job.
        // If the prerequisites will not met, the job will not be
        // executed to avoid non submariner job failures.
        stage('Validate prerequisites') {
            when {
                anyOf {
                    // The job flow will be executed only if TEST_TAGS parameter
                    // will be empty or definited with the values below.
                    environment name: 'TEST_TAGS', value: ''
                    environment name: 'TEST_TAGS', value: '@e2e'
                    environment name: 'TEST_TAGS', value: '@Submariner'
                    environment name: 'TEST_TAGS', value: '@post-release'
                    environment name: 'TEST_TAGS', value: '@api'
                    environment name: 'TEST_TAGS', value: '@api-post-release'
                    environment name: 'TEST_TAGS', value: '@post-restore'
                    environment name: 'TEST_TAGS', value: '@pre-upgrade'
                    environment name: 'TEST_TAGS', value: '@post-upgrade'
                }
            }
            steps {
                sh """
                ./run.sh --validate-prereq --platform "${params.PLATFORM}"

                # In acm-qe jenkins, logs stored in a PV, so they are persistent over jobs.
                # Delete "logs/" dir so in case the job skipped, previous job results will not be reported.
                echo "Make sure older logs deleted"
                rm -rf logs/
                """

                script {
                    def state = readFile(file: 'validation_state.log')
                    println(state)
                    // If validation_state.log file contains string "Not ready!",
                    // meaning environment prerequisites are not ready.
                    // The job will not be executed.
                    if (state.contains('Not ready!')) {
                        EXECUTE_JOB = false
                        currentBuild.result = 'NOT_BUILT'
                    } else {
                        EXECUTE_JOB = true
                    }

                    def mch_ver = readFile(file: 'mch_version.log')
                    println("MultiClusterHub version: " + mch_ver)
                    mch_ver_major = mch_ver.split('\\.')[0] as Integer
                    mch_ver_minor = mch_ver.split('\\.')[1] as Integer

                    // Checks the version of the MultiClusterHub
                    // If the version is below 2.5.0, globalnet
                    // is not supported - disable it.
                    // Otherwise, use parameter definition.
                    globalnet_ver = '2.5'
                    globalnet_ver_major = globalnet_ver.split('\\.')[0] as Integer
                    globalnet_ver_minor = globalnet_ver.split('\\.')[1] as Integer
                    if (mch_ver_major < globalnet_ver_major ||
                        mch_ver_minor < globalnet_ver_minor) {
                            println("Disable Globalnet as it's not supported in ACM " + mch_ver)
                            GLOBALNET_TRIGGER = false
                        }

                    // Backup/Restore feature added to for Submariner in version 2.7
                    // If backup/resore scenario triggered and ACM version lower than 2.7, skip the job
                    backup_restore_ver = '2.7'
                    backup_restore_ver_major = backup_restore_ver.split('\\.')[0] as Integer
                    backup_restore_ver_minor = backup_restore_ver.split('\\.')[1] as Integer
                    if (params.TEST_TAGS == '@post-restore') {
                        if (mch_ver_major < backup_restore_ver_major ||
                            mch_ver_minor < backup_restore_ver_minor) {
                                println("Backup/Restore scenario triggered but not supported in ACM " + mch_ver)
                                EXECUTE_JOB = false
                            } else {
                                println("Backup/Restore scenario triggered")
                            }
                    }
                }
            }
        }
        stage('Deploy') {
            when {
                allOf {
                    expression {
                        EXECUTE_JOB == true
                    }
                    expression {
                        JOB_STAGES == 'all' || JOB_STAGES == 'deploy'
                    }
                }
            }
            steps {
                script {
                    GLOBALNET = "--globalnet ${params.GLOBALNET}"
                    // The "GLOBALNET_TRIGGER" will be used as a
                    // control point to for ACM versions below 2.5.0
                    // As it's not supported.
                    if (GLOBALNET_TRIGGER.toBoolean() == false) {
                        GLOBALNET = "--globalnet false"
                    }

                    DOWNSTREAM = "--downstream false"
                    if (params.DOWNSTREAM) {
                        DOWNSTREAM = "--downstream true"
                    }

                    // Deploy two submariner gateways on one of the clusters
                    // to test submariner gateway fail-over scenario.
                    SUBMARINER_GATEWAY_RANDOM = "--subm-gateway-random false"
                    if (params.SUBMARINER_GATEWAY_RANDOM) {
                        SUBMARINER_GATEWAY_RANDOM = "--subm-gateway-random true"
                    }

                    // The '@post-release' tag meant to test post GA release
                    // thus don't use the downstream tag.
                    // Override the any state of the DOWNSTREAM param.
                    if (params.TEST_TAGS == '@post-release') {
                        DOWNSTREAM = "--downstream false"
                    }

                    // The FLOW controls execution of the "Deploy" stage.
                    // By default - "--deploy"
                    // When '@pre-upgrade' tag provided, it will use "--subm-catalog-update" to update the Submariner catalog source
                    FLOW = "--deploy"
                    if (params.TEST_TAGS == '@pre-upgrade') {
                        FLOW = "--subm-catalog-update"
                        println("Upgrade job is triggered")
                    }
                }

                sh """
                ./run.sh $FLOW --platform "${params.PLATFORM}" $GLOBALNET $DOWNSTREAM $SUBMARINER_GATEWAY_RANDOM
                """
            }
        }
        stage('Test') {
            when {
                allOf {
                    expression {
                        EXECUTE_JOB == true
                    }
                    expression {
                        JOB_STAGES == 'all' || JOB_STAGES == 'test'
                    }
                }
            }
            steps {
                script {
                    DOWNSTREAM = "--downstream false"
                    if (params.DOWNSTREAM) {
                        DOWNSTREAM = "--downstream true"
                    }

                    // The '@post-release' tag meant to test post GA release
                    // thus don't use the downstream tag.
                    // Override the any state of the DOWNSTREAM param.
                    if (params.TEST_TAGS == '@post-release') {
                        DOWNSTREAM = "--downstream false"
                    }

                    TESTS_TYPE = "--test-type e2e,ui"
                    if (params.TEST_TAGS == '@api' ||
                        params.TEST_TAGS == '@api-post-release') {
                            TESTS_TYPE = "--test-type e2e"
                        }

                    TEST_BROWSER = ""
                    if (params.BROWSER != '') {
                        TEST_BROWSER = "--test-browser ${params.BROWSER}"
                    }

                    REPORT_SUFFIX = ""
                    if (params.TEST_TAGS == '@post-restore') {
                        REPORT_SUFFIX = "--report-suffix backup-restore"
                    }

                    if (params.TEST_TAGS == '@post-upgrade') {
                        REPORT_SUFFIX = "--report-suffix upgrade"
                    }
                }

                sh """
                ./run.sh --test --platform "${params.PLATFORM}" $DOWNSTREAM $TESTS_TYPE $TEST_BROWSER $REPORT_SUFFIX
                """
            }
        }
        stage('Report to Polarion') {
            when {
                allOf {
                    expression {
                        EXECUTE_JOB == true
                    }
                    expression {
                        JOB_STAGES == 'all' || JOB_STAGES == 'test'
                    }
                }
            }
            steps {
                script {
                    POLARION = ""
                    if (params.POLARION) {
                        POLARION = "--polarion-vars-file ${POLARION_SECRET}"
                    }
                }

                sh """
                ./run.sh --report $POLARION
                """
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: "logs/**/*.*", followSymlinks: false, allowEmptyArchive: true
            junit allowEmptyResults: true, testResults: "logs/**/**/*junit.xml"
        }
    }
}
