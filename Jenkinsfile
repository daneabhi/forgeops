/*
 * Copyright 2020-2022 ForgeRock AS. All Rights Reserved
 *
 * Use of this code requires a commercial software license with ForgeRock AS.
 * or with one of its affiliates. All use shall be exclusively subject
 * to such license between the licensee and ForgeRock AS.
 */

//============================================================================
// ForgeOps pipeline to run long running perf tests for sprint release testing
//============================================================================

import java.net.URLDecoder

import org.jenkinsci.plugins.workflow.steps.FlowInterruptedException

@Library([
    'forgerock-pipeline-libs@a4f6060382c4edd70bedfd33adfa87120772fb53',
    'java-pipeline-libs@7d909d2ffb9ab751dc96e9c7bc9d253d3d993dbb'
])
import com.forgerock.pipeline.Build
import com.forgerock.pipeline.reporting.PipelineRun
import com.forgerock.pipeline.reporting.PipelineRunLegacyAdapter

def pipelineRun

def jobProperties = [
    disableConcurrentBuilds(),
    buildDiscarder(logRotator(numToKeepStr: '20')),
    parameters([
        booleanParam(name: 'PerfSprintRelease_authn_rest', defaultValue: true, description: 'approx 6h'),
        booleanParam(name: 'PerfSprintRelease_access_token', defaultValue: true, description: 'approx 6h'),
        booleanParam(name: 'PerfSprintRelease_platform', defaultValue: true, description: 'approx 6h'),
        booleanParam(name: 'PerfSprintRelease_simple_managed_users', defaultValue: true, description: 'approx 6h'),
    ])
]

if (env.TAG_NAME) {
    currentBuild.result = 'ABORTED'
    error 'This pipeline does not currently support building from a tag\nFor support, email releng@forgerock.com'
} else if (isPR()) {
    currentBuild.result = 'ABORTED'
    error 'Please check your Multibranch Pipeline configuration for this job' +
            '- it should not include settings that allow this build to be run from a PR.\n' +
            'For support, email releng@forgerock.com'
} else if (env.BRANCH_NAME.equals('master') || env.BRANCH_NAME.startsWith('idcloud-') || env.BRANCH_NAME.equals('sustaining/7.1.x')) {
    properties(jobProperties)
} else {
    // safety guard, to prevent non-master branches from building
    currentBuild.result = 'ABORTED'
    error 'Only master PaaS release branches are allowed to run long Perf tests .\n' +
            'For support, email releng@forgerock.com'
}

timestamps {
    manageNotifications {
        node('build&&linux') {
            stage('Setup') {
                checkout scm

                def stagesLocation = "${env.WORKSPACE}/jenkins-scripts/stages"
                def libsLocation = "${env.WORKSPACE}/jenkins-scripts/libs"

                localGitUtils = load("${libsLocation}/git-utils.groovy")
                commonModule = load("${libsLocation}/common.groovy")

                // Load the QaCloudUtils dynamically based on Lodestar commit promoted to Forgeops
                library "QaCloudUtils@${commonModule.LODESTAR_GIT_COMMIT}"

                def FORGEOPS_GIT_COMMIT = commonModule.FORGEOPS_GIT_COMMIT
                def FORGEOPS_GIT_COMMITTER = commonModule.FORGEOPS_GIT_COMMITTER
                def FORGEOPS_GIT_MESSAGE = commonModule.FORGEOPS_GIT_MESSAGE
                def FORGEOPS_GIT_COMMITTER_DATE = commonModule.FORGEOPS_GIT_COMMITTER_DATE
                def FORGEOPS_GIT_BRANCH = commonModule.FORGEOPS_GIT_BRANCH
                def currentProductCommitHashes = commonModule.getCurrentProductCommitHashes()
                if (commonModule.FORGEOPS_GIT_BRANCH.endsWith('-stable')) {
                    FORGEOPS_GIT_COMMIT = sh(script: 'git rev-list --max-count 1 --skip 1 HEAD', returnStdout: true).trim()
                    FORGEOPS_GIT_COMMITTER = sh(returnStdout: true, script: "git show ${FORGEOPS_GIT_COMMIT} -s --pretty=%cn").trim()
                    FORGEOPS_GIT_MESSAGE = sh(returnStdout: true, script: "git show ${FORGEOPS_GIT_COMMIT} -s --pretty=%s").trim()
                    FORGEOPS_GIT_COMMITTER_DATE = sh(returnStdout: true, script: "git show ${FORGEOPS_GIT_COMMIT} -s --pretty=%cd --date=iso8601").trim()
                    FORGEOPS_GIT_BRANCH = commonModule.FORGEOPS_GIT_BRANCH.replaceAll('-stable', '')
                    currentProductCommitHashes.forgeops = FORGEOPS_GIT_COMMIT
                }

                pipelineRun = new PipelineRunLegacyAdapter(PipelineRun.builder(env, steps)
                        .pipelineName('forgeops-perf-sprint-release')
                        .branch(FORGEOPS_GIT_BRANCH)
                        .commit(FORGEOPS_GIT_COMMIT)
                        .commits(currentProductCommitHashes)
                        .committer(FORGEOPS_GIT_COMMITTER)
                        .commitMessage(FORGEOPS_GIT_MESSAGE)
                        .committerDate(dateTimeUtils.convertIso8601DateToInstant(FORGEOPS_GIT_COMMITTER_DATE))
                        .repo('forgeops')
                        .build())

                // Test stages
                perfSprintReleaseTests = load("${stagesLocation}/perf-sprint-release-tests.groovy")

                currentBuild.displayName = "#${BUILD_NUMBER} - ${commonModule.FORGEOPS_SHORT_GIT_COMMIT}"

                echo "Testing ForgeOps commit ${commonModule.FORGEOPS_SHORT_GIT_COMMIT} " +
                        "(${commonModule.FORGEOPS_GIT_COMMIT})"
                echo "Using Lodestar commit ${commonModule.LODESTAR_GIT_COMMIT} for the tests"
            }
        }

        perfSprintReleaseTests.runStage(pipelineRun)

        currentBuild.result = 'SUCCESS'
    }
}

/**
 * Manage the build notifications.
 * @param notificationsEnabled Quickly disable notifications by setting this value to @code{false}.
 * @param body The build script.
 */
def manageNotifications(boolean notificationsEnabled = true, Closure body) {
    def slackChannel = '#performance-notify'
    def sprintReleaseBuild = new Build(steps, env, currentBuild)
    try {
        body() // perform the build
        if (notificationsEnabled) {
            slackUtils.sendMessage(
                slackChannel,
                " ${URLDecoder.decode(env.JOB_NAME)} #${env.BUILD_NUMBER} passed on " +
                        "commit ${commonModule.FORGEOPS_SHORT_GIT_COMMIT} " +
                        "from ${env.BRANCH_NAME} (<${env.BUILD_URL}|Open>)",
                slackUtils.colour('SUCCESS')
            )
        }
    } catch (FlowInterruptedException ex) {
        currentBuild.result = 'ABORTED'
        throw ex
    } catch (exception) {
        currentBuild.result = 'FAILURE'
        if (notificationsEnabled) {
            slackUtils.sendNoisyStatusMessage(slackChannel)
        }
        throw exception
    }
}
