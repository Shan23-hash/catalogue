@Library('jenkins-shared-library') _

def configMap = [
    project  : "roboshop",
    component: "catalogue"
]

if (!(env.BRANCH_NAME ?: "").equalsIgnoreCase('main')) {
    nodejsEKSPipeline(configMap)
}
else {
    echo "Please proceed with PROD process"
}