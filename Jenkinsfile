#!/usr/bin/env groovy
import com.ipsy.buddy.DeploymentTarget;

node("slave") {
    def LABEL = "[EKS]"

    stage("Prep for per-dir Jenkinsfile") {
      checkout(scm)
      sh 'git clean -f -d'
    }

    withEnv([
            "PATH+KUBECTL=${tool 'kubectl-1.19.3'}",
    ]) {

        stage("${LABEL} Dependencies") {
            sh("kubectl version --client")
        }
    }
}
