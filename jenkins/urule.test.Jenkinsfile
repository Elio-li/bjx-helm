pipeline {
    agent any

    parameters {
        string(name: 'GIT_REPO', defaultValue: 'git@github.com:bjx-code-backend/tanzania_loan.git', description: 'Git 仓库地址')
        string(name: 'BRANCH', defaultValue: 'dev_gh', description: 'Git 分支')
        string(name: 'SERVER_NAME', defaultValue: 'urule-springboot', description: '模块名称')
        string(name: 'service', defaultValue: 'urule', description: '服务名')
        string(name: 'deployment_name', defaultValue: 'urule-ghana', description: 'Deployment 名称')
        choice(name: 'DEPLOY_TYPE', choices: ['Deploy', 'Rollback'], description: '操作类型：Deploy=部署新版本，Rollback=回滚')
        choice(name: 'CANARY_STRATEGY', choices: ['1-pod', '30%', '50%', '100%'], description: '金丝雀策略：先更新多少比例？')
    }

    environment {
        REGISTRY = 'harbor.bjxsre.com'
        PROJECT  = 'bjx-ghana-test'
        BUILD_VERSION = "${params.BRANCH}-${env.BUILD_NUMBER}-${new Date().format('yyyyMMddHHmmss')}"
        IMAGE_FULL = "${REGISTRY}/${PROJECT}/${params.service}:${BUILD_VERSION}"
        CHAT_DIR = "./bjx-helm/charts/urule"
        JAR_PATH = "urule-springboot/target/urule.jar"
        NAMESPACE = "ghana"
    }

    stages {
        // ==================== 回滚阶段 ====================
        stage('Rollback Check') {
            when { expression { params.DEPLOY_TYPE == 'Rollback' } }
            steps {
                script {
                    def versions = sh(
                        script: "helm history ${params.deployment_name} -n ${env.NAMESPACE} -o json | jq -r '.[].app_version' | grep -v null",
                        returnStdout: true
                    ).trim().split("\n").findAll { it }.reverse()

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
                        helm rollback ${params.deployment_name} \$REVISION -n ${env.NAMESPACE}
                        kubectl rollout status deployment/${params.deployment_name} -n ${env.NAMESPACE} --timeout=5m
                    """
                }
            }
        }

        // ==================== 部署流程 ====================
        stage('Checkout') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                git branch: "${params.BRANCH}",
                    credentialsId: 'GIT_CREDENTIALS',
                    url: "${params.GIT_REPO}"
            }
        }

        stage('Build Jar') {
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
            }
        }

        stage('Generate Dockerfile') {
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

        stage('Build & Push Image') {
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

        stage('Archive Jar') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                archiveArtifacts artifacts: "${env.JAR_PATH}", fingerprint: true
            }
        }

        // ==================== 金丝雀部署 ====================
        stage('Helm Canary Deploy') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def CHART_DIR = env.CHAT_DIR
                    def VALUES_FILE = "${CHART_DIR}/urule-ghana-test.yaml"
                    def BUILD_TAG = env.BUILD_VERSION

                    // update values & chart appVersion
                    sh """
                        sed -i "s|^  tag:.*|  tag: ${BUILD_TAG}|" ${VALUES_FILE}
                        sed -i "s|^appVersion:.*|appVersion: \\"${BUILD_TAG}\\"|" ${CHART_DIR}/Chart.yaml
                    """

                    // check deployment existence
                    def exists = sh(script: "kubectl get deploy ${RELEASE} -n ${NS} >/dev/null 2>&1 && echo true || echo false", returnStdout: true).trim()
                    if (exists != 'true') {
                        echo "Deployment ${RELEASE} not found in namespace ${NS} — 执行首次全量部署"
                        sh """
                            helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} \
                                --namespace ${NS} --wait --timeout=10m
                        """
                        echo "首次部署完成"
                        return
                    }

                    // get replica count
                    def replicasRaw = sh(script: "kubectl get deploy ${RELEASE} -n ${NS} -o jsonpath='{.spec.replicas}' || echo 0", returnStdout: true).trim()
                    def replicas = 0
                    try { replicas = replicasRaw.toInteger() } catch(e) { replicas = 0 }
                    if (replicas <= 0) {
                        echo "Deployment 副本数为 0 — 执行全量部署"
                        sh """
                            helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} \
                                --namespace ${NS} --wait --timeout=10m
                        """
                        echo "全量部署完成"
                        return
                    }

                    echo "Deployment 存在，副本数=${replicas}. 开始 Helm 升级"
                    // 1) 触发 helm upgrade（不加 --wait）
                    sh """
                        helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} \
                            --namespace ${NS} --timeout=5m
                    """

                    // 2) 轮询等待出现第一个使用新镜像 tag 的 Pod
                    echo "等待第一个使用镜像 tag='${BUILD_TAG}' 的 Pod 出现..."
                    def newPodFound = false
                    def pollStart = System.currentTimeMillis()
                    def pollTimeoutMs = 3 * 60 * 1000  // 3 分钟内必须出现新 Pod，否则超时并提示人工决策（可以调整）
                    while ((System.currentTimeMillis() - pollStart) < pollTimeoutMs) {
                        def count = sh(
                            script: """
                                kubectl get pods -n ${NS} -l app=${RELEASE} -o jsonpath='{range .items[*]}{.metadata.name}::{.status.containerStatuses[0].image}{"\\n"}{end}' 2>/dev/null \
                                | awk -F '::' '{print \$2}' | grep -c '${BUILD_TAG}' || echo 0
                            """,
                            returnStdout: true
                        ).trim().toInteger()
                        if (count > 0) {
                            newPodFound = true
                            break
                        }
                        sleep(time: 5, unit: 'SECONDS')
                    }

                    if (!newPodFound) {
                        echo "⚠️ 在 3 分钟内未检测到任何新镜像的 Pod（tag=${BUILD_TAG})."
                        def action = input(
                            message: "未检测到新 Pod 启动，选择下一步操作：",
                            parameters: [choice(name: 'ACTION', choices: ['继续等待', '回滚'], description: '选择')]
                        )
                        if (action == '回滚') {
                            rollbackDeployment(RELEASE, NS)
                            error("已回滚（因为未检测到新 Pod）")
                        } else {
                            echo "继续等待中——你可以手工检查集群状态。"
                            // 继续等待一次短时周期（5 分钟），然后再决定（简化处理）
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    def c = sh(
                                        script: """
                                            kubectl get pods -n ${NS} -l app=${RELEASE} -o jsonpath='{range .items[*]}{.status.containerStatuses[0].image}{"\\n"}{end}' \
                                            | grep -c '${BUILD_TAG}' || echo 0
                                        """, returnStdout: true
                                    ).trim().toInteger()
                                    return c > 0
                                }
                            }
                        }
                    }

                    // 到此已检测到第一个新 Pod（或用户选择继续等待之后检测到）
                    echo "检测到使用新镜像的 Pod，立即暂停 Deployment 的滚动更新以阻止继续替换其余 Pods"
                    sh "kubectl rollout pause deployment/${RELEASE} -n ${NS}"

                    // 获取新 Pod 名称（第一个匹配 BUILD_TAG 的 Pod）
                    def newPodName = sh(
                        script: """
                            kubectl get pods -n ${NS} -l app=${RELEASE} -o jsonpath='{range .items[*]}{.metadata.name}::{.status.containerStatuses[0].image}{"\\n"}{end}' \
                            | awk -F '::' '{if (index(\$2, "${BUILD_TAG}")) print \$1;}' | head -n1
                        """,
                        returnStdout: true
                    ).trim()

                    if (!newPodName) {
                        echo "⚠️ 未能解析到新 Pod 名称（尽管检测到新镜像）。尝试继续并等待 Ready。"
                    } else {
                        echo "第一个新 Pod: ${newPodName}"
                    }

                    // 3) 等待该 Pod Ready（5 分钟超时）
                    def podReady = false
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                def ready = 0
                                if (newPodName) {
                                    ready = sh(
                                        script: "kubectl get pod ${newPodName} -n ${NS} -o jsonpath='{.status.containerStatuses[0].ready}' 2>/dev/null || echo false",
                                        returnStdout: true
                                    ).trim()
                                    return (ready == 'true')
                                } else {
                                    // 如果没有拿到 name，则通过 image 匹配至少有一个 ready 的 pod
                                    def readyCount = sh(
                                        script: """
                                            kubectl get pods -n ${NS} -l app=${RELEASE} -o jsonpath='{range .items[*]}{.status.containerStatuses[0].image}::{.status.containerStatuses[0].ready}{"\\n"}{end}' \
                                            | awk -F '::' '{if (index(\$1, "${BUILD_TAG}") && \$2==\"true\") print "ok"}' | wc -l
                                        """,
                                        returnStdout: true
                                    ).trim().toInteger()
                                    return readyCount > 0
                                }
                            }
                        }
                        podReady = true
                    } catch (e) {
                        podReady = false
                        echo "❌ 新 Pod 未在 5 分钟内 Ready."
                    }

                    if (!podReady) {
                        def action = input(
                            message: "新 Pod 在 5 分钟内未 Ready，选择操作：",
                            parameters: [choice(name: 'ACTION', choices: ['回滚', '继续等待/人工处理'], description: '选择')]
                        )
                        if (action == '回滚') {
                            rollbackDeployment(RELEASE, NS)
                            error("已回滚（因为新 Pod 未就绪）")
                        } else {
                            echo "你选择了继续等待/人工处理，请手动检查问题并在准备好后手动 resume（或在 Jenkins 中继续）。"
                            // 保持 paused 状态，结束 pipeline（或继续由人工在集群上处理）
                            error("暂停并等待人工处理（Deployment 保持 paused）")
                        }
                    }

                    echo "✅ 第一个新 Pod Ready 且通过检查。现在请确认是否继续更新剩余 Pod（恢复 rolling update）或回滚到上一版本。"
                    def userChoice = input(
                        message: "第一个 Pod 已就绪，下一步：",
                        parameters: [choice(name: 'ACTION', choices: ['继续更新（恢复并完成）', '回滚到上一版本'], description: '选择')]
                    )

                    if (userChoice == '回滚到上一版本') {
                        rollbackDeployment(RELEASE, NS)
                        error("已回滚到上一版本（用户选择）")
                    }

                    // 若用户选择继续：恢复并等待完整滚动更新完成
                    echo "▶️ 恢复 Deployment（rolling update），并等待所有 Pod 更新完成..."
                    sh "kubectl rollout resume deployment/${RELEASE} -n ${NS}"

                    // 等待 rollout 完成（设置合理 timeout）
                    sh """
                        kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=10m
                    """

                    echo "🎉 部署完成：所有 Pod 已更新到 ${BUILD_TAG}"
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
        """
        echo "已回滚到 revision ${prevRev}"
    } else {
        echo "无历史版本可回滚"
    }
}