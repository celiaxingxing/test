pipeline {
    agent any
    
    environment {
        // 设置 Git 相关环境变量
        GIT_SSL_VERSION = 'tlsv1.2'
        
        // 你的测试文件在collections目录
        TEST_FILES_DIR = "${WORKSPACE}/collections"
        REPORT_DIR = "${WORKSPACE}/reports"
    }
    
    stages {
        stage('配置 Git 环境') {
            steps {
                echo "🔧 配置 Git 使用 HTTP/1.1 协议..."
                
                sh '''
                    echo "=== Git 环境配置 ==="
                    echo "1. 当前 Git 版本:"
                    git --version || echo "Git 未安装，跳过配置"
                    
                    echo ""
                    echo "2. 设置 Git 使用 HTTP/1.1 协议:"
                    git config --global http.version HTTP/1.1 || echo "Git 配置失败"
                    
                    echo ""
                    echo "3. 其他 Git 优化配置:"
                    git config --global http.postBuffer 524288000 || true
                    git config --global core.compression 0 || true
                    
                    echo ""
                    echo "4. 验证 Git 配置:"
                    git config --global --get http.version || echo "无法读取配置"
                    
                    echo "✅ Git 环境配置完成"
                '''
            }
        }
        
        stage('检出代码') {
            steps {
                echo "📥 检出仓库代码..."
                
                script {
                    try {
                        // 方法1：使用 checkout 命令
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/api-test']],  // 修改为您的分支名
                            extensions: [[
                                $class: 'GitConfig',
                                configKey: 'http.version',
                                configValue: 'HTTP/1.1'
                            ]],
                            userRemoteConfigs: [[
                                url: 'https://github.com/celiaxingxing/test.git',
                                credentialsId: 'github-token-celiaxingxing'  // 替换为您的凭据ID
                            ]]
                        ])
                        
                        echo "✅ 代码检出成功"
                        
                    } catch (Exception e) {
                        echo "❌ 检出失败，尝试备用方法..."
                        
                        // 方法2：手动执行 Git 命令
                        sh '''
                            # 清理现有目录
                            rm -rf ${WORKSPACE}/* || true
                            
                            # 使用完整 Git 命令
                            git init
                            git config http.version HTTP/1.1
                            git remote add origin https://github.com/celiaxingxing/test.git
                            
                            # 设置认证信息（如果使用令牌）
                            # git config --local http.extraheader "Authorization: token YOUR_TOKEN"
                            
                            git fetch --depth=1 origin main
                            git checkout -b main origin/main
                            
                            echo "备用方法检出完成"
                        '''
                    }
                }
            }
        }
        
        stage('验证环境') {
            steps {
                echo "🔍 验证APIFox CLI和测试文件..."
                
                sh '''
                    echo "=== 环境检查 ==="
                    echo "工作目录内容:"
                    ls -la
                    
                    echo ""
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
                echo "=== 构建详情 ==="
                echo "工作空间: $(pwd)"
                echo "Git 配置验证:"
                git config --global --get http.version || echo "无法读取 Git 配置"
                echo "目录结构:"
                ls -la
                echo "报告目录:"
                ls -la reports/ || echo "无报告目录"
            '''
            
            // 清理 Git 配置（可选）
            sh '''
                echo "清理临时配置..."
                git config --global --unset http.version || true
            '''
        }
        success {
            echo "🎉 所有测试执行成功！"
            // 可以添加通知，如邮件、Slack等
        }
        failure {
            echo "⚠️ 测试执行过程中出现问题"
            
            // 失败时提供诊断信息
            sh '''
                echo "=== 故障诊断 ==="
                echo "网络连接测试:"
                curl -I https://github.com || echo "网络连接失败"
                echo ""
                echo "Git 远程仓库测试:"
                git ls-remote --get-url origin || echo "无法获取远程仓库"
            '''
        }
    }
}
