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
                    mvn clean install -pl ${params.SERVER_NAME} -am -Dmaven.test.skip=true
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
                    ENTRYPOINT ["/app/urule.jar"]
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
                    def STRATEGY = params.CANARY_STRATEGY

                    // 更新镜像 tag
                    sh """
                        sed -i "s|^  tag:.*|  tag: ${env.BUILD_VERSION}|" ${VALUES_FILE}
                        sed -i "s|^appVersion:.*|appVersion: \"${env.BUILD_VERSION}\"|" ${CHART_DIR}/Chart.yaml
                    """

                    // 获取当前副本数
                    def replicas = sh(
                        script: "kubectl get deployment ${RELEASE} -n ${NS} -o jsonpath='{.spec.replicas}'",
                        returnStdout: true
                    ).trim().toInteger()

                    if (replicas <= 0) error "副本数为 0，无法部署！"

                    // 计算批次
                    def batchSizes = []
                    if (STRATEGY == '1-pod') {
                        batchSizes = [1]
                    } else if (STRATEGY == '30%') {
                        def first = Math.max(1, (replicas * 0.3).round())
                        batchSizes = [first, replicas - first]
                    } else if (STRATEGY == '50%') {
                        def half = (replicas * 0.5).round()
                        batchSizes = [half, replicas - half]
                    } else if (STRATEGY == '100%') {
                        batchSizes = [replicas]
                    }

                    echo "部署策略: ${STRATEGY} → 分批大小: ${batchSizes}"

                    // 分批部署
                    for (int i = 0; i < batchSizes.size(); i++) {
                        def batchSize = batchSizes[i]
                        def isLast = (i == batchSizes.size() - 1)

                        echo "第 ${i+1} 批：更新 ${batchSize} 个 Pod"

                        // 设置滚动策略
                        sh """
                            kubectl patch deployment ${RELEASE} -n ${NS} -p '{
                                "spec": {"strategy": {"rollingUpdate": {"maxSurge": ${batchSize}, "maxUnavailable": 0}}}
                            }' || true
                        """

                        // 触发升级
                        sh """
                            helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} --namespace ${NS} --wait --timeout=5m
                        """

                        // 关键：5分钟超时检测 + 人工确认回滚
                        def batchReady = false
                        try {
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    def readyCount = sh(
                                        script: """
                                            kubectl get pods -n ${NS} -l app=${RELEASE} \
                                            -o jsonpath='{range .items[?(@.spec.containers[0].image=~".*${env.BUILD_VERSION}.*")]}{.status.containerStatuses[0].ready}{"\\n"}{end}' | grep -c true || echo 0
                                        """,
                                        returnStdout: true
                                    ).trim().toInteger()
                                    def result = readyCount >= batchSize
                                    if (result) {
                                        echo "第 ${i+1} 批 ${batchSize} 个 Pod 已就绪"
                                    }
                                    return result
                                }
                            }
                            batchReady = true
                        } catch (err) {
                            echo "第 ${i+1} 批 Pod 5分钟内未全部就绪！"
                        }

                        // 超时或未就绪 → 强制人工确认是否回滚
                        if (!batchReady) {
                            def rollbackChoice = input(
                                message: "第 ${i+1} 批 Pod 启动失败或超时，是否立即回滚？",
                                parameters: [
                                    choice(name: 'ROLLBACK_ACTION', choices: ['立即回滚', '继续等待（不推荐）'], description: '选择操作')
                                ]
                            )

                            if (rollbackChoice == '立即回滚') {
                                rollbackDeployment(RELEASE, NS)
                                error("第 ${i+1} 批启动失败，用户选择回滚，部署终止")
                            } else {
                                echo "用户选择继续等待，但仍需手动确认后续"
                                // 继续，但仍需人工确认下一批
                            }
                        } else {
                            echo "第 ${i+1} 批 ${batchSize} 个 Pod 正常就绪"
                        }

                        // 非最后一轮，正常人工确认（即使超时也走这里）
                        if (!isLast) {
                            def action = input(
                                message: "第 ${i+1} 批处理完成，是否继续下一批？",
                                parameters: [choice(name: 'ACTION', choices: ['继续下一批', '停止并回滚'], description: '选择')]
                            )
                            if (action == '停止并回滚') {
                                rollbackDeployment(RELEASE, NS)
                                error("用户主动取消部署，已回滚")
                            }
                        }
                    }

                    echo "所有 Pod 更新完成！"
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