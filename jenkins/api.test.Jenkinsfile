pipeline {
    agent any

    parameters {
        string(name: 'GIT_REPO', defaultValue: 'git@github.com:bjx-code-backend/tanzania_loan.git', description: 'Git仓库地址')
        string(name: 'BRANCH', defaultValue: 'dev_gh', description: 'Git 分支')
        string(name: 'SERVER_NAME', defaultValue: 'loan-api', description: '模块名称')
        string(name: 'service', defaultValue: 'loan-api', description:'服务名' )
        string(name: 'deployment_name', defaultValue: 'api-ghana', description:'deployname' )
        choice(name: 'DEPLOY_TYPE', choices: ['Deploy', 'Rollback'], description: '请选择操作类型：Deploy=部署新版本，Rollback=回滚历史版本')
        }

    environment {
        REGISTRY = 'harbor.bjxsre.com'
        PROJECT  = 'bjx-ghana-test'
        BUILD_VERSION = "${params.BRANCH}-${env.BUILD_NUMBER}-${new Date().format('yyyyMMddHHmmss')}"
        IMAGE_FULL = "${REGISTRY}/${PROJECT}/${params.SERVER_NAME}:${BUILD_VERSION}"
        CHAT_DIR = "./bjx-helm/charts/api"
        jar_path = "www/${params.SERVER_NAME}/target/${params.SERVER_NAME}.jar"
    }


    stages {
        stage('Rollback Check') {
             when { expression { params.DEPLOY_TYPE == 'Rollback' } }
            steps {
                script {
                    // 获取 helm 历史版本 appVersion 列表
                    def versions = sh(script: "helm history ${params.deployment_name} -o json | jq -r '.[].app_version'", returnStdout: true)
                                    .trim()
                                    .split("\n")
                                    .reverse() // 最新版本排在前面
                    echo "可回滚版本列表：\n${versions.join('\n')}"
                    
                    // 让用户在 Jenkins UI 输入选择
                    def selectedVersion = input(
                        message: "请选择要回滚的 appVersion",
                        parameters: [
                            choice(name: 'APP_VERSION', choices: versions.join("\n"), description: '历史版本')
                        ]
                    )
                    
                    // 查 revision 并回滚
                    sh """
                        REVISION=\$(helm history ${params.deployment_name} -o json | jq -r '.[] | select(.app_version=="${selectedVersion}") | .revision')
                        if [ -z "\$REVISION" ]; then
                            echo "❌ 找不到指定版本 ${selectedVersion}"
                            exit 1
                        fi
                        helm rollback ${params.deployment_name} \$REVISION      
                    """
                }
            }
        }

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
                    sed -i 's#127.0.0.1:2012#172.20.50.13:2012#;s#47.236.186.192:8848#172.20.75.117:8848#' www/loan-api/src/main/resources/ghana/bootstrap-dev.properties
                    mvn clean install -pl www/${params.SERVER_NAME} -am -Dmaven.test.skip=true
                """
            }
        }

        stage('Generate Dockerfile') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                script {
                    def dockerfileContent = """
                    FROM eclipse-temurin:8-jdk
                    COPY ${env.jar_path}  /app/${params.SERVER_NAME}.jar
                    WORKDIR /app
                    ENTRYPOINT ["/app/${params.SERVER_NAME}.jar"]
                    """.stripIndent()
                    writeFile file: 'Dockerfile', text: dockerfileContent
                    echo "✅ Dockerfile 已动态生成"
                }
            }
        }

        stage('Build & Push Docker Image') {
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
                archiveArtifacts artifacts: "${env.jar_path}", fingerprint: true
            }
        }
        stage('Helm Deploy') {
            when { expression { params.DEPLOY_TYPE == 'Deploy' } }
            steps {
                sh """
                sed -i "s|^  tag:.*|  tag: ${env.BUILD_VERSION}|" ${env.CHAT_DIR}/api-ghana-test.yaml
                sed -i "s|^appVersion:.*|appVersion: \"${env.BUILD_VERSION}\"|" ${env.CHAT_DIR}/Chart.yaml
                git clone https://github.com/Elio-li/bjx-helm.git    
                helm upgrade --install ${params.deployment_name}  ${env.CHAT_DIR} -f ${env.CHAT_DIR}/api-ghana-test.yaml --namespace ghana
                
                """
                waitForPodsRunning('ghana', "app=${params.deployment_name}", 600, 10)
            }
        }


    }

    post {
        always {
            echo "✅ 构建完成：分支 ${params.BRANCH}"
            echo "🐳 镜像名称：${env.IMAGE_FULL}"
        }
        success {
            echo "🎉 打包成功"
        }
        failure {
            echo "❌ 打包失败"
        }
    }
}

def waitForPodsRunning(namespace, labelSelector, timeoutSeconds = 600, intervalSeconds = 10) {
    def startTime = System.currentTimeMillis()
    while (true) {
        // 获取 Pod 状态
        def podStatus = sh(
        script: "kubectl get pods -n ${namespace} -l ${labelSelector} -o jsonpath='{range .items[*]}{.metadata.name}:{.status.phase}{\"\\n\"}{end}'",
        returnStdout: true
        ).trim()
        
        if (!podStatus) {
            echo "❌ 没有找到匹配的 Pod，检查标签或命名空间是否正确"
            error("No pods found for label ${labelSelector} in namespace ${namespace}")
        }

        // 检查是否全部 Running
        def allRunning = true
        podStatus.split(' ').each { pod ->
            def parts = pod.split(':')
            def name = parts[0]
            def status = parts[1]
            if (status != 'Running') {
                echo "⏳ Pod ${name} 当前状态：${status}"
                allRunning = false
            }
        }

        if (allRunning) {
            echo "✅ 所有 Pod 已全部 Running"
            break
        }

        // 超时检查
        def elapsed = (System.currentTimeMillis() - startTime) / 1000
        if (elapsed > timeoutSeconds) {
            error("⏰ 等待 Pod 超时，超过 ${timeoutSeconds} 秒")
        }

        sleep(intervalSeconds)
    }
}
