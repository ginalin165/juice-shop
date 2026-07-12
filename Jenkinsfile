pipeline {
    agent any
    stages {
        stage('1. Kéo mã nguồn (Checkout)') {
            steps {
                git url: 'https://github.com/ginalin165/juice-shop.git', branch: 'master'
            }
        }

        stage('2. Quét thư viện bên thứ 3 (SCA)') {
            steps {
                echo 'Bắt đầu kiểm tra lỗ hổng CVE trong các thư viện Node.js...'
                // Dùng Docker gọi Node.js ra để chạy npm audit độc lập, tránh lỗi môi trường Jenkins
                sh 'docker run --rm -v ${PWD}:/app -w /app node:18 npm audit --audit-level=high || true'
            }
        }
        
        stage('Unit Test & Code Coverage') {
            steps {
                echo 'Bắt đầu chạy Unit Test để lấy dữ liệu Code Coverage...'
                sh 'docker run --rm -v ${PWD}:/app -w /app -e CHROME_BIN=/usr/bin/google-chrome -e PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true timbru31/node-chrome:18 sh -c "npm install && npm test || true"'
            }
        }
        
        stage('3. Quét tĩnh mã nguồn (SAST - SonarQube)') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=owasp-juice-shop \
                            -Dsonar.projectName='OWASP Juice Shop' \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/node_modules/**,**/dist/** \
                            -Dsonar.javascript.lcov.reportPaths=build/reports/coverage/server/lcov.info,build/reports/coverage/frontend/lcov.info"
                    }
                }
            }
        }
        
        stage('4. Triển khai ứng dụng (Deploy)') {
            steps {
                echo 'Khởi chạy ứng dụng lên môi trường kiểm thử (Cổng 3000)...'
                sh 'docker restart juice-shop || true'
            }
        }

        stage('5. Quét động chủ động (DAST - ZAP Full Scan)') {
            steps {
                echo 'Thực hiện tấn công chủ động (Active Scan) vào Juice Shop...'
                sh 'docker rm -f zap-scanner || true'
                // Thay zap-baseline.py thành zap-full-scan.py
                sh 'docker run -u root --name zap-scanner -v zap_temp:/zap/wrk -t zaproxy/zap-stable zap-full-scan.py -t http://host.docker.internal:3000 -r zap_report.html || true'
                sh 'docker cp zap-scanner:/zap/wrk/zap_report.html zap_report.html || true'
                sh 'docker rm -f zap-scanner || true'
                sh 'docker volume rm zap_temp || true'
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
        }
    }
}
