#!/usr/bin/env groovy

pipeline {
    agent { label 'gelman-group-linux' }
    options {
        skipDefaultCheckout()
        preserveStashes(buildCount: 5)
    }
    parameters {
        string(defaultValue: '', name: 'major_version', description: "Major version of the docs to be built")
        string(defaultValue: '', name: 'minor_version', description: "Minor version of the docs to be built")
        string(defaultValue: '', name: 'last_docs_version_dir', description: "Last docs version found in /docs. Example: 2_21")
        booleanParam(defaultValue: true, description: 'Build docs and create a PR ?', name: 'buildDocs')
        booleanParam(defaultValue: true, description: 'Change docs version for stan-dev.github.io ?', name: 'docsStanDev')
        booleanParam(defaultValue: false, description: 'Build cmdstan manual ?', name: 'cmdstanManual')
    }
    environment {
        GITHUB_TOKEN = credentials('6e7c1e8f-ca2c-4b11-a70e-d934d3f6b681')
    }
    stages {
        stage("Checkout docs, build and create PR") {
            when {
              expression {
                params.buildDocs
              }
            }
            steps{
                /* Checkout source code */
                deleteDir()
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/master']],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [],
                          submoduleCfg: [],
                          userRemoteConfigs: [[url: "https://github.com/stan-dev/docs.git", credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b']]]
                )

                /* Create a new branch */
                withCredentials([usernamePassword(credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b',
                    usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh """#!/bin/bash
                        git checkout -b docs-$major_version-$minor_version
                    """
                }

                /* Build docs */
                sh "python build.py $major_version $minor_version"

                /* Add redirects */
                sh "python add_redirects.py $major_version $minor_version functions-reference"
                sh "python add_redirects.py $major_version $minor_version reference-manual"
                sh "python add_redirects.py $major_version $minor_version stan-users-guide"

                /* Link docs to latest */
                sh "ls -lhart docs"
                sh "chmod +x add_links.sh"
                sh "./add_links.sh $last_docs_version_dir"

                /* Create Pull Request */
                withCredentials([usernamePassword(credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b',
                    usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh """#!/bin/bash
                        git config --global user.email "mc.stanislaw@gmail.com"
                        git config --global user.name "Stan Jenkins"
                        git config --global auth.token ${GITHUB_TOKEN}

                        git add .
                        git commit -m "Documentation generated from Jenkins for docs-$major_version-$minor_version"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/stan-dev/docs.git docs-$major_version-$minor_version

                        curl -s -H "Authorization: token ${GITHUB_TOKEN}" -X POST -d '{"title": "Docs generated by Jenkins for v$major_version-$minor_version", "head":"docs-$major_version-$minor_version", "base":"master", "body":"Docs generated through the [Jenkins Job](https://jenkins.mc-stan.org/job/BuildDocs/)"}' "https://api.github.com/repos/stan-dev/docs/pulls"
                    """
                }
            }
        }
        stage('Checkout stan-dev.github.io and update docs version') {
            when {
              expression {
                params.docsStanDev
              }
            }
            steps {
                /* Checkout source code */
                deleteDir()
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/master']],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [],
                          submoduleCfg: [],
                          userRemoteConfigs: [[url: "https://github.com/stan-dev/stan-dev.github.io.git", credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b']]]
                )

                /* Switch to a new branch */
                withCredentials([usernamePassword(credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh """#!/bin/bash
                        git checkout -b docs-$major_version-$minor_version
             
                    """
                }

                /* Format version numbers with dot and underscore then replace the old values */
                script {
                    def last_version_parts = last_docs_version_dir.split("_")
                    def last_version_dot = last_version_parts[0] + "." + last_version_parts[1]
                    def new_version_underscore = major_version + "_" + minor_version
                    def new_version_dot = major_version + "." + minor_version

                    sh """
                        sed -i 's/$last_docs_version_dir/$new_version_underscore/g' users/documentation/index.md
                        sed -i 's/$last_version_dot/$new_version_dot/g' users/documentation/index.md

                        sed -i 's/$last_docs_version_dir/$new_version_underscore/g' users/interfaces/cmdstan.md
                        sed -i 's/$last_version_dot/$new_version_dot/g' users/interfaces/cmdstan.md
                    """

                }

                /* Create a merge request */
                withCredentials([usernamePassword(credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b',
                    usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh """#!/bin/bash
                        git config --global user.email "mc.stanislaw@gmail.com"
                        git config --global user.name "Stan Jenkins"
                        git config --global auth.token ${GITHUB_TOKEN}

                        git add .
                        git commit -m "Documentation generated from Jenkins for docs-$major_version-$minor_version"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/stan-dev/stan-dev.github.io.git docs-$major_version-$minor_version

                        curl -s -H "Authorization: token ${GITHUB_TOKEN}" -X POST -d '{"title": "Docs updated by Jenkins for v$major_version-$minor_version", "head":"docs-$major_version-$minor_version", "base":"master", "body":"Docs updated through the [Jenkins Job](https://jenkins.mc-stan.org/job/BuildDocs/)"}' "https://api.github.com/repos/stan-dev/stan-dev.github.io/pulls"
                    """
                }

            }
        }
        stage('Checkout cmdstan and build manual') {
            when {
              expression {
                params.cmdstanManual
              }
            }
            steps {
                /* Checkout source code */
                deleteDir()
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/master']],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [],
                          submoduleCfg: [],
                          userRemoteConfigs: [[url: "https://github.com/stan-dev/cmdstan.git", credentialsId: 'a630aebc-6861-4e69-b497-fd7f496ec46b']]]
                )

                /* make manual */
                sh "make manual"

                /* archive manual for it to be shown in job result artifacts */
                archiveArtifacts 'doc/*.pdf'
            }
        }
    }
}
