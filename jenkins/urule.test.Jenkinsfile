pipeline {
    agent any

    parameters {
        gitParameter(
            name: 'BRANCH',
            type: 'PT_BRANCH', 
            defaultValue: 'dev_gh', 
            description: '选择要部署的 Git 分支',
            branch: '', 
            useRepository: 'git@github.com:bjx-code-backend/tanzania_loan.git'
        )
        string(name: 'SERVER_NAME', defaultValue: 'urule-springboot', description: '模块名称')
        string(name: 'deployment_name', defaultValue: 'urule-ghana', description: 'Deployment 名称')
        choice(name: 'DEPLOY_TYPE', choices: ['Deploy', 'Rollback'], description: '操作类型：Deploy=部署新版本，Rollback=回滚')
    }

    environment {
        REGISTRY = 'harbor.bjxsre.com'
        PROJECT  = 'bjx-ghana-test'
        GIT_REPO = 'git@github.com:bjx-code-backend/tanzania_loan.git'
        BUILD_VERSION = "${params.BRANCH.replaceFirst(/^origin\\//,'')}-${env.BUILD_NUMBER}"
        IMAGE_FULL = "${REGISTRY}/${PROJECT}/${params.deployment_name}:${BUILD_VERSION}"
 
        CHAT_DIR = "./bjx-helm/charts/urule"
        JAR_PATH = "urule-springboot/target/urule.jar"
        NAMESPACE = "ghana"
    }

    stages {
        stage('版本回滚') {
            when { expression { params.DEPLOY_TYPE == 'Rollback' } }
            steps {
                script {
                    def versions = sh(
                        script: """
                            helm history ${params.deployment_name} -n ${env.NAMESPACE} -o json \
                            | jq -r '.[] | select(.status=="deployed") | .app_version' \
                            | grep -v null
                        """,
                        returnStdout: true
                    ).trim().split("\\n").findAll { it }.reverse()


                    if (versions.isEmpty()) {
                        error "没有可回滚的历史版本！"
                    }

                    echo "可回滚版本：\n${versions.join('\n')}"

                    def selectedVersion = input(
                        message: "请选择要回滚的 appVersion",
                        parameters: [choice(name: 'APP_VERSION', choices: versions, description: '历史版本')]
                    )

                    sh """
                        REVISION=\$(helm history ${params.deployment_name} -n ${env.NAMESPACE} -o json | jq -r '.[] | select(.app_version=="${selectedVersion}") | .revision')
                        if [ -z "\$REVISION" ]; then
                            echo "找不到版本 ${selectedVersion}"
                            exit 1
                        fi
                        kubectl rollout resume deployment/${release} -n ${env.NAMESPACE}
                        helm rollback ${params.deployment_name} \$REVISION -n ${env.NAMESPACE}
                        kubectl rollout status deployment/${params.deployment_name} -n ${env.NAMESPACE} --timeout=5m
                    """
                }
            }
        }

        stage('代码拉取') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: "${params.BRANCH.startsWith('origin/') ? params.BRANCH : "*/${params.BRANCH}"}"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [],
                    userRemoteConfigs: [[
                        url: "${env.GIT_REPO}",
                        credentialsId: 'GIT_CREDENTIALS'
                    ]]
                ])
            }
        }

        stage('MVN打包') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                sh """
                    rm -rf bjx-helm
                    git clone https://github.com/Elio-li/bjx-helm.git
                    sed -i 's|urule.mysql.url=jdbc:mysql://127.0.0.1:3306/ghana_loan?useSSL=false|urule.mysql.url=jdbc:mysql://bjx-hk-test.cluster-cbuwkmuwoycy.ap-east-1.rds.amazonaws.com/ghana_loan?useSSL=false|' urule-springboot/src/main/resources/ghana/application-dev.properties
                    sed -i 's|urule.mysql.username=root|urule.mysql.username=admin|' urule-springboot/src/main/resources/ghana/application-dev.properties
                    sed -i 's|urule.mysql.password=9skLyjBrvnqmCltkeqrazfqfoxc20:|urule.mysql.password=D4mFXq5fscAFh4tf49v6|' urule-springboot/src/main/resources/ghana/application-dev.properties
                    #mvn clean install -pl ${params.SERVER_NAME} -am -Dmaven.test.skip=true
                    
                """
                steps {
                    archiveArtifacts artifacts: "${env.JAR_PATH}", fingerprint: true
                   }
            }
        }

        stage('制作Dockerfile') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def dockerfile = """
                    FROM eclipse-temurin:8-jdk
                    COPY ${env.JAR_PATH} /app/urule.jar
                    WORKDIR /app
                    EXPOSE 8080
                    ENTRYPOINT [ "/app/urule.jar"]
                    """.stripIndent()
                    writeFile file: 'Dockerfile', text: dockerfile
                    echo "Dockerfile 已生成"
                }
            }
        }

        stage('打包镜像') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Harbor_id',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh """
                        echo "${HARBOR_PASS}" | docker login ${REGISTRY} -u "${HARBOR_USER}" --password-stdin
                        docker build -t ${IMAGE_FULL} .
                        docker push ${IMAGE_FULL}
                        docker logout ${REGISTRY}
                    """
                }
            }
        }


        stage('Helm检查') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def CHART_DIR = env.CHAT_DIR
                    def VALUES_FILE = "${CHART_DIR}/urule-ghana-test.yaml"
                    def BUILD_TAG = env.BUILD_VERSION

                    // 更新 values & Chart appVersion
                    sh """
                        sed -i "s|^  tag:.*|  tag: ${BUILD_TAG}|" ${VALUES_FILE}
                        sed -i "s|^appVersion:.*|appVersion: \\"${BUILD_TAG}\\"|" ${CHART_DIR}/Chart.yaml
                    """

                    // 检查 Deployment 是否存在
                    def exists = sh(script: "kubectl get deploy ${RELEASE} -n ${NS} >/dev/null 2>&1 && echo true || echo false", returnStdout: true).trim()
                    env.IS_FIRST_DEPLOY = (exists != 'true').toString()

                    // 获取副本数
                    if (!env.IS_FIRST_DEPLOY.toBoolean()) {
                        def replicas = sh(script: "kubectl get deploy ${RELEASE} -n ${NS} -o jsonpath='{.spec.replicas}' || echo 0", returnStdout: true).trim()
                        env.REPLICAS = replicas ?: '0'
                    } else {
                        env.REPLICAS = '0'
                    }
                }
            }
        }

        stage('Helm deploy') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def CHART_DIR = env.CHAT_DIR
                    def VALUES_FILE = "${CHART_DIR}/urule-ghana-test.yaml"

                    if (env.IS_FIRST_DEPLOY.toBoolean() || env.REPLICAS.toInteger() <= 0) {
                        echo "首次部署或副本数为0，全量部署"
                        sh "helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} --namespace ${NS} --wait --timeout=10m"
                        echo "部署完成"
                    } else {
                        echo "Deployment存在，触发滚动更新（不等待全部 Ready）"
                        sh "helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} --namespace ${NS} --timeout=5m"
                    }
                }
            }
        }

        stage('等待pod') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' && env.REPLICAS.toInteger() > 0 } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def BUILD_TAG = env.BUILD_VERSION
                    def newPodName = ''
                    def pollStart = System.currentTimeMillis()
                    def pollTimeoutMs = 3 * 60 * 1000

                    while ((System.currentTimeMillis() - pollStart) < pollTimeoutMs) {
                        def podList = sh(
                            script: "kubectl get pods -n ${NS} -l app=${RELEASE} -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image --no-headers",
                            returnStdout: true
                        ).trim()

                        podList.split("\n").each { line ->
                            def (name, image) = line.tokenize(' ')
                            if (image.contains("${BUILD_TAG}")) {
                                newPodName = name
                                return
                            }
                        }
                        if (newPodName) break
                        sleep(time: 5, unit: 'SECONDS')
                    }

                    if (!newPodName) {
                        def action = input(message: "3分钟内未检测到新 Pod，操作选择:", parameters: [choice(name:'ACTION', choices:['继续等待','回滚'], description:'')])
                        if (action == '回滚') {
                            rollbackDeployment(RELEASE, NS)
                            error("已回滚（未检测到新 Pod）")
                        }
                    }

                    echo "检测到新 Pod: ${newPodName}，暂停 Deployment"
                    sh "kubectl rollout pause deployment/${RELEASE} -n ${NS}"
                    env.CANARY_POD = newPodName
                }
            }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def newPod = env.CANARY_POD
                    def podReady = false

                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                def ready = sh(
                                    script: "kubectl get pod ${newPod} -n ${NS} -o jsonpath='{.status.containerStatuses[0].ready}' 2>/dev/null || echo false",
                                    returnStdout: true
                                ).trim()
                                return (ready == 'true')
                            }
                        }
                        podReady = true
                    } catch(e) {
                        podReady = false
                        echo "❌ 新 Pod 未在 5 分钟内 Ready"
                    }

                    if (!podReady) {
                        def action = input(message:"新 Pod 未 Ready，操作选择:", parameters:[choice(name:'ACTION', choices:['回滚','人工处理'])])
                        if (action == '回滚') {
                            rollbackDeployment(RELEASE, NS)
                            error("已回滚")
                        } else {
                            sh "kubectl rollout resume deployment/${RELEASE} -n ${NS}"
                            error("人工处理退出")
                        }
                    }
                }
            }
        }



        stage('继续更新或者回滚') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' && env.REPLICAS.toInteger() > 0 } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def userChoice = input(
                        message: "第一个 Pod Ready，是否继续更新剩余 Pod 或回滚？",
                        parameters: [choice(name: 'ACTION', choices: ['继续更新（恢复）','回滚'])]
                    )
                    if (userChoice == '回滚') {
                        rollbackDeployment(RELEASE, NS)
                        error("已回滚到上一版本")
                    }

                    echo "▶️ 继续更新剩余 Pod..."
                    sh "kubectl rollout resume deployment/${RELEASE} -n ${NS}"
                    sh "kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=10m"
                    echo "🎉 部署完成：所有 Pod 已更新到 ${env.BUILD_VERSION}"
                }
            }
}

    }

    post {
        always {
            
            echo "构建完成：${env.IMAGE_FULL}"
        }
        success {
            echo "部署成功！"
        }
        failure {
            echo "部署失败"
        }
    }
}

// ==================== 回滚函数 ====================
def rollbackDeployment(String release, String ns) {
    echo "开始回滚到上一版本..."
    def prevRev = sh(
        script: "helm history ${release} -n ${ns} -o json | jq -r '.[-2].revision // empty'",
        returnStdout: true
    ).trim()

    if (prevRev) {
        sh """
            helm rollback ${release} ${prevRev} -n ${ns}
            kubectl rollout status deployment/${release} -n ${ns} --timeout=5m
            kubectl rollout resume deployment/${release} -n ${ns}
        """
        echo "已回滚到 revision ${prevRev}"
    } else {
        echo "无历史版本可回滚"
    }
}
