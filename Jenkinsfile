def cron_spec = BRANCH_NAME == 'master' ? 'H 0 * * *' : ''
properties([
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '30')),
    pipelineTriggers([cron(cron_spec)])
])

node('osx && ios') {
    def contributors = null
    def Utils
    def buildLabel
    currentBuild.result = "SUCCESS"

    stage ('Checkout Source') {
        deleteDir()
        checkout scm
        getUtils()
        // load pipeline utility functions
        Utils = load "utils/Utils.groovy"
        buildLabel = Utils.&getBuildLabel()
    }

    stage ('Create Change Logs') {
        sshagent(['38bf8b09-9e52-421a-a8ed-5280fcb921af']) {
            try {
                Utils.&copyArtifactWhenAvailable("Cogosense/iOSNordicDfuFramework/${env.BRANCH_NAME}", 'SCM/CHANGELOG', 1, 0)
            }
            catch(err) {}

            dir('./SCM') {
                sh '../utils/scmBuildDate > TIMESTAMP'
                writeFile file: "TAG", text: buildLabel
                writeFile file: "URL", text: env.BUILD_URL
                writeFile file: "BRANCH", text: env.BRANCH_NAME
                sh '../utils/scmBuildContributors > CONTRIBUTORS'
                sh '../utils/scmBuildOnHookEmail > ONHOOK_EMAIL'
                sh "../utils/scmUpdateChangeLog -t ${buildLabel} -o CHANGELOG"
                sh '../utils/scmTagLastBuild'
            }
        }
    }

    try {
        contributors = readFile './SCM/ONHOOK_EMAIL'

        stage ('Capture Build Environment') {
            sh 'env -u PWD -u HOME -u PATH -u \'BASH_FUNC_copy_reference_file()\' > SCM/build.env'
        }

        stage ('Notify Build Started') {
            Utils.&sendOnHookEmail(contributors)
        }

        stage ('Build') {
            withEnv(['PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin']) {
                Utils.&upgradeBrew()
                Utils.&brewUpstall('carthage')
                Utils.&brewUpstall('swiftlint')
                sh 'make'
            }
        }

        stage ('Archive Artifacts') {
            // Archive the SCM logs, the framework directory
            step([$class: 'ArtifactArchiver',
                artifacts: 'SCM/**, *.framework.tar.bz2',
                fingerprint: true,
                onlyIfSuccessful: true])
        }

        stage ('Tag Build') {
            sshagent(['38bf8b09-9e52-421a-a8ed-5280fcb921af']) {
                sh "utils/scmTagBuild ${buildLabel}"
            }
        }

    } catch(err) {
        currentBuild.result = "FAILURE"
        Utils.&sendFailureEmail(contributors, err)
        throw err
    }

    stage ('Notify Build Completion') {
        Utils.&sendOffHookEmail(contributors)
    }
}

def getUtils() {
    // Load the SCM util scripts first
    checkout([$class: 'GitSCM',
        branches: [[name: "*/${env.BRANCH_NAME}"]],
        doGenerateSubmoduleConfigurations: false,
        extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'utils']],
        submoduleCfg: [],
        userRemoteConfigs: [[url: 'git@github.com:Cogosense/JenkinsUtils.git', credentialsId: '38bf8b09-9e52-421a-a8ed-5280fcb921af']]])
}
