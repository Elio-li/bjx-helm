pipeline {
    agent any

    // 添加并发控制和超时
    options {
        disableConcurrentBuilds()  // 防止同时部署多个版本
        timeout(time: 30, unit: 'MINUTES')  // 全局超时30分钟
        buildDiscarder(logRotator(numToKeepStr: '10'))  // 保留最近10次构建
    }

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
        string(name: 'POD_POLL_TIMEOUT', defaultValue: '3', description: '新 Pod 检测超时时间(分钟)')
        string(name: 'POD_READY_TIMEOUT', defaultValue: '5', description: 'Pod Ready 等待超时时间(分钟)')
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
        // 部署信息展示
        stage('Deploy Info') {
            steps {
                script {
                    echo """
                    ╔════════════════════════════════════════╗
                    ║         部署信息 Deploy Info          ║
                    ╠════════════════════════════════════════╣
                    ║ 操作类型: ${params.DEPLOY_TYPE}
                    ║ 分支名称: ${params.BRANCH}
                    ║ 构建版本: ${env.BUILD_VERSION}
                    ║ 镜像地址: ${env.IMAGE_FULL}
                    ║ 命名空间: ${env.NAMESPACE}
                    ║ 部署名称: ${params.deployment_name}
                    ║ 构建时间: ${new Date().format('yyyy-MM-dd HH:mm:ss')}
                    ╚════════════════════════════════════════╝
                    """.stripIndent()
                }
            }
        }

        stage('Rollback Check') {
            when { expression { params.DEPLOY_TYPE == 'Rollback' } }
            steps {
                script {
                    echo "🔄 开始回滚检查..."
                    
                    // 优化: 使用更健壮的错误处理
                    def versionsJson = sh(
                        script: """
                            helm history ${params.deployment_name} -n ${env.NAMESPACE} -o json 2>/dev/null || echo '[]'
                        """,
                        returnStdout: true
                    ).trim()

                    if (versionsJson == '[]' || versionsJson.isEmpty()) {
                        error "❌ 没有找到 Helm Release 历史记录，无法回滚！"
                    }

                    def versions = sh(
                        script: """
                            echo '${versionsJson}' \
                            | jq -r '.[] | select(.status=="deployed" or .status=="superseded") | .app_version' \
                            | grep -v null \
                            | grep -v '^$'
                        """,
                        returnStdout: true
                    ).trim().split("\\n").findAll { it }.reverse()

                    if (versions.isEmpty()) {
                        error "❌ 没有可回滚的历史版本！"
                    }

                    echo "✅ 可回滚版本：\n${versions.join('\n')}"

                    def selectedVersion = input(
                        message: "请选择要回滚的 appVersion",
                        parameters: [choice(name: 'APP_VERSION', choices: versions, description: '历史版本')]
                    )

                    echo "🔄 开始回滚到版本: ${selectedVersion}"
                    
                    sh """
                        REVISION=\$(echo '${versionsJson}' | jq -r '.[] | select(.app_version=="${selectedVersion}") | .revision')
                        if [ -z "\$REVISION" ]; then
                            echo "❌ 找不到版本 ${selectedVersion} 对应的 revision"
                            exit 1
                        fi
                        echo "📌 回滚到 revision: \$REVISION"
                        helm rollback ${params.deployment_name} \$REVISION -n ${env.NAMESPACE}
                        kubectl rollout status deployment/${params.deployment_name} -n ${env.NAMESPACE} --timeout=5m
                    """
                    
                    echo "✅ 回滚成功！"
                }
            }
        }

        stage('Checkout') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    echo "📥 开始检出代码..."
                    checkout([$class: 'GitSCM',
                        branches: [[name: "${params.BRANCH.startsWith('origin/') ? params.BRANCH : "*/${params.BRANCH}"}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [],
                        userRemoteConfigs: [[
                            url: "${env.GIT_REPO}",
                            credentialsId: 'GIT_CREDENTIALS'
                        ]]
                    ])
                    echo "✅ 代码检出完成"
                }
            }
        }

        stage('Build Jar') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    echo "🔨 开始构建 JAR 包..."
                    
                    // 优化: 使用 Credentials 管理敏感信息
                    withCredentials([
                        string(credentialsId: 'MYSQL_URL', variable: 'DB_URL'),
                        string(credentialsId: 'MYSQL_USERNAME', variable: 'DB_USER'),
                        string(credentialsId: 'MYSQL_PASSWORD', variable: 'DB_PASS')
                    ]) {
                        sh """
                            rm -rf bjx-helm
                            git clone https://github.com/Elio-li/bjx-helm.git
                            
                            echo "📝 配置数据库连接信息..."
                            sed -i 's|urule.mysql.url=.*|urule.mysql.url=${DB_URL}|' urule-springboot/src/main/resources/ghana/application-dev.properties
                            sed -i 's|urule.mysql.username=.*|urule.mysql.username=${DB_USER}|' urule-springboot/src/main/resources/ghana/application-dev.properties
                            sed -i 's|urule.mysql.password=.*|urule.mysql.password=${DB_PASS}|' urule-springboot/src/main/resources/ghana/application-dev.properties
                            
                            echo "🔨 执行 Maven 构建..."
                            mvn clean install -pl ${params.SERVER_NAME} -am -Dmaven.test.skip=true
                            
                            # 验证 JAR 是否生成
                            if [ ! -f "${env.JAR_PATH}" ]; then
                                echo "❌ JAR 文件未生成: ${env.JAR_PATH}"
                                exit 1
                            fi
                            echo "✅ JAR 构建成功: ${env.JAR_PATH}"
                        """
                    }
                }
            }
        }

        stage('Generate Dockerfile') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    echo "📄 生成 Dockerfile..."
                    def dockerfile = """
                    FROM eclipse-temurin:8-jdk
                    COPY ${env.JAR_PATH} /app/urule.jar
                    WORKDIR /app
                    EXPOSE 8080
                    ENTRYPOINT ["java", "-jar", "/app/urule.jar"]
                    """.stripIndent()
                    writeFile file: 'Dockerfile', text: dockerfile
                    echo "✅ Dockerfile 已生成"
                }
            }
        }

        stage('Build & Push Image') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    echo "🐳 开始构建并推送 Docker 镜像..."
                    withCredentials([usernamePassword(
                        credentialsId: 'Harbor_id',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                            echo "${HARBOR_PASS}" | docker login ${REGISTRY} -u "${HARBOR_USER}" --password-stdin
                            
                            echo "🔨 构建镜像: ${IMAGE_FULL}"
                            docker build -t ${IMAGE_FULL} .
                            
                            echo "📤 推送镜像到 Harbor..."
                            docker push ${IMAGE_FULL}
                            
                            echo "🧹 清理本地镜像..."
                            docker rmi ${IMAGE_FULL} || true
                            
                            docker logout ${REGISTRY}
                        """
                    }
                    echo "✅ 镜像构建和推送完成"
                }
            }
        }

        stage('Archive Jar') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    echo "📦 归档 JAR 文件..."
                    archiveArtifacts artifacts: "${env.JAR_PATH}", fingerprint: true
                    echo "✅ JAR 归档完成"
                }
            }
        }

        stage('Helm Pre-Check') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    echo "🔍 Helm 部署前检查..."
                    
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

                    // 优化: 使用 returnStatus 判断 Deployment 是否存在
                    def exitCode = sh(
                        script: "kubectl get deploy ${RELEASE} -n ${NS} >/dev/null 2>&1",
                        returnStatus: true
                    )
                    env.IS_FIRST_DEPLOY = (exitCode != 0).toString()

                    // 获取副本数
                    if (!env.IS_FIRST_DEPLOY.toBoolean()) {
                        def replicas = sh(
                            script: "kubectl get deploy ${RELEASE} -n ${NS} -o jsonpath='{.spec.replicas}' 2>/dev/null || echo 0",
                            returnStdout: true
                        ).trim()
                        env.REPLICAS = replicas ?: '0'
                        echo "📊 当前副本数: ${env.REPLICAS}"
                    } else {
                        env.REPLICAS = '0'
                        echo "🆕 首次部署，副本数初始化为 0"
                    }
                    
                    echo "✅ Pre-Check 完成 - 首次部署: ${env.IS_FIRST_DEPLOY}, 副本数: ${env.REPLICAS}"
                }
            }
        }

        stage('Helm Upgrade / Install') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def CHART_DIR = env.CHAT_DIR
                    def VALUES_FILE = "${CHART_DIR}/urule-ghana-test.yaml"

                    if (env.IS_FIRST_DEPLOY.toBoolean() || env.REPLICAS.toInteger() <= 0) {
                        echo "🚀 首次部署或副本数为0，执行全量部署..."
                        sh "helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} --namespace ${NS} --wait --timeout=10m"
                        echo "✅ 全量部署完成"
                    } else {
                        echo "🔄 Deployment 存在，触发滚动更新（不等待全部 Ready）..."
                        sh "helm upgrade --install ${RELEASE} ${CHART_DIR} -f ${VALUES_FILE} --namespace ${NS} --timeout=5m"
                        echo "✅ 滚动更新已触发"
                    }
                }
            }
        }

        stage('Canary Pod Wait') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' && env.REPLICAS.toInteger() > 0 } }
            steps {
                script {
                    echo "🔍 等待新 Pod 出现（金丝雀部署第一阶段）..."
                    
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def BUILD_TAG = env.BUILD_VERSION
                    def newPodName = ''
                    def pollStart = System.currentTimeMillis()
                    // 优化: 超时时间可配置
                    def pollTimeoutMs = params.POD_POLL_TIMEOUT.toInteger() * 60 * 1000

                    while ((System.currentTimeMillis() - pollStart) < pollTimeoutMs) {
                        def podList = sh(
                            script: "kubectl get pods -n ${NS} -l app=${RELEASE} -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image --no-headers 2>/dev/null || echo ''",
                            returnStdout: true
                        ).trim()

                        if (podList) {
                            podList.split("\n").each { line ->
                                if (line.trim()) {
                                    def parts = line.trim().split(/\s+/)
                                    if (parts.size() >= 2) {
                                        def name = parts[0]
                                        def image = parts[1]
                                        if (image.contains("${BUILD_TAG}")) {
                                            newPodName = name
                                            echo "✅ 发现新版本 Pod: ${newPodName}"
                                            return
                                        }
                                    }
                                }
                            }
                        }
                        
                        if (newPodName) break
                        
                        def elapsed = (System.currentTimeMillis() - pollStart) / 1000
                        echo "⏳ 等待新 Pod 中... (已等待 ${elapsed.intValue()} 秒)"
                        sleep(time: 5, unit: 'SECONDS')
                    }

                    // 优化: 更好的错误处理
                    if (!newPodName) {
                        echo "⚠️ 在 ${params.POD_POLL_TIMEOUT} 分钟内未检测到新 Pod"
                        def action = input(
                            message: "${params.POD_POLL_TIMEOUT} 分钟内未检测到新 Pod，请选择操作:",
                            parameters: [
                                choice(
                                    name: 'ACTION',
                                    choices: ['继续等待', '回滚'],
                                    description: '继续等待会再等待相同时长，回滚会立即回滚到上一版本'
                                )
                            ]
                        )
                        if (action == '回滚') {
                            rollbackDeployment(RELEASE, NS)
                            error("❌ 已回滚（未检测到新 Pod）")
                        } else {
                            // 继续等待
                            echo "⏳ 继续等待新 Pod..."
                            timeout(time: params.POD_POLL_TIMEOUT.toInteger(), unit: 'MINUTES') {
                                waitUntil {
                                    def podList = sh(
                                        script: "kubectl get pods -n ${NS} -l app=${RELEASE} -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image --no-headers 2>/dev/null || echo ''",
                                        returnStdout: true
                                    ).trim()
                                    
                                    if (podList) {
                                        podList.split("\n").each { line ->
                                            if (line.trim()) {
                                                def parts = line.trim().split(/\s+/)
                                                if (parts.size() >= 2 && parts[1].contains("${BUILD_TAG}")) {
                                                    newPodName = parts[0]
                                                    return true
                                                }
                                            }
                                        }
                                    }
                                    return false
                                }
                            }
                        }
                    }

                    echo "🛑 暂停 Deployment 滚动更新..."
                    sh "kubectl rollout pause deployment/${RELEASE} -n ${NS}"
                    env.CANARY_POD = newPodName
                    echo "✅ 金丝雀 Pod 已锁定: ${newPodName}"
                }
            }
        }

        stage('Canary Pod Ready Check') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' && env.REPLICAS.toInteger() > 0 } }
            steps {
                script {
                    echo "🔍 检查金丝雀 Pod 是否 Ready..."
                    
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    def newPod = env.CANARY_POD
                    def podReady = false

                    try {
                        // 优化: 超时时间可配置
                        timeout(time: params.POD_READY_TIMEOUT.toInteger(), unit: 'MINUTES') {
                            waitUntil {
                                def ready = sh(
                                    script: "kubectl get pod ${newPod} -n ${NS} -o jsonpath='{.status.containerStatuses[0].ready}' 2>/dev/null || echo false",
                                    returnStdout: true
                                ).trim()
                                
                                if (ready == 'true') {
                                    echo "✅ Pod ${newPod} 已 Ready"
                                    return true
                                } else {
                                    // 检查 Pod 状态，提供更多信息
                                    def phase = sh(
                                        script: "kubectl get pod ${newPod} -n ${NS} -o jsonpath='{.status.phase}' 2>/dev/null || echo Unknown",
                                        returnStdout: true
                                    ).trim()
                                    echo "⏳ Pod 状态: ${phase}, 等待 Ready..."
                                    sleep(time: 5, unit: 'SECONDS')
                                    return false
                                }
                            }
                        }
                        podReady = true
                    } catch(e) {
                        podReady = false
                        echo "❌ 新 Pod 未在 ${params.POD_READY_TIMEOUT} 分钟内 Ready"
                        
                        // 输出 Pod 日志帮助排查
                        try {
                            def logs = sh(
                                script: "kubectl logs ${newPod} -n ${NS} --tail=50 2>/dev/null || echo '无法获取日志'",
                                returnStdout: true
                            ).trim()
                            echo "📋 Pod 最近 50 行日志:\n${logs}"
                        } catch(logError) {
                            echo "⚠️ 无法获取 Pod 日志"
                        }
                    }

                    // 优化: 更清晰的回滚逻辑
                    if (!podReady) {
                        def action = input(
                            message: "新 Pod 未在 ${params.POD_READY_TIMEOUT} 分钟内 Ready，请选择操作:",
                            parameters: [
                                choice(
                                    name: 'ACTION',
                                    choices: ['回滚', '人工处理'],
                                    description: '回滚=立即回滚到上一版本，人工处理=保持当前状态退出流水线'
                                )
                            ]
                        )
                        if (action == '回滚') {
                            rollbackDeployment(RELEASE, NS)
                            error("❌ 已回滚到上一版本")
                        } else {
                            sh "kubectl rollout resume deployment/${RELEASE} -n ${NS}"
                            error("⚠️ 已恢复滚动更新，流水线退出，请人工处理")
                        }
                    }
                    
                    echo "✅ 金丝雀 Pod Ready 检查通过"
                }
            }
        }

        stage('Manual Confirmation & Full Update') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' && env.REPLICAS.toInteger() > 0 } }
            steps {
                script {
                    echo "⏸️ 等待人工确认是否继续部署..."
                    
                    def RELEASE = params.deployment_name
                    def NS = env.NAMESPACE
                    
                    def userChoice = input(
                        message: "✅ 第一个 Pod 已 Ready，请确认是否继续更新剩余 Pod？",
                        parameters: [
                            choice(
                                name: 'ACTION',
                                choices: ['继续更新（恢复）', '回滚'],
                                description: '继续更新=更新所有剩余 Pod 到新版本，回滚=回滚到上一版本'
                            )
                        ]
                    )
                    
                    if (userChoice == '回滚') {
                        rollbackDeployment(RELEASE, NS)
                        error("❌ 用户选择回滚，已回滚到上一版本")
                    }

                    echo "▶️ 继续更新剩余 Pod..."
                    sh "kubectl rollout resume deployment/${RELEASE} -n ${NS}"
                    
                    echo "⏳ 等待所有 Pod 更新完成..."
                    sh "kubectl rollout status deployment/${RELEASE} -n ${NS} --timeout=10m"
                    
                    echo "🎉 部署完成：所有 Pod 已更新到 ${env.BUILD_VERSION}"
                }
            }
        }
    }

    post {
        always {
            script {
                echo """
                ╔════════════════════════════════════════╗
                ║            构建结束信息               ║
                ╠════════════════════════════════════════╣
                ║ 构建状态: ${currentBuild.result ?: 'SUCCESS'}
                ║ 构建编号: ${env.BUILD_NUMBER}
                ║ 构建版本: ${env.BUILD_VERSION}
                ║ 镜像地址: ${env.IMAGE_FULL}
                ║ 结束时间: ${new Date().format('yyyy-MM-dd HH:mm:ss')}
                ║ 构建耗时: ${currentBuild.durationString.replace(' and counting', '')}
                ╚════════════════════════════════════════╝
                """.stripIndent()
                
                // 清理 Docker 资源
                sh """
                    echo "🧹 清理 Docker 资源..."
                    docker system prune -f --volumes || true
                """ 
            }
        }
        success {
            script {
                echo "✅ 🎉 部署成功！"
                // TODO: 添加成功通知（钉钉/企业微信/Slack）
                // notifySuccess()
            }
        }
        failure {
            script {
                echo "❌ 部署失败！请检查日志"
                // TODO: 添加失败通知
                // notifyFailure()
            }
        }
        unstable {
            script {
                echo "⚠️ 构建不稳定"
            }
        }
    }
}

// ==================== 回滚函数 ====================
def rollbackDeployment(String release, String ns) {
    echo "🔄 开始回滚到上一版本..."
    
    try {
        // 优化: 更健壮的回滚逻辑
        def historyJson = sh(
            script: "helm history ${release} -n ${ns} -o json 2>/dev/null || echo '[]'",
            returnStdout: true
        ).trim()
        
        if (historyJson == '[]' || historyJson.isEmpty()) {
            echo "❌ 无法获取 Helm 历史记录"
            return
        }
        
        // 获取所有已部署或已替换的版本（排除当前版本）
        def prevRev = sh(
            script: """
                echo '${historyJson}' \
                | jq -r '[.[] | select(.status=="deployed" or .status=="superseded")] | sort_by(.revision) | .[-2].revision // empty'
            """,
            returnStdout: true
        ).trim()

        if (prevRev && prevRev != 'null' && prevRev != '') {
            echo "📌 回滚到 revision: ${prevRev}"
            sh """
                helm rollback ${release} ${prevRev} -n ${ns} --wait --timeout=5m
                kubectl rollout status deployment/${release} -n ${ns} --timeout=5m
                kubectl rollout resume deployment/${release} -n ${ns} || true
            """
            echo "✅ 已成功回滚到 revision ${prevRev}"
        } else {
            echo "⚠️ 无历史版本可回滚（可能是首次部署）"
            // 如果没有历史版本，尝试缩容到 0
            echo "🔄 尝试将 Deployment 缩容到 0..."
            sh """
                kubectl scale deployment/${release} -n ${ns} --replicas=0 || true
                kubectl rollout resume deployment/${release} -n ${ns} || true
            """
        }
    } catch (Exception e) {
        echo "❌ 回滚过程出现异常: ${e.message}"
        // 确保恢复 rollout
        sh "kubectl rollout resume deployment/${release} -n ${ns} || true"
        throw e
    }
}