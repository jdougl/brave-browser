pipeline {
    agent none
    options {
        // 15m quiet period as described at https://jenkins.io/blog/2010/08/11/quiet-period-feature/
        // quietPeriod(900)
        disableConcurrentBuilds()
        timeout(time: 4, unit: "HOURS")
        timestamps()
    }
    parameters {
        choice (name: "CHANNEL", choices: ["dev", "beta", "release"])
    }
    environment {
        CHANNEL_CAPITALIZED = "${CHANNEL}".capitalize()
        BUILD_TYPE = "Release"
        OUT_DIR = "src/out/${BUILD_TYPE}"
        LINT_BRANCH = "TEMP_LINT_BRANCH_${BUILD_NUMBER}"
        REFERRAL_API_KEY = credentials("REFERRAL_API_KEY")
        BRAVE_GOOGLE_API_KEY = credentials("npm_config_brave_google_api_key")
        BRAVE_ARTIFACTS_BUCKET = credentials("brave-jenkins-artifacts-s3-bucket")
    }
    stages {
        stage ("all") {
            parallel {
                stage ("linux") {
                    agent { label "linux-ci" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                        SCCACHE_BUCKET = credentials("brave-browser-sccache-linux-s3-bucket")
                    }
                    stages {
                        stage ("unlock") {
                            steps {
                                sh "rm -rf ${GIT_CACHE_PATH}/*.lock"
                            }
                        }
                        stage ("install") {
                            steps {
                                sh "npm install"
                            }
                        }
                        stage ("init") {
                            when {
                                not {
                                    expression { return fileExists("src/brave/package.json") }
                                }
                            }
                            steps {
                                sh "npm run init"
                            }
                        }
                        stage ("sync") {
                            steps {
                                sh "npm run sync --all"
                            }
                        }
                        stage ("lint") {
                            steps {
                                sh """
                                    git -C src/brave config user.name brave-builds
                                    git -C src/brave config user.email devops@brave.com

                                    git -C src/brave checkout -b ${LINT_BRANCH}

                                    npm run lint

                                    git -C src/brave checkout -q -
                                    git -C src/brave branch -D ${LINT_BRANCH}
                                """
                            }
                        }
                        stage ("build") {
                            steps {
                                sh """
                                    npm config --userconfig=.npmrc set brave_referrals_api_key ${REFERRAL_API_KEY}
                                    npm config --userconfig=.npmrc set brave_google_api_endpoint https://location.services.mozilla.com/v1/geolocate?key=
                                    npm config --userconfig=.npmrc set brave_google_api_key ${BRAVE_GOOGLE_API_KEY}
                                    npm config --userconfig=.npmrc set google_api_endpoint "safebrowsing.brave.com"
                                    npm config --userconfig=.npmrc set google_api_key "dummytoken"
                                    npm config --userconfig=.npmrc set sccache "sccache"

                                    npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --debug_build=false --official_build=true
                                """
                            }
                        }
                        stage ("test-security") {
                            steps {
                                script {
                                    try {
                                        sh "npm run test-security -- --output_path=\"${OUT_DIR}/brave\""
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("test-unit") {
                            steps {
                                script {
                                    try {
                                        sh "npm run test -- brave_unit_tests ${BUILD_TYPE} --output brave_unit_tests.xml"
                                        xunit([GoogleTest(deleteOutputFiles: true, failIfNotNew: true, pattern: "src/brave_unit_tests.xml", skipNoTestFiles: false, stopProcessingIfError: true)])
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("test-browser") {
                            steps {
                                script {
                                    try {
                                        sh "npm run test -- brave_browser_tests ${BUILD_TYPE} --output brave_browser_tests.xml"
                                        xunit([GoogleTest(deleteOutputFiles: true, failIfNotNew: true, pattern: "src/brave_browser_tests.xml", skipNoTestFiles: false, stopProcessingIfError: true)])
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("dist") {
                            steps {
                                sh "npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} --debug_build=false --official_build=true"
                            }
                        }
                        stage ("archive") {
                            steps {
                                archiveArtifacts artifacts: "${OUT_DIR}/*.deb,${OUT_DIR}/*.rpm", fingerprint: true
                                awsIdentity()
                                s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "*.deb",
                                    path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                )
                                s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "*.rpm",
                                    path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                )
                                echo "https://s3.console.aws.amazon.com/s3/buckets/${BRAVE_ARTIFACTS_BUCKET}/${JOB_NAME}/${BUILD_NUMBER}/"
                            }
                        }
                    }
                }
                stage ("mac") {
                    agent { label "mac-ci" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                        SCCACHE_BUCKET = credentials("brave-browser-sccache-mac-s3-bucket")
                    }
                    stages {
                        stage ("unlock") {
                            steps {
                                sh "rm -rf ${GIT_CACHE_PATH}/*.lock"
                            }
                        }
                        stage ("install") {
                            steps {
                                sh "npm install"
                            }
                        }
                        stage ("init") {
                            when {
                                not {
                                    expression { return fileExists("src/brave/package.json") }
                                }
                            }
                            steps {
                                sh "npm run init"
                            }
                        }
                        stage ("sync") {
                            steps {
                                sh "npm run sync --all"
                            }
                        }
                        stage ("lint") {
                            steps {
                                sh """
                                    git -C src/brave config user.name brave-builds
                                    git -C src/brave config user.email devops@brave.com

                                    git -C src/brave checkout -b ${LINT_BRANCH}

                                    npm run lint

                                    git -C src/brave checkout -q -
                                    git -C src/brave branch -D ${LINT_BRANCH}
                                """
                            }
                        }
                        stage ("build") {
                            steps {
                                sh """
                                    npm config --userconfig=.npmrc set brave_referrals_api_key ${REFERRAL_API_KEY}
                                    npm config --userconfig=.npmrc set brave_google_api_endpoint https://location.services.mozilla.com/v1/geolocate?key=
                                    npm config --userconfig=.npmrc set brave_google_api_key ${BRAVE_GOOGLE_API_KEY}
                                    npm config --userconfig=.npmrc set google_api_endpoint "safebrowsing.brave.com"
                                    npm config --userconfig=.npmrc set google_api_key "dummytoken"
                                    npm config --userconfig=.npmrc set sccache "sccache"

                                    npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --debug_build=false --official_build=true
                                """
                            }
                        }
                        stage ("test-security") {
                            steps {
                                script {
                                    try {
                                        sh "npm run test-security -- --output_path=\"${OUT_DIR}/Brave\\ Browser\\ ${CHANNEL_CAPITALIZED}.app/Contents/MacOS/Brave\\ Browser\\ ${CHANNEL_CAPITALIZED}\""
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("test-unit") {
                            steps {
                                script {
                                    try {
                                        sh "npm run test -- brave_unit_tests ${BUILD_TYPE} --output brave_unit_tests.xml"
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                    xunit([GoogleTest(deleteOutputFiles: true, failIfNotNew: true, pattern: "src/brave_unit_tests.xml", skipNoTestFiles: false, stopProcessingIfError: true)])
                                }
                            }
                        }
                        stage ("test-browser") {
                            steps {
                                script {
                                    try {
                                        sh "npm run test -- brave_browser_tests ${BUILD_TYPE} --output brave_browser_tests.xml"
                                        xunit([GoogleTest(deleteOutputFiles: true, failIfNotNew: true, pattern: "src/brave_browser_tests.xml", skipNoTestFiles: false, stopProcessingIfError: true)])
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("dist") {
                            steps {
                                sh "npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} --debug_build=false --official_build=true --skip_signing"
                            }
                        }
                        stage ("archive") {
                            steps {
                                archiveArtifacts artifacts: "${OUT_DIR}/unsigned_dmg/*.dmg", fingerprint: true
                                withAWS(credentials: "mac-build-s3-upload-artifacts", region: "us-west-2") {
                                    awsIdentity()
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "unsigned_dmg/*.dmg",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                }
                                echo "https://s3.console.aws.amazon.com/s3/buckets/${BRAVE_ARTIFACTS_BUCKET}/${JOB_NAME}/${BUILD_NUMBER}/"
                            }
                        }
                    }
                }
                stage ("windows-x64") {
                    agent { label "windows-ci" }
                    environment {
                        GIT_CACHE_PATH = "${USERPROFILE}\\cache"
                        SCCACHE_BUCKET = credentials("brave-browser-sccache-win-s3-bucket")
                        PATH = "C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.17134.0\\x64\\;C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\Common7\\IDE\\Remote Debugger\\x64;${PATH}"
                        SIGNTOOL_ARGS = "sign /t  http://timestamp.verisign.com/scripts/timstamp.dll  /fd sha256 /sm"
                        CERT = "Brave"
                        KEY_CER_PATH = "C:\\jenkins\\digicert-key\\digicert.cer"
                        KEY_PFX_PATH = "C:\\jenkins\\digicert-key\\digicert.pfx"
                        AUTHENTICODE_PASSWORD = credentials("digicert-brave-browser-development-certificate-ps-escaped")
                        AUTHENTICODE_PASSWORD_UNESCAPED = credentials("digicert-brave-browser-development-certificate")
                    }
                    stages {
                        stage ("unlock") {
                            steps {
                                powershell "Remove-Item ${GIT_CACHE_PATH}/*.lock"
                            }
                        }
                        stage ("install") {
                            steps {
                                powershell "npm install"
                            }
                        }
                        stage ("init") {
                            when {
                                not {
                                    expression { return fileExists("src/brave/package.json") }
                                }
                            }
                            steps {
                                powershell "npm run init"
                            }
                        }
                        stage ("sync") {
                            steps {
                                powershell "npm run sync --all"
                            }
                        }
                        stage ("lint") {
                            steps {
                                powershell """
                                    git -C src/brave config user.name brave-builds
                                    git -C src/brave config user.email devops@brave.com

                                    git -C src/brave checkout -b ${LINT_BRANCH}

                                    npm run lint

                                    git -C src/brave checkout -q -
                                    git -C src/brave branch -D ${LINT_BRANCH}
                                """
                            }
                        }
                        stage ("build") {
                            steps {
                                powershell """
                                    npm config --userconfig=.npmrc set brave_referrals_api_key ${REFERRAL_API_KEY}
                                    npm config --userconfig=.npmrc set brave_google_api_endpoint https://location.services.mozilla.com/v1/geolocate?key=
                                    npm config --userconfig=.npmrc set brave_google_api_key ${BRAVE_GOOGLE_API_KEY}
                                    npm config --userconfig=.npmrc set google_api_endpoint "safebrowsing.brave.com"
                                    npm config --userconfig=.npmrc set google_api_key "dummytoken"
                                    # npm config --userconfig=.npmrc set sccache "sccache"

                                    npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --debug_build=false --official_build=true
                                """
                            }
                        }
                        stage ("test-security") {
                            steps {
                                script {
                                    try {
                                        powershell "npm run test-security -- --output_path=\"${OUT_DIR}/brave.exe\""
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("test-unit") {
                            steps {
                                script {
                                    try {
                                        powershell "npm run test -- brave_unit_tests ${BUILD_TYPE} --output brave_unit_tests.xml"
                                        xunit([GoogleTest(deleteOutputFiles: true, failIfNotNew: true, pattern: "src/brave_unit_tests.xml", skipNoTestFiles: false, stopProcessingIfError: true)])
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("test-browser") {
                            steps {
                                script {
                                    try {
                                        powershell "npm run test -- brave_browser_tests ${BUILD_TYPE} --output brave_browser_tests.xml"
                                        xunit([GoogleTest(deleteOutputFiles: true, failIfNotNew: true, pattern: "src/brave_browser_tests.xml", skipNoTestFiles: false, stopProcessingIfError: true)])
                                    }
                                    catch (ex) {
                                        currentBuild.result = "UNSTABLE"
                                    }
                                }
                            }
                        }
                        stage ("dist") {
                            steps {
                                powershell """
                                    Import-PfxCertificate -FilePath \"${KEY_PFX_PATH}\" -CertStoreLocation "Cert:\\LocalMachine\\My" -Password (ConvertTo-SecureString -String \"${AUTHENTICODE_PASSWORD_UNESCAPED}\" -AsPlaintext -Force)
                                    npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} --debug_build=false --official_build=true
                                """
                                powershell '(Get-Content src\\brave\\vendor\\omaha\\omaha\\hammer-brave.bat) | % { $_ -replace "10.0.15063.0\", "" } | Set-Content src\\brave\\vendor\\omaha\\omaha\\hammer-brave.bat'
                                powershell "npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} --build_omaha --tag_ap=x64-${CHANNEL} --target_arch=x64 --debug_build=false --official_build=true"
                            }
                        }
                        stage ("archive") {
                            steps {
                                    archiveArtifacts artifacts: "${OUT_DIR}/BraveBrowser*.exe", fingerprint: true
                                    awsIdentity()
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "brave_installer_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "BraveBrowser${CHANNEL_CAPITALIZED}Setup_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "BraveBrowserSilent${CHANNEL_CAPITALIZED}Setup_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "BraveBrowserStandalone${CHANNEL_CAPITALIZED}Setup_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "BraveBrowserStandaloneSilent${CHANNEL_CAPITALIZED}Setup_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "BraveBrowserStandaloneUntagged${CHANNEL_CAPITALIZED}Setup_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    s3Upload(acl: "Private", bucket: "${BRAVE_ARTIFACTS_BUCKET}", includePathPattern: "BraveBrowserUntagged${CHANNEL_CAPITALIZED}Setup_*.exe",
                                        path: "${JOB_NAME}/${BUILD_NUMBER}/", pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: "${OUT_DIR}"
                                    )
                                    echo "https://s3.console.aws.amazon.com/s3/buckets/${BRAVE_ARTIFACTS_BUCKET}/${JOB_NAME}/${BUILD_NUMBER}/"
                            }
                        }
                    }
                }
            }
        }
    }
}
