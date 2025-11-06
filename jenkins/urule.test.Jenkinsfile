pipeline {
    agent any

    parameters {
        string(name: 'GIT_REPO', defaultValue: 'git@github.com:bjx-code-backend/tanzania_loan.git', description: 'Git 仓库地址')
        string(name: 'BRANCH', defaultValue: 'dev_gh', description: 'Git 分支')
        string(name: 'SERVER_NAME', defaultValue: 'urule-springboot', description: '模块名称')
        string(name: 'service', defaultValue: 'urule', description: '服务名')
        string(name: 'deployment_name', defaultValue: 'urule-ghana', description: 'Deployment 名称')
        choice(name: 'DEPLOY_TYPE', choices: ['Deploy', 'Rollback'], description: '操作类型：Deploy=部署新版本，Rollback=回滚')
        choice(name: 'BATCH_MODE', choices: ['单批发布', '多批发布'], description: '发布模式')
        choice(name: 'BATCH_COUNT', choices: ['2', '3', '4', '5'], description: '多批发布时的批次数（单批发布时忽略）')
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

        // ==================== 分批发布 ====================
        stage('Batch Deploy') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def CHART_DIR = env.CHART_DIR
                    def VALUES_FILE = "${CHART_DIR}/urule-ghana-test.yaml"
                    def IMAGE_TAG = env.BUILD_VERSION
                    def BATCH_MODE = params.BATCH_MODE
                    def BATCH_COUNT = params.BATCH_COUNT.toInteger()

                    echo "========================================"
                    echo "发布模式: ${BATCH_MODE}"
                    echo "Release: ${RELEASE}"
                    echo "Namespace: ${NS}"
                    echo "Image Tag: ${IMAGE_TAG}"
                    echo "========================================"

                    // 1. 获取当前副本数和旧镜像
                    def originalReplicas = sh(
                        script: "kubectl get deployment ${RELEASE} -n ${NS} -o jsonpath='{.spec.replicas}' 2>/dev/null || echo '0'",
                        returnStdout: true
                    ).trim().toInteger()

                    if (originalReplicas <= 0) {
                        error "副本数为 0 或 Deployment 不存在"
                    }

                    def oldImage = sh(
                        script: "kubectl get deployment ${RELEASE} -n ${NS} -o jsonpath='{.spec.template.spec.containers[0].image}'",
                        returnStdout: true
                    ).trim()

                    echo "当前副本数: ${originalReplicas}"
                    echo "旧镜像: ${oldImage}"

                    // 2. 更新 Helm values 和 Chart.yaml
                    sh """
                        sed -i "s|^  tag:.*|  tag: ${IMAGE_TAG}|" ${VALUES_FILE}
                        sed -i "s|^appVersion:.*|appVersion: \\"${IMAGE_TAG}\\"|" ${CHART_DIR}/Chart.yaml
                    """

                    // 3. 计算分批策略
                    def batches = []
                    if (BATCH_MODE == '单批发布') {
                        batches = [originalReplicas]
                    } else {
                        // 多批发布：均匀分配
                        def avgSize = (originalReplicas / BATCH_COUNT).intValue()
                        def remainder = originalReplicas % BATCH_COUNT
                        
                        for (int i = 0; i < BATCH_COUNT; i++) {
                            def size = avgSize + (i < remainder ? 1 : 0)
                            batches.add(size)
                        }
                    }

                    echo "分批计划: ${batches} (共 ${batches.size()} 批)"

                    // 4. 执行分批发布
                    def cumulativePods = 0
                    
                    for (int i = 0; i < batches.size(); i++) {
                        def batchSize = batches[i]
                        cumulativePods += batchSize
                        def isLast = (i == batches.size() - 1)

                        echo "========================================"
                        echo "第 ${i+1}/${batches.size()} 批"
                        echo "本批新增: ${batchSize} 个 Pod"
                        echo "累计新版: ${cumulativePods}/${originalReplicas}"
                        echo "========================================"

                        // 4.1 先缩容到累计数（删除旧 Pod）
                        sh """
                            kubectl scale deployment ${RELEASE} -n ${NS} --replicas=${cumulativePods}
                            sleep 3
                        """

                        // 4.2 更新镜像（通过 Helm）
                        sh """
                            helm upgrade ${RELEASE} ${CHART_DIR} \
                                -f ${VALUES_FILE} \
                                --namespace ${NS} \
                                --set replicaCount=${cumulativePods}
                        """

                        // 4.3 等待新 Pod 就绪
                        echo "等待新 Pod 启动并就绪..."
                        def batchReady = false
                        try {
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    // 检查新版本 Pod 数量
                                    def newPodCount = sh(
                                        script: """
                                            kubectl get pods -n ${NS} -l app=${RELEASE} \
                                                -o json | jq '[.items[] | select(.spec.containers[0].image | contains("${IMAGE_TAG}")) | select(.status.phase=="Running") | select(.status.containerStatuses[0].ready==true)] | length'
                                        """,
                                        returnStdout: true
                                    ).trim().toInteger()

                                    echo "新版本就绪 Pod: ${newPodCount}/${cumulativePods}"

                                    if (newPodCount >= cumulativePods) {
                                        return true
                                    }
                                    
                                    sleep 5
                                    return false
                                }
                            }
                            batchReady = true
                        } catch (err) {
                            echo "⚠️ 第 ${i+1} 批超时未完全就绪"
                            echo "错误: ${err.message}"
                        }

                        // 4.4 显示 Pod 状态
                        def podStatus = sh(
                            script: """
                                kubectl get pods -n ${NS} -l app=${RELEASE} \
                                    -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image,STATUS:.status.phase,READY:.status.containerStatuses[0].ready
                            """,
                            returnStdout: true
                        ).trim()
                        echo "当前 Pod 状态:\n${podStatus}"

                        // 4.5 失败处理
                        if (!batchReady) {
                            def choice = input(
                                message: "⚠️ 第 ${i+1} 批未完全就绪，是否继续？",
                                parameters: [choice(name: 'ACT', choices: ['继续（风险自担）', '立即回滚'], description: '选择操作')]
                            )
                            if (choice == '立即回滚') {
                                echo "开始回滚..."
                                sh """
                                    helm upgrade ${RELEASE} ${CHART_DIR} \
                                        -f ${VALUES_FILE} \
                                        --namespace ${NS} \
                                        --set image.tag=${oldImage.split(':')[1]} \
                                        --set replicaCount=${originalReplicas}
                                    kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=5m
                                """
                                error("用户选择回滚，部署终止")
                            }
                        }

                        // 4.6 人工确认继续
                        if (!isLast) {
                            echo "✅ 第 ${i+1} 批完成"
                            def action = input(
                                message: "是否继续第 ${i+2} 批？",
                                parameters: [choice(name: 'NEXT', choices: ['继续下一批', '停止并回滚'], description: '选择操作')]
                            )
                            if (action == '停止并回滚') {
                                echo "开始回滚..."
                                sh """
                                    helm upgrade ${RELEASE} ${CHART_DIR} \
                                        -f ${VALUES_FILE} \
                                        --namespace ${NS} \
                                        --set image.tag=${oldImage.split(':')[1]} \
                                        --set replicaCount=${originalReplicas}
                                    kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=5m
                                """
                                error("用户取消部署，已回滚")
                            }
                        }
                    }

                    // 5. 最后恢复到原副本数（如果有偏差）
                    echo "========================================"
                    echo "✅ 所有批次完成，恢复副本数"
                    echo "========================================"
                    sh """
                        helm upgrade ${RELEASE} ${CHART_DIR} \
                            -f ${VALUES_FILE} \
                            --namespace ${NS} \
                            --set replicaCount=${originalReplicas}
                        kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=5m
                    """

                    // 6. 显示最终状态
                    def finalStatus = sh(
                        script: "kubectl get deployment ${RELEASE} -n ${NS} -o wide",
                        returnStdout: true
                    ).trim()
                    echo "最终状态:\n${finalStatus}"

                    def allPods = sh(
                        script: "kubectl get pods -n ${NS} -l app=${RELEASE}",
                        returnStdout: true
                    ).trim()
                    echo "所有 Pod:\n${allPods}"
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