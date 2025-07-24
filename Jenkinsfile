#!groovy
node {
    def BUILD_NUMBER = env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR = "tests/${BUILD_NUMBER}"

    def JWT_KEY_CRED_ID = env.JWT_CRED_ID_DH
    def HUB_ORG = env.HUB_ORG_DH
    def SFDC_HOST = env.SFDC_HOST_DH
    def CONNECTED_APP_CONSUMER_KEY = env.CONNECTED_APP_CONSUMER_KEY_DH

    def toolbelt = tool 'toolbelt'

    stage('Checkout Source') {
        checkout scm
    }

    def changedFiles = []
    stage('Identify Changed Metadata') {
        changedFiles = bat(
            script: 'git diff-tree --no-commit-id --name-only -r HEAD',
            returnStdout: true
        ).trim().split('\r?\n')

        echo "Changed Files: ${changedFiles.join(', ')}"

        // Filter Salesforce metadata components only
        changedFiles = changedFiles.findAll {
            it.startsWith("force-app") && !it.endsWith(".xml")
        }

        if (changedFiles.isEmpty()) {
            error 'No Salesforce metadata components to deploy.'
        }
    }

    withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {

        stage('Authorize DevHub Org') {
            echo "Using JWT Auth for DevHub"
            echo "Hub Org: ${HUB_ORG}"
            echo "Connected App Consumer Key: ${CONNECTED_APP_CONSUMER_KEY}"

            def checkrc = bat returnStatus: true, script: """
                ${toolbelt}sf org login jwt --instance-url ${SFDC_HOST} --client-id ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwt-key-file %jwt_key_file% --setalias Devhub
            """.stripIndent()

            if (checkrc != 0) {
                error 'DevHub org authorization failed.'
            } else {
                echo 'DevHub authorized successfully.'
            }
        }

        stage('Deploy to DevHub') {
            def fileList = changedFiles.collect { "\"${it}\"" }.join(' ')
            def deployCmd = "${toolbelt}sf project deploy start --target-org Devhub --source-dir ${fileList} --wait 10 --ignore-warnings"

            echo "Deploying changed metadata to DevHub: ${fileList}"

            def rc = bat returnStatus: true, script: deployCmd
            if (rc != 0) {
                error 'Push to DevHub org failed.'
            } else {
                echo 'Push to DevHub org successful.'
            }
        }
    }
}
