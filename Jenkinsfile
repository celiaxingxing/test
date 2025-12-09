pipeline {
    agent any
    
    environment {
        // 你的测试文件在collections目录
        TEST_FILES_DIR = "${WORKSPACE}/collections"
        REPORT_DIR = "${WORKSPACE}/reports"
    }
    
    stages {
        stage('验证环境') {
            steps {
                echo "🔍 验证APIFox CLI和测试文件..."
                
                sh '''
                    echo "=== 环境检查 ==="
                    echo "1. APIFox CLI版本:"
                    apifox --version || { echo "❌ APIFox CLI未安装"; exit 1; }
                    
                    echo ""
                    echo "2. 测试文件目录:"
                    ls -la ${TEST_FILES_DIR}/ || { echo "❌ collections目录不存在"; exit 1; }
                    
                    echo ""
                    echo "3. 找到的JSON测试文件:"
                    find ${TEST_FILES_DIR} -name "*.json" | head -10
                '''
            }
        }
        
        stage('顺序执行全部测试') {
            steps {
                script {
                    echo "🚀 开始顺序执行所有测试文件..."
                    
                    // 查找所有JSON文件
                    def testFiles = findFiles(glob: 'collections/*.json')
                    
                    if (testFiles.isEmpty()) {
                        error "❌ 在collections目录未找到任何.json测试文件"
                    }
                    
                    echo "找到 ${testFiles.size()} 个测试文件"
                    
                    // 创建报告目录
                    sh 'mkdir -p ${REPORT_DIR}'
                    
                    // 顺序执行每个文件
                    def results = [:]
                    testFiles.eachWithIndex { file, index ->
                        def testNum = index + 1
                        def testName = file.name.replace('.json', '')
                        
                        // 为每个测试创建独立的报告子目录
                        def testReportDir = "${REPORT_DIR}/${testName}"
                        
                        results["测试${testNum}"] = {
                            stage("${testNum}. ${file.name}") {
                                echo "▶️ 执行: ${file.name}"
                                
                                sh """
                                    echo "--- 开始执行 ${file.name} ---"
                                    mkdir -p ${testReportDir}
                                    
                                    # 关键命令：使用官方文档格式运行本地JSON文件
                                    apifox run "${TEST_FILES_DIR}/${file.name}" \
                                        -r html,json \
                                        --report-dir "${testReportDir}" \
                                        --timeout 120000
                                    
                                    echo "✅ ${file.name} 执行完成"
                                    echo "报告保存至: ${testReportDir}"
                                """
                            }
                        }
                    }
                    
                    // 并行执行（但实际是顺序，结构清晰）
                    parallel results
                }
            }
        }
        
        stage('生成汇总报告') {
            steps {
                echo "📊 生成测试汇总..."
                
                sh '''
                    echo "=== 执行结果汇总 ==="
                    echo "报告目录: ${REPORT_DIR}"
                    echo ""
                    echo "各测试报告位置:"
                    find ${REPORT_DIR} -name "*.html" -o -name "*.json" | sort
                    
                    # 统计HTML报告数量
                    HTML_COUNT=$(find ${REPORT_DIR} -name "*.html" | wc -l)
                    echo ""
                    echo "生成的HTML报告: ${HTML_COUNT} 个"
                '''
                
                // 可选：归档报告文件
                archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
            }
        }
    }
    
    post {
        always {
            echo "🏁 构建流程执行完毕"
            sh '''
                echo "工作空间: $(pwd)"
                echo "总耗时: 构建已完成"
            '''
        }
        success {
            echo "🎉 所有测试执行成功！"
            // 可以添加通知，如邮件、Slack等
        }
        failure {
            echo "⚠️ 测试执行过程中出现问题"
        }
    }
}
