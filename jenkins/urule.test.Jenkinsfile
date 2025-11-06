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
        CHART_DIR = "./bjx-helm/charts/urule"
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
                    def CHART_DIR = env.CHART_DIR
                    def VALUES_FILE = "${CHART_DIR}/urule-ghana-test.yaml"
                    def STRATEGY = params.CANARY_STRATEGY
                    def IMAGE_TAG = env.BUILD_VERSION

                    echo "========================================"
                    echo "开始金丝雀部署"
                    echo "Release: ${RELEASE}"
                    echo "Namespace: ${NS}"
                    echo "Image Tag: ${IMAGE_TAG}"
                    echo "Strategy: ${STRATEGY}"
                    echo "========================================"

                    // 更新 values 和 Chart.yaml
                    sh """
                        sed -i "s|^  tag:.*|  tag: ${IMAGE_TAG}|" ${VALUES_FILE}
                        sed -i "s|^appVersion:.*|appVersion: \\"${IMAGE_TAG}\\"|" ${CHART_DIR}/Chart.yaml
                    """

                    // 获取当前副本数
                    def replicas = sh(
                        script: "kubectl get deployment ${RELEASE} -n ${NS} -o jsonpath='{.spec.replicas}' 2>/dev/null || echo '0'",
                        returnStdout: true
                    ).trim().toInteger()
                    
                    if (replicas <= 0) {
                        error "副本数为 0 或 Deployment 不存在"
                    }

                    // 计算分批策略
                    def batchSizes = []
                    if (STRATEGY == '1-pod') {
                        batchSizes = [1]
                    } else if (STRATEGY == '30%') {
                        def first = Math.max(1, (replicas * 0.3).round() as Integer)
                        batchSizes = [first, replicas - first]
                    } else if (STRATEGY == '50%') {
                        def half = Math.max(1, (replicas * 0.5).round() as Integer)
                        batchSizes = [half, replicas - half]
                    } else {
                        batchSizes = [replicas]
                    }

                    echo "总副本数: ${replicas}"
                    echo "分批策略: ${batchSizes}"

                    // 首次 helm upgrade（触发变更）
                    sh "helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} --namespace ${NS}"

                    // 立即暂停滚动更新
                    sh "kubectl rollout pause deployment/${RELEASE} -n ${NS} || true"
                    sleep 2

                    // 分批发布
                    for (int i = 0; i < batchSizes.size(); i++) {
                        def batchSize = batchSizes[i]
                        def isLast = (i == batchSizes.size() - 1)

                        echo "========================================"
                        echo "第 ${i+1}/${batchSizes.size()} 批：目标 ${batchSize} 个新 Pod"
                        echo "========================================"

                        // 设置 maxSurge
                        sh """
                            kubectl patch deployment ${RELEASE} -n ${NS} --type=json -p '[{
                                "op": "replace",
                                "path": "/spec/strategy/rollingUpdate/maxSurge",
                                "value": ${batchSize}
                            },{
                                "op": "replace",
                                "path": "/spec/strategy/rollingUpdate/maxUnavailable",
                                "value": 0
                            }]'
                        """

                        // 恢复滚动，让这一批启动
                        sh "kubectl rollout resume deployment/${RELEASE} -n ${NS}"
                        sleep 3

                        // 等待本批就绪
                        def batchReady = false
                        try {
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    // 获取运行中的新版本 Pod
                                    def newPodsList = sh(
                                        script: """
                                            kubectl get pods -n ${NS} -l app=${RELEASE} \
                                            --field-selector=status.phase=Running \
                                            -o json | jq -r '.items[] | select(.spec.containers[0].image | contains("${IMAGE_TAG}")) | .metadata.name'
                                        """,
                                        returnStdout: true
                                    ).trim()

                                    if (!newPodsList) {
                                        echo "等待新 Pod 启动..."
                                        sleep 5
                                        return false
                                    }

                                    def newPods = newPodsList.split('\n').findAll { it }
                                    def readyCount = 0

                                    for (pod in newPods) {
                                        def isReady = sh(
                                            script: "kubectl get pod ${pod} -n ${NS} -o jsonpath='{.status.containerStatuses[0].ready}' 2>/dev/null || echo 'false'",
                                            returnStdout: true
                                        ).trim()
                                        if (isReady == 'true') {
                                            readyCount++
                                        }
                                    }

                                    echo "本批就绪: ${readyCount}/${batchSize}"
                                    
                                    if (readyCount >= batchSize) {
                                        return true
                                    }
                                    
                                    sleep 5
                                    return false
                                }
                            }
                            batchReady = true
                        } catch (err) {
                            echo "⚠️  第 ${i+1} 批 5分钟未完全就绪！"
                            echo "错误信息: ${err.message}"
                        }

                        // 打印新 Pod 列表和状态
                        def newPodsInfo = sh(
                            script: """
                                kubectl get pods -n ${NS} -l app=${RELEASE} -o json | \
                                jq -r '.items[] | select(.spec.containers[0].image | contains("${IMAGE_TAG}")) | 
                                "\\(.metadata.name)\\t\\(.status.phase)\\t\\(.status.containerStatuses[0].ready)"'
                            """,
                            returnStdout: true
                        ).trim()
                        
                        echo "新 Pod 状态：\n${newPodsInfo}"
                        echo "\n查看日志命令：kubectl logs <pod-name> -n ${NS}"

                        // 超时或未就绪 → 强制确认回滚
                        if (!batchReady) {
                            def choice = input(
                                message: "⚠️  第 ${i+1} 批启动失败，是否回滚？",
                                parameters: [choice(name: 'ACT', choices: ['立即回滚', '继续（风险自担）'], description: '选择操作')]
                            )
                            if (choice == '立即回滚') {
                                rollbackDeployment(RELEASE, NS)
                                error("用户选择回滚，部署终止")
                            } else {
                                echo "⚠️  用户选择继续，风险自担"
                            }
                        }

                        // 暂停，等待人工确认下一批
                        sh "kubectl rollout pause deployment/${RELEASE} -n ${NS}"

                        if (!isLast) {
                            def action = input(
                                message: "✅ 第 ${i+1} 批完成，是否继续下一批？",
                                parameters: [choice(name: 'NEXT', choices: ['继续下一批', '停止并回滚'], description: '选择操作')]
                            )
                            if (action == '停止并回滚') {
                                rollbackDeployment(RELEASE, NS)
                                error("用户取消部署，已回滚")
                            }
                        } else {
                            echo "✅ 这是最后一批，准备完成部署"
                        }
                    }

                    // 最后恢复并完成
                    echo "========================================"
                    echo "开始最终部署确认"
                    echo "========================================"
                    
                    sh """
                        kubectl rollout resume deployment/${RELEASE} -n ${NS}
                        kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=10m
                    """
                    
                    // 显示最终状态
                    def finalStatus = sh(
                        script: "kubectl get deployment ${RELEASE} -n ${NS} -o wide",
                        returnStdout: true
                    ).trim()
                    
                    echo "========================================"
                    echo "✅ 所有 Pod 更新完成！"
                    echo "========================================"
                    echo "部署状态：\n${finalStatus}"
                }
            }
        }
    }

    post {
        always {
            echo "构建完成：${env.IMAGE_FULL}"
        }
        success {
            echo "✅ 部署成功！"
        }
        failure {
            echo "❌ 部署失败"
        }
    }
}

// ==================== 回滚函数 ====================
def rollbackDeployment(String release, String ns) {
    echo "========================================"
    echo "开始回滚到上一版本..."
    echo "========================================"
    
    def prevRev = sh(
        script: "helm history ${release} -n ${ns} -o json | jq -r '.[-2].revision // empty'",
        returnStdout: true
    ).trim()

    if (prevRev) {
        sh """
            helm rollback ${release} ${prevRev} -n ${ns}
            kubectl rollout status deployment/${release} -n ${ns} --timeout=5m
        """
        echo "✅ 已回滚到 revision ${prevRev}"
    } else {
        echo "⚠️  无历史版本可回滚"
    }
}