#!/usr/bin/env groovy
import com.ipsy.buddy.DeploymentTarget;

node("slave") {
    def LABEL = "[PUPPET]"

    stage("Prep for per-dir Jenkinsfile") {
      checkout(scm)
      sh 'git clean -f -d'
    }

    utils.runChangedDirs([
      "misc/terraform": {
        echo "terraforming it up with branch ${BRANCH_NAME}"
        load "Jenkinsfile"
      },
      "gems": {
          echo "generating gems it up with branch ${BRANCH_NAME}"
          load "Jenkinsfile"
      },
    ])

    def gem_stub_path = "${WORKSPACE}/gem-bin"
    def scl_ruby = "scl enable rh-ruby23 -- "
    def lint_opts = "--no-documentation-check --show-ignored --fail-on-warnings"
    withEnv(["PATH+GEM=${gem_stub_path}"]) {

        stage("${LABEL} Dependencies") {
            sh("${scl_ruby} gem install bundle")
            sh("${scl_ruby} bundle install --verbose --binstubs ${gem_stub_path}")
        }

        def ipsymodules = stage("${LABEL} Find Modules") {
            def modules = utils.capture('ls -1 modules').split(/\n/)
            def thirdpartymodules = utils.capture("${scl_ruby} librarian-puppet show")
                    .split(/\n/)
                    .collect { it.split().first().split(/-/).last() }
                    .unique()

            modules - thirdpartymodules
        }

        stage("${LABEL} Validate") {
            dir('modules') {
                def failedModules = ipsymodules.collectEntries { mod ->
                    [mod, sh(returnStatus: true, name: mod, script: "${scl_ruby} puppet parser validate ${mod}") == 0]
                }.findAll { key, value -> !value }

                if (!failedModules.isEmpty()) {
                    print "Modules failing validation: ${failedModules.keySet().join(" ")}"
                    error('SYNTAX VALIDATION FAILED')
                }
            }
        }

        stage("${LABEL} Lint") {
            dir('modules') {
                def uglyModules = ipsymodules.collectEntries { mod ->
                    [mod, sh(returnStatus: true, name: mod, script: "${scl_ruby} puppet-lint ${lint_opts} ${mod}") == 0]
                }.findAll { key, value -> !value }

                if (!uglyModules.isEmpty()) {
                    print "Modules failing linting: ${uglyModules.keySet().join(" ")}"
                    error('QUALITY CHECK FAILED')
                }
            }
        }

        stage("${LABEL} Bundle") {
            def versionLabel = version_label()
            def applicationName = 'puppet'
            def builtFile = "build.zip"

            def appDef = [
                    applicationName: applicationName,
                    skip_puppet_run: true,
                    runtime        : [type: 'bash', version: 'ENOCARE'],
                    commands       : [
                            deploy_puppet: '/usr/sbin/deploy-puppet-code',
                    ],
            ]

            sh "rm -rf ${gem_stub_path}"
            sh "rm -rf ./vendor"
            sh "rm -rf *@*"
            sh "zip --exclude=*.terraform* -x build -r ${builtFile} *"

            bundleKey = bundles.upload([
                    path                 : builtFile,
                    bundleKey            : versionLabel,
                    applicationDefinition: appDef
            ])

            if (BRANCH_NAME == "develop") {
                targets = get_deployment_targets(applicationName)
                deploy(
                        applicationName: applicationName,
                        bundleKey: bundleKey,
                        targets: targets
                )
            }

            //Compatibility with old puppet servers
            s3MirrorGit { }
        }
    }
}

def get_deployment_targets(appName) {
    def deployableAccounts = ["SHARED_STAGING_LOCKDOWN","SHARED_STAGING"]
    targets = awsDeploy.listTargets(appName)
    return targets.findAll{ tg ->
        deployableAccounts.find{ account -> account == tg.account} != null
    }
}

def version_label() {
    def gitCommit = utils.capture('git rev-parse --short HEAD')
    def name = "puppet"
    "${name}_${BRANCH_NAME}_${gitCommit}"
}
