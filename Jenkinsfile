def buildService(serviceName, dockerfilePath, lastCommitId) {
    stage("Build ${serviceName}") {
        script {
            sh "cat ${dockerfilePath}/Dockerfile"
            sh "cp /var/jenkins_home/jacoco/jacocoagent.jar ${dockerfilePath}/"
            sh "docker build -t ${serviceName}-${lastCommitId} -f ${dockerfilePath}/Dockerfile ${dockerfilePath}/"
        }
    }
}

def startService(serviceName, lastCommitId, servicePort, jacocoPort, currentBranch) {
    stage("Start ${serviceName} Container") {
        script {
            sh "docker run -it -d -p ${servicePort}:8080 -p ${jacocoPort}:6300 -v /home/www/xgame/logs/${currentBranch}/${serviceName}:/app/logs/ ${serviceName}-${lastCommitId}"
        }
    }
}


def generateJacocoReport(serviceName, jacocoPort, path) {
    def jacocoDir = "/var/jenkins_home/jacoco/"
    // Change working directory to the one containing jacococli.jar
    dir(jacocoDir) {
        sh "java -jar ./jacococli.jar dump --address 127.0.0.1 --port ${jacocoPort} --destfile ${serviceName}.exec"
        sh "java -jar ./jacococli.jar report ${serviceName}.exec --classfiles ${path}/target/classes --html reports/jacoco-report/${serviceName} --sourcefiles ${path}/src/main/java"
    }
    archiveArtifacts "${path}/target/classes/**/*.class"
    archiveArtifacts "reports/jacoco-report/${serviceName}"
}

def pushDockerImage(serviceName, lastCommitId, harborUser, harborPasswd, harborUrl, harborRepo) {
    sh """
        docker login -u ${harborUser} -p ${harborPasswd} ${harborUrl}
        docker tag ${serviceName}-${lastCommitId} ${harborUrl}/${harborRepo}/${serviceName}-${lastCommitId}
        docker push ${harborUrl}/${harborRepo}/${serviceName}-${lastCommitId}
    """
}

def services = [
        ['name': 'xgame-web', 'servicePort': '8194', 'jacocoPort': '16301', 'path': 'xgame-web'],
        ['name': 'xgame-api', 'servicePort': '8195', 'jacocoPort': '16302', 'path': 'xgame-api/xgame-api-server'],
        ['name': 'xgame-admin', 'servicePort': '8196', 'jacocoPort': '16303', 'path': 'xgame-admin']
]

@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "Gathering SCM changes"
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += "> - ${truncated_msg}   [${entry.author}]\n"
        }
    }
    if (!changeString) {
        changeString = "> - 没有更新记录"
    }
    return changeString
}

pipeline {
    agent any
    tools {
        jdk "jdk8"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '3'))
        disableConcurrentBuilds()
        skipDefaultCheckout()
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }

    environment {
        SONAR_SCANNER_HOME = '/var/jenkins_home/sonar-scanner/bin/sonar-scanner'
        SONAR_LOGIN = '3f3666fbb9872d67b25800d82bbf9cb4172b36a7'
        SONAR_PROJECT_KEY = "${JOB_NAME}"
        SONAR_SOURCES = './'
        SONAR_JAVA_BINARIES = './xgame-api/xgame-api-client/target/,./xgame-api/xgame-api-server/target/,./xgame-admin/target/,./xgame-base/target/,./xgame-dao/target/,./xgame-event/target/,./xgame-gateway/target/,./xgame-rule/target/,./xgame-web/target/'
        lastCommitId = ''
        currentBranch = ''
        harborUser = 'xxx'
        harborPasswd = 'xxxx.'
        harborUrl = '127.0.0.1:8191'
        harborRepo = 'repo'
        sonar = "http://127.0.0.1:8193/projects"
        harbor = "http://127.0.0.1:8191/harbor/projects/2/repositories"
    }

    parameters {
        booleanParam(name: 'generateReport', defaultValue: false, description: 'Generate report? Starting the service needs to be set to false, generating jacoco reports only needs to be set to true')
        booleanParam(name: 'skipSonarQubeScan', defaultValue: false, description: 'Skip static code scan?')
        booleanParam(name: 'skipPushDockerImage', defaultValue: false, description: 'Skip push docker image?')
    }

    stages {
        stage('Checkout') {
            when {
                expression { generateReport != 'true' }
            }
            steps {
                script {
                    def currentBranch = env.BRANCH_NAME
                    // 获取当前分支的代码
                    // checkout scm

                    // 判断是否为 master 分支，只有非 master 分支才执行 checkout
                    if (currentBranch != 'master') {
                        sh 'git checkout master'
                    }
                    sh "git checkout ${currentBranch}"
                    sh 'git branch --list'
                    lastCommitId = sh(script: 'git log -n 1 --pretty=format:"%h"', returnStdout: true).trim()
                }
            }
        }

        stage('Build') {
            when {
                expression { generateReport != 'true' }
            }
            steps {
                script {
                    // 使用Maven编译和打包
                    sh '/var/jenkins_home/maven/bin/mvn  clean'
                    sh '/var/jenkins_home/maven/bin/mvn  install package -Dmaven.test.skip=true'
                }
            }
        }

        stage('Diff') {
            when {
                // 根据条件判断是否执行 Diff 阶段
                allOf {
                    expression { return env.BRANCH_NAME != 'master' }
                    expression { params.generateReport != 'true' }
                }

            }
            steps {
                script {
                    sh 'git branch --list'
                    // 获取当前分支
                    def currentBranch = env.BRANCH_NAME
                    def diffDir = "${WORKSPACE}/code_diff"
                    dir(diffDir) {
                        // 使用git diff生成文件路径，并输出到文件
                        def diffOutput = sh(script: "git diff --name-only ${currentBranch}..master -- |grep '.java'", returnStdout: true).trim()
                        writeFile file: 'code_diff_files.txt', text: diffOutput.replaceAll('\n', ',')
                    }
                    echo "文件路径已保存到 ${WORKSPACE}/code_diff/code_diff_files.txt 文件中"
                }
            }
        }

        stage('SonarQube Scan') {
            when {
                allOf {
                    expression { return skipSonarQubeScan != 'true' }
                    expression { generateReport != 'true' }
                }
            }

            steps {
                script {
                    // 获取当前分支
                    def currentBranch = env.BRANCH_NAME

                    // 定义projectname
                    def projectName

                    // 在master分支执行全量扫描
                    if (currentBranch != 'master') {
                        // 在非master分支执行全量和增量扫描
                        projectName = "${JOB_NAME}"

                        // 读取差异文件
                        def diffFilePath = "${WORKSPACE}/code_diff/code_diff_files.txt"
                        def diffOutput = readFile(file: diffFilePath).trim()

                        // 如果有差异，则执行增量扫描
                        if (diffOutput) {
                            echo "Running incremental SonarQube scan based on diff:"
                            echo diffOutput
                            projectName += "_incremental"
                            // 使用正则表达式确保 projectName 符合规范，并添加一个非数字字符
                            projectName = projectName.replaceAll('[^a-zA-Z0-9-_:.]', '_')
                            // 检查是否包含至少一个非数字字符
                            if (!(projectName =~ /[^\d]/)) {
                                projectName += '_x'  // 添加一个非数字字符
                            }
                            // 执行增量扫描
                            sh "${SONAR_SCANNER_HOME} -Dsonar.sources=${diffOutput} -Dsonar.projectname=${projectName} -Dsonar.login=${SONAR_LOGIN} -Dsonar.projectKey=${projectName} -Dsonar.java.binaries=${SONAR_JAVA_BINARIES}"
                        } else {
                            echo "No diff found. Skipping incremental SonarQube scan."
                        }
                    }

                    projectName = "${JOB_NAME}_full"
                    // 使用正则表达式确保 projectName 符合规范，并添加一个非数字字符
                    projectName = projectName.replaceAll('[^a-zA-Z0-9-_:.]', '_')

                    // 检查是否包含至少一个非数字字符
                    if (!(projectName =~ /[^\d]/)) {
                        projectName += '_x'  // 添加一个非数字字符
                    }

                    // 执行全量扫描
                    sh "${SONAR_SCANNER_HOME} -Dsonar.sources=${SONAR_SOURCES} -Dsonar.projectname=${projectName} -Dsonar.login=${SONAR_LOGIN} -Dsonar.projectKey=${projectName} -Dsonar.java.binaries=${SONAR_JAVA_BINARIES}"
                }
            }
        }

        stage("build Image") {
            when {
                expression { generateReport != 'true' }
            }
            steps {
                script {
                    sh "echo ${lastCommitId}"
                    for (service in services) {
                        buildService(service['name'], service['path'], lastCommitId)
                    }
                }
            }
        }

        stage("Start Containers") {
            when {
                expression { generateReport != 'true' }
            }
            steps {
                script {
                    def currentBranch = env.BRANCH_NAME
                    sh "echo ${lastCommitId}"
                    sh "echo ${currentBranch}"
                    for (service in services) {
                        startService(service['name'], lastCommitId, service['servicePort'], service['jacocoPort'], currentBranch)
                    }
                }
            }
        }

        stage('Test') {
            when {
                expression { generateReport != 'true' }
            }
            steps {
                script {
                    // 在这里可以添加测试步骤
                    echo 'Running tests...'
                }
            }
        }

        stage("Generate jacoco report") {
            when {
                expression { generateReport == 'true' }
            }
            steps {
                script {
                    for (service in services) {
                        generateJacocoReport(service['name'], service['jacocoPort'], service['path'])
                    }
                }
            }
        }

        stage("push Docker Image") {
            when {
                allOf {
                    expression { skipPushDockerImage != 'true' }
                    expression { generateReport != 'true' }
                }
            }
            steps {
                script {
                    services.each { service ->
                        pushDockerImage(service.name, lastCommitId, harborUser, harborPasswd, harborUrl, harborRepo)
                    }
                }
            }
        }
    }

    post {
        success {
            echo "发版成功"
            dingtalk(
                    robot: 'xxxxx',
                    type: 'MARKDOWN',
                    atAll: true,
                    title: 'success:${JOB_NAME}',
                    text: [
                            "### [${JOB_NAME}](${JOB_URL})",
                            "---",
                            "- 任务：[${BUILD_DISPLAY_NAME}](${BUILD_URL})",
                            "- 状态：<font color='red'>构建成功</font>",
                            "- sonarqube: ${sonar}",
                            "- harbor: ${harbor}",
                            "${changeString}"
                    ],
            )
        }
        failure {
            dingtalk(
                    robot: 'xxxxx',
                    type: 'MARKDOWN',
                    atAll: true,
                    title: 'success:${JOB_NAME}',
                    text: [
                            '### [${JOB_NAME}](${JOB_URL})',
                            '---',
                            '- 任务：[${BUILD_DISPLAY_NAME}](${BUILD_URL})',
                            '- 状态：<font color="red">构建失败</font>'
                    ],
            )
        }
    }
}