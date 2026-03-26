@Library('commons@main') _

pipeline {
    agent {
        label 'agent-1'
    }

    parameters {
        string(name: 'GIT_REPOSITORY_BRANCH', defaultValue: 'master', description: 'Docker image tag')
        booleanParam(name: 'DRY_RUN', defaultValue: true, description: 'Run Ansible in check mode')
        string(name: 'EXTRA_ARGS', defaultValue: '', description: 'Ansible extra variable')
    }

    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    stages{
        stage('Validate Parameters') {
            steps {
                script {
                    assert params.GIT_REPOSITORY_BRANCH?.trim() : "GIT_REPOSITORY_BRANCH parameter is required and cannot be empty"
                }
            }
        }
        stage('Run Ansible Playbook') {
            steps {
                script {
                    def playbookPath = 'ansible/main.yml'
                    def inventory = params.GIT_REPOSITORY_BRANCH.contains('master') ? 'ansible/inventories/production' : 'ansible/inventories/staging'
                    def baseVersion = sh(script: "git tag --sort=-v:refname | head -n 1", returnStdout: true).trim()
                    echo "Latest git tag: ${baseVersion}"
                    env.IMAGE_TAG = resolveBaseTag(baseVersion)
                    echo "Resolved base Docker tag: ${env.IMAGE_TAG}"
                    echo "Mocking ansible playbook..."
                    echo "Ansible success!"
                }
            }
        }
        stage('Publish API Documentation') {
            when {
                allOf {
                    expression { !params.DRY_RUN }
                    expression { params.GIT_REPOSITORY_BRANCH == 'master' }
                }
            }
            steps {
                script {
                    echo "Publishing api documentation"
                }
            }
        }
        stage('Trigger Integration Test') {
            steps {
                script {
                    if (!params.DRY_RUN) {
                        currentBranch = params.GIT_REPOSITORY_BRANCH.tokenize('/').last()
                        echo "Triggering integration test pipeline for branch ${currentBranch}..."
                    } else {
                        echo "Dry run is enabled; skipping integration test trigger."
                    }
                }
            }
        }
        stage('Trigger Smoke Test') {
            steps {
                script {
                    if (!params.DRY_RUN) {
                        def environment = params.GIT_REPOSITORY_BRANCH.contains('master') ? 'production' : 'staging'
                        echo "Triggering smoke test pipeline for environment: ${environment}..."
                    } else {
                        echo "Dry run is enabled; skipping smoke test trigger."
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                currentBuild.result = currentBuild.result ?: 'SUCCESS'

                def dryRunStatus = params.DRY_RUN
                def targetHosts = params.GIT_REPOSITORY_BRANCH == 'master' ? 'production' : 'staging'
                def extras = params.EXTRA_ARGS

                currentBuild.description = """
                    <b>Branch:</b> ${params.GIT_REPOSITORY_BRANCH}<br/>
                    <b>Image Tag:</b> ${env.IMAGE_TAG}<br/>
                    <b>Dry Run:</b> ${dryRunStatus}<br/>
                    <b>Environment:</b> ${targetHosts}<br/>
                    <b>Extra arguments:</b> ${extras}
                """.stripIndent()
            }
        }
    }
}

def resolveBaseTag(version){
  // Resolves the tag that will be used to pull the image (depending on the branch)
  if (params.GIT_REPOSITORY_BRANCH == "master"){
    // if branch = master, then the base tag must be the “<version>
    return "${version}"
  } else if (params.GIT_REPOSITORY_BRANCH == "staging"){
    // if branch = staging, then the base tag must be the “<version>-staging”
    return "${version}-staging"
  } else {
    // if it is a feature branch, then the pipeline must PULL images with a tag that corresponds to “git describe --tags” (ex: 1.0.1-1-g22c39cc)
    return sh(script: "git describe --tags", returnStdout: true).trim()
  }
}