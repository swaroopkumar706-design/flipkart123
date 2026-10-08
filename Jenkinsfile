pipeline {
    agent any

    parameters {
        
        booleanParam(name: 'DEV_JSON',   defaultValue: false, description: 'node/dev.json')
        booleanParam(name: 'PROD_JSON',  defaultValue: false, description: 'node/prod.json')
        booleanParam(name: 'STAGE_JSON', defaultValue: false, description: 'node/stage.json')
        booleanParam(name: 'UAT_JSON',   defaultValue: false, description: 'node/uat.json')

        // ---- Values ----
        string(name: 'P_ENVIRONMENT',   defaultValue: '', description: '"environment" (e.g. dev)')
        string(name: 'P_NODE_NAME',     defaultValue: '', description: '"nodeName" (e.g. node-02)')
        string(name: 'P_NODE_TYPE',     defaultValue: '', description: '"nodeType" (e.g. worker)')
        string(name: 'P_REGION',        defaultValue: '', description: '"region" (e.g. us-east-1)')
        string(name: 'P_AZ',            defaultValue: '', description: '"availabilityZone" (e.g. us-east-1b)')
        string(name: 'P_INSTANCE_TYPE', defaultValue: '', description: '"instanceType" (e.g. t3.large)')
        string(name: 'P_OS',            defaultValue: '', description: '"os" (e.g. Amazon Linux 2023)')
        string(name: 'P_K8S_ROLE',      defaultValue: '', description: '"kubernetes.role" (e.g. worker)')
        string(name: 'P_K8S_VERSION',   defaultValue: '', description: '"kubernetes.version" (e.g. 1.36)')
        string(name: 'P_CPU',           defaultValue: '', description: '"resources.cpu" (e.g. 2)')
        string(name: 'P_MEMORY',        defaultValue: '', description: '"resources.memory" (e.g. 4Gi)')
        string(name: 'P_DISK',          defaultValue: '', description: '"resources.disk" (e.g. 50Gi)')
        string(name: 'P_LABEL_ENV',     defaultValue: '', description: '"labels.environment" (e.g. dev)')
        string(name: 'P_LABEL_TEAM',    defaultValue: '', description: '"labels.team" (e.g. platform)')
    }

    environment {
    REPO = 'https://github.com/swaroopkumar706-design/flipkart.git'
    BRANCH = 'main'
}

    stages {
        stage('Select Files') {
            steps {
                script {
                    def files = []
                    if (params.DEV_JSON)   { files << 'node/dev.json' }
                    if (params.PROD_JSON)  { files << 'node/prod.json' }
                    if (params.STAGE_JSON) { files << 'node/stage.json' }
                    if (params.UAT_JSON)   { files << 'node/uat.json' }
                    if (files.isEmpty()) {
                        error('select one file (dev/prod/stage/uat) tick ')
                    }
                    env.FILES = files.join(' ')
                    echo "Update files: ${env.FILES}"
                }
            }
        }

        stage('Checkout') {
    steps {
        cleanWs()

        git(
            branch: "${env.BRANCH}",
            url: "${env.REPO}",
            credentialsId: 'github-credentials'
        )
    }
}

        stage('Update JSON') {
            steps {
                sh '''
                    set -e
                    cat > filter.jq <<'JQ'
def p($k): ($ENV[$k] // "");
def setstr($path; $k): if p($k) != "" then setpath($path; p($k)) else . end;
setstr(["environment"]; "P_ENVIRONMENT")
| setstr(["nodeName"]; "P_NODE_NAME")
| setstr(["nodeType"]; "P_NODE_TYPE")
| setstr(["region"]; "P_REGION")
| setstr(["availabilityZone"]; "P_AZ")
| setstr(["instanceType"]; "P_INSTANCE_TYPE")
| setstr(["os"]; "P_OS")
| setstr(["kubernetes","role"]; "P_K8S_ROLE")
| setstr(["kubernetes","version"]; "P_K8S_VERSION")
| setstr(["resources","cpu"]; "P_CPU")
| setstr(["resources","memory"]; "P_MEMORY")
| setstr(["resources","disk"]; "P_DISK")
| setstr(["labels","environment"]; "P_LABEL_ENV")
| setstr(["labels","team"]; "P_LABEL_TEAM")
JQ
                    for f in $FILES; do
                        [ -f "$f" ] || { echo "File not available: $f"; exit 1; }
                        jq -f filter.jq "$f" > tmp.json
                        mv tmp.json "$f"
                        echo "=== Updated $f ==="
                        cat "$f"
                    done
                    rm -f filter.jq
                '''
            }
        }

stage('Commit & Push') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'github-credentials',
                usernameVariable: 'GIT_USER',
                passwordVariable: 'GIT_TOKEN'
            )
        ]) {
            sh '''
                set -e

                git config user.name "Jenkins"
                git config user.email "jenkins@example.com"

                echo "======================================"
                echo "Git Status"
                echo "======================================"

                git add $FILES
                git status

                if git diff --cached --quiet
                then
                    echo "No changes detected."
                    exit 0
                fi

                echo "======================================"
                echo "Creating Commit"
                echo "======================================"

                git commit \
                  -m "[skip ci] Jenkins: updated $FILES (build #${BUILD_NUMBER})"

                echo "======================================"
                echo "Pull Latest Changes"
                echo "======================================"

                git pull --rebase origin "$BRANCH"

                echo "======================================"
                echo "Configure Git Authentication"
                echo "======================================"

                git config credential.helper \
                  '!f() {
                      echo username=$GIT_USER;
                      echo password=$GIT_TOKEN;
                  }; f'

                echo "======================================"
                echo "Push Changes"
                echo "======================================"

                git push origin "HEAD:$BRANCH"

                echo "======================================"
                echo "Push Successful"
                echo "======================================"
            '''
        }
    }
}
}
}
