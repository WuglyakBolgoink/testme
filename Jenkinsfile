def platform = params?.PLATFORM?.trim()                      // e.g. "ios" or "android"
BUILD_CONFIG = params?.BUILD_CONFIG?.trim()                 // e.g. "Debug" or "Release"

PROJECT_NAME = "TestMe"
CLEAN = true                          // Do a clean build and sign
INFO_PLIST = "${PROJECT_NAME}/${PROJECT_NAME}-Info.plist"
VERSION = "1.0.0"
SHORT_VERSION = "1.0"
BUNDLE_ID = "de.cyberkatze.testme"
OUTPUT_FILE_NAME = "${PROJECT_NAME}-${BUILD_CONFIG}.ipa".replace(" ", "").toLowerCase()
SDK = "iphoneos"

if (BUILD_CONFIG.toLowerCase() == "debug") {
    OSX_BUILD_CONFIG = "Debug"
} else if (BUILD_CONFIG.toLowerCase() == "release" || BUILD_CONFIG.toLowerCase() == "distribution") {
    OSX_BUILD_CONFIG = "Release"
}


pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    stages {

        stage('Check global module') {
            steps {
                script {
                    sh "java -version"
                    sh "cordova --version"
                    sh "node --version"
                    sh "npm --version"
                }
            }
        }

        stage('Initialize') {
            steps {
                script {
                    // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - Set ENV variables
                    env.LOGICAL_BRANCH_NAME = env.BRANCH_NAME
                    env.BUILD_TIER = "integ"

                    if (env.BRANCH_NAME == 'release/release') {
                        env.BUILD_TIER = "stage"
                        env.LOGICAL_BRANCH_NAME = 'staging'
                    } else if (env.BRANCH_NAME == 'master') {
                        env.BUILD_TIER = "prod"
                    }

                    env.SERVICE_ID = "test-me"
                    env.SERVICE_VERSION = "${env.LOGICAL_BRANCH_NAME}.${env.BUILD_NUMBER}"

                    // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - Outputs

                    echo "BRANCH_NAME: $env.BRANCH_NAME"
                    echo "LOGICAL_BRANCH_NAME: $env.LOGICAL_BRANCH_NAME"
                    echo "BUILD_TIER: $env.BUILD_TIER"
                    echo "SERVICE_ID: $env.SERVICE_ID"
                    echo "SERVICE_VERSION: $env.SERVICE_VERSION"

                    // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - GIT
                    sh "pwd"
                    sh "ls -la"
                    // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- - GIT
                    sh "git clean -f && git reset --hard origin/$env.BRANCH_NAME"
                }
            }
        }

        stage('Cordova Prepare') {
            steps {
                sh 'npm install --production'
                sh "cordova platform rm ${platform}"
                sh "cordova platform add ${platform}"
                sh "cordova prepare ${platform}"
                sh 'rm -rf node_modules'
            }
        }

        stage("Cordova Build") {
            if (platform == 'android') {
                if (BUILD_CONFIG == 'debug') {
                    sh "cordova build ${platform} --debug"
                } else {
                    sh "cordova build ${platform} --release"
                }
            } else {
                xcodeBuild(
                        cleanBeforeBuild: CLEAN,
                        src: "./platforms/${platform}",
                        schema: "${PROJECT_NAME}",
                        workspace: "${PROJECT_NAME}",
                        buildDir: "build",
                        sdk: "${SDK}",
                        version: "${VERSION}",
                        shortVersion: "${SHORT_VERSION}",
                        bundleId: "${BUNDLE_ID}",
                        infoPlistPath: "${INFO_PLIST}",
                        xcodeBuildArgs: 'ENABLE_BITCODE=NO OTHER_CFLAGS="-fstack-protector -fstack-protector-all"',
                        autoSign: false,
                        config: "${OSX_BUILD_CONFIG}"
                )
            }
        }

        stage("Sign") {
            if (platform == 'android') {
                if (BUILD_CONFIG == 'release') {
                    signAndroidApks(
                            keyStoreId: "${params.BUILD_CREDENTIAL_ID}",
                            keyAlias: "${params.BUILD_CREDENTIAL_ALIAS}",
                            apksToSign: "platforms/android/**/*-unsigned.apk",
                            // uncomment the following line to output the signed APK to a separate directory as described above
                            // signedApkMapping: [ $class: UnsignedApkBuilderDirMapping ],
                            // uncomment the following line to output the signed APK as a sibling of the unsigned APK, as described above, or just omit signedApkMapping
                            // you can override these within the script if necessary
                            // androidHome: '/usr/local/Cellar/android-sdk'
                    )
                } else {
                    println('Debug Build - Using default developer signing key')
                }
            }
            if (platform == 'ios') {
                codeSign(
                        profileId: "${CODE_SIGN_PROFILE_ID}",
                        clean: CLEAN,
                        verify: true,
                        ipaName: OUTPUT_FILE_NAME,
                        appPath: "platforms/${platform}/build/${OSX_BUILD_CONFIG}-${SDK}/${PROJECT_NAME}.app"
                )
            }
        }

        stage("Archive") {
            if (platform == 'android') {
                archiveArtifacts artifacts: "platforms/android/build/outputs/apk/android-${BUILD_CONFIG}.apk", excludes: 'platforms/android/build/outputs/apk/*-unaligned.apk'
            }
            if (platform == 'ios') {
                archiveArtifacts artifacts: "platforms/${platform}/build/${OSX_BUILD_CONFIG}-${SDK}/${OUTPUT_FILE_NAME}"
            }
        }
    }

    post {
        success {
            emailext(
                    subject: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]'",
                    body: """<p>SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]':</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}@[${
                        env.LOGICAL_BRANCH_NAME
                    }]]</a>&QUOT;</p>""",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                    mimeType: 'text/html'
            )
        }
        failure {
            emailext(
                    subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]'",
                    body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${
                        env.LOGICAL_BRANCH_NAME
                    }]':</p><hr><p>Build trigger: $cause</p><hr>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}@[${
                        env.LOGICAL_BRANCH_NAME
                    }]]</a>&QUOT;</p>""",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                    mimeType: 'text/html'
            )
        }
        unstable {
            emailext(
                    subject: "UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]'",
                    body: """<p>UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]@[${env.LOGICAL_BRANCH_NAME}]':</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}@[${
                        env.LOGICAL_BRANCH_NAME
                    }]]</a>&QUOT;</p>""",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                    mimeType: 'text/html'
            )
        }
    }
}
