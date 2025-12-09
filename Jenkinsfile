pipeline {
    agent any
    
    environment {
        TEST_DIR = "${WORKSPACE}/collections"
        REPORT_DIR = "${WORKSPACE}/reports"
        HTML_REPORT_DIR = "${WORKSPACE}/html-reports"
    }
    
    stages {
        stage('准备环境') {
            steps {
                sh '''
                    echo "=== 环境准备 ==="
                    echo "工作目录: $(pwd)"
                    echo "测试文件目录: ${TEST_DIR}"
                    echo ""
                    echo "📂 collections目录内容:"
                    ls -la ${TEST_DIR}/ 2>/dev/null || echo "目录为空"
                '''
            }
        }
        
        stage('顺序执行API测试') {
            steps {
                script {
                    echo "🚀 开始顺序执行所有测试..."
                    
                    // 查找所有测试文件
                    def testFiles = findFiles(glob: 'collections/*.json')
                    
                    if (testFiles.isEmpty()) {
                        error "❌ 未找到测试文件！请在collections目录添加.json文件"
                    }
                    
                    echo "找到 ${testFiles.size()} 个测试文件："
                    testFiles.eachWithIndex { file, index ->
                        echo "  ${index + 1}. ${file.name}"
                    }
                    
                    // 创建报告目录
                    sh '''
                        mkdir -p ${REPORT_DIR}
                        mkdir -p ${HTML_REPORT_DIR}
                    '''
                    
                    // 顺序执行每个测试
                    def failedTests = []
                    def successCount = 0
                    
                    for (int i = 0; i < testFiles.size(); i++) {
                        def file = testFiles[i]
                        def testNum = i + 1
                        def testName = file.name.replace('.json', '')
                        
                        stage("测试 ${testNum}/${testFiles.size()}: ${file.name}") {
                            echo "▶️ 执行: ${file.name}"
                            
                            def startTime = System.currentTimeMillis()
                            
                            try {
                                // 执行APIFox测试命令
                                sh """
                                    echo "开始执行: ${file.name}"
                                    
                                    # 这里是APIFox命令，请根据实际情况修改
                                    # 示例命令 - 请替换成你的真实命令
                                    echo "模拟执行: apifox run --data-file collections/${file.name}"
                                    
                                    # 真实命令应该类似：
                                    # apifox run \\
                                    #   --data-file "collections/${file.name}" \\
                                    #   --env-name "testing" \\
                                    #   --report-output "${REPORT_DIR}/${testName}" \\
                                    #   --report-type json,html
                                    
                                    # 等待模拟执行
                                    sleep 2
                                    
                                    # 模拟成功结果
                                    echo "✅ ${file.name} 执行成功"
                                    echo '{"test": "${file.name}", "status": "passed", "timestamp": "'$(date +%s)'"}' > "${REPORT_DIR}/${testName}.json"
                                """
                                
                                successCount++
                                
                            } catch (Exception e) {
                                echo "❌ ${file.name} 执行失败: ${e.getMessage()}"
                                failedTests.add(file.name)
                                
                                // 记录失败信息
                                sh """
                                    echo '{"test": "${file.name}", "status": "failed", "error": "执行错误"}' > "${REPORT_DIR}/${testName}.json"
                                """
                            }
                            
                            // 测试间等待（可选，避免服务器压力）
                            if (i < testFiles.size() - 1) {
                                sleep time: 1, unit: 'SECONDS'
                            }
                        }
                    }
                    
                    // 保存执行结果
                    env.SUCCESS_COUNT = successCount
                    env.FAILURE_COUNT = failedTests.size()
                    env.TOTAL_TESTS = testFiles.size()
                    
                    if (failedTests.size() > 0) {
                        echo "⚠️ 失败的测试: ${failedTests}"
                    }
                }
            }
        }
        
        stage('生成测试报告') {
            steps {
                script {
                    echo "📊 生成测试报告..."
                    
                    sh '''
                        echo "=== 测试执行汇总 ==="
                        echo "总测试数: ${TOTAL_TESTS}"
                        echo "成功: ${SUCCESS_COUNT}"
                        echo "失败: ${FAILURE_COUNT}"
                        echo "成功率: $(( ${SUCCESS_COUNT} * 100 / ${TOTAL_TESTS} ))%"
                        
                        echo ""
                        echo "📁 报告文件列表:"
                        ls -la ${REPORT_DIR}/ 2>/dev/null || echo "暂无报告"
                    '''
                    
                    // 生成HTML报告（简单版）
                    def htmlReport = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>API测试报告</title>
                        <style>
                            body { font-family: Arial, sans-serif; margin: 20px; }
                            .summary { background: #f5f5f5; padding: 20px; border-radius: 5px; }
                            .success { color: green; }
                            .failure { color: red; }
                            table { border-collapse: collapse; width: 100%; margin-top: 20px; }
                            th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                            th { background-color: #f2f2f2; }
                        </style>
                    </head>
                    <body>
                        <h1>API自动化测试报告</h1>
                        <div class="summary">
                            <h2>执行摘要</h2>
                            <p><strong>执行时间:</strong> ${new Date().format('yyyy-MM-dd HH:mm:ss')}</p>
                            <p><strong>总测试数:</strong> ${env.TOTAL_TESTS}</p>
                            <p class="success"><strong>成功:</strong> ${env.SUCCESS_COUNT}</p>
                            <p class="failure"><strong>失败:</strong> ${env.FAILURE_COUNT}</p>
                        </div>
                        
                        <h2>详细结果</h2>
                        <table>
                            <tr>
                                <th>序号</th>
                                <th>测试文件</th>
                                <th>状态</th>
                                <th>报告文件</th>
                            </tr>
                    """
                    
                    // 获取测试文件列表
                    def testFiles = findFiles(glob: 'collections/*.json')
                    testFiles.eachWithIndex { file, index ->
                        def testName = file.name.replace('.json', '')
                        htmlReport += """
                            <tr>
                                <td>${index + 1}</td>
                                <td>${file.name}</td>
                                <td>${env.FAILURE_COUNT.toInteger() > 0 ? '❌ 失败' : '✅ 成功'}</td>
                                <td><a href="${REPORT_DIR}/${testName}.json">查看JSON报告</a></td>
                            </tr>
                        """
                    }
                    
                    htmlReport += """
                        </table>
                        <p><i>报告生成时间: ${new Date()}</i></p>
                    </body>
                    </html>
                    """
                    
                    // 写入HTML文件
                    writeFile file: "${HTML_REPORT_DIR}/index.html", text: htmlReport
                    
                    echo "✅ HTML报告已生成: ${HTML_REPORT_DIR}/index.html"
                }
            }
        }
        
        stage('归档报告') {
            steps {
                echo "📦 归档测试报告..."
                
                sh '''
                    echo "归档目录结构:"
                    find ${REPORT_DIR} -type f 2>/dev/null | head -10
                    
                    echo ""
                    echo "HTML报告位置:"
                    ls -la ${HTML_REPORT_DIR}/ 2>/dev/null || echo "无HTML报告"
                '''
                
                // 归档所有报告文件
                archiveArtifacts artifacts: "${REPORT_DIR}/**/*, ${HTML_REPORT_DIR}/**/*", allowEmptyArchive: true
            }
        }
    }
    
    post {
        always {
            echo "📋 构建完成摘要"
            sh '''
                echo "=========================================="
                echo "📊 最终统计"
                echo "总测试数: ${TOTAL_TESTS}"
                echo "成功: ${SUCCESS_COUNT}"
                echo "失败: ${FAILURE_COUNT}"
                echo "=========================================="
            '''
            
            // 更新构建描述
            script {
                currentBuild.description = "✅ ${env.SUCCESS_COUNT}成功 ❌ ${env.FAILURE_COUNT}失败"
            }
        }
        
        success {
            echo "🎉 所有测试执行完成！"
        }
        
        failure {
            echo "⚠️ 有测试执行失败，请检查"
        }
    }
}
