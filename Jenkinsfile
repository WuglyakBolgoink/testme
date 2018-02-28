// ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- -- Initialize

PLATFORM = "android" //params?.PLATFORM?.trim()                      // e.g. "ios" or "android"
BUILD_CONFIG = "Release"//params?.BUILD_CONFIG?.trim()                 // e.g. "Debug" or "Release"

CLEAN = true                          // Do a clean build and sign

PROJECT_NAME = "TestMe"
BUNDLE_ID = "de.cyberkatze.testme"
VERSION = "1.0.0"
SHORT_VERSION = "1.0"

INFO_PLIST = "${PROJECT_NAME}/${PROJECT_NAME}-Info.plist"
OUTPUT_FILE_NAME = "${PROJECT_NAME}-${BUILD_CONFIG}.ipa".replace(" ", "").toLowerCase()

SDK = "iphoneos"

if (BUILD_CONFIG.toLowerCase() == "debug") {
    OSX_BUILD_CONFIG = "Debug"
} else if (BUILD_CONFIG.toLowerCase() == "release" || BUILD_CONFIG.toLowerCase() == "distribution") {
    OSX_BUILD_CONFIG = "Release"
}

// ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- --

pipeline {

    environment {
        LOGICAL_BRANCH_NAME = env.BRANCH_NAME
        BUILD_TIER = "integ"

        if (env.BRANCH_NAME == 'release/release') {
            BUILD_TIER = "stage"
            LOGICAL_BRANCH_NAME = 'staging'
        } else if (env.BRANCH_NAME == 'master') {
            BUILD_TIER = "prod"
        }

        SERVICE_ID = "test-me"
        SERVICE_VERSION = "${env.LOGICAL_BRANCH_NAME}.${env.BUILD_NUMBER}"
    }

    agent any

    options {
        disableConcurrentBuilds()
    }

    stages {

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Check global module
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

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Initialize
        stage('Initialize') {
            steps {
                script {
                    echo "BRANCH_NAME: ${BRANCH_NAME}"
                    echo "LOGICAL_BRANCH_NAME: ${LOGICAL_BRANCH_NAME}"
                    echo "BUILD_TIER: ${BUILD_TIER}"
                    echo "SERVICE_ID: ${SERVICE_ID}"
                    echo "SERVICE_VERSION: ${SERVICE_VERSION}"


                    sh "pwd"
                    sh "ls -la"

                    sh "git clean -f && git reset --hard origin/${BRANCH_NAME}"
                }
            }
        }

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Cordova Prepare
        stage('Cordova Prepare') {
            steps {
                sh 'npm install --production'
                sh "cordova platform rm ${PLATFORM}"
                sh "cordova platform add ${PLATFORM}"
                sh "cordova prepare ${PLATFORM}"
                sh 'rm -rf node_modules'
            }
        }

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Cordova Build
        stage("Cordova Build") {
            if (PLATFORM == 'android') {
                if (BUILD_CONFIG == 'debug') {
                    sh "cordova build ${PLATFORM} --debug"
                } else {
                    sh "cordova build ${PLATFORM} --release"
                }
            } else {
                xcodeBuild(
                        cleanBeforeBuild: CLEAN,
                        src: "./platforms/${PLATFORM}",
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

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Sign
        stage("Sign") {
            if (PLATFORM == 'android') {
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
            if (PLATFORM == 'ios') {
                codeSign(
                        profileId: "${CODE_SIGN_PROFILE_ID}",
                        clean: CLEAN,
                        verify: true,
                        ipaName: OUTPUT_FILE_NAME,
                        appPath: "platforms/${PLATFORM}/build/${OSX_BUILD_CONFIG}-${SDK}/${PROJECT_NAME}.app"
                )
            }
        }

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Archive
        stage("Archive") {
            if (PLATFORM == 'android') {
                archiveArtifacts artifacts: "platforms/android/build/outputs/apk/android-${BUILD_CONFIG}.apk", excludes: 'platforms/android/build/outputs/apk/*-unaligned.apk'
            }
            if (PLATFORM == 'ios') {
                archiveArtifacts artifacts: "platforms/${PLATFORM}/build/${OSX_BUILD_CONFIG}-${SDK}/${OUTPUT_FILE_NAME}"
            }
        }

        // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- Publish Android
        stage("Publish Android") {
            if (PLATFORM == 'android') {
                androidApkUpload googleCredentialsId: 'My Google Play account', apkFilesPattern: '**/*.apk', trackName: 'alpha',
                        recentChangeList: [
                                [language: 'en-GB', text: "Please test the changes from Jenkins build ${env.BUILD_NUMBER}."],
                                [language: 'de-DE', text: "Bitte die Änderungen vom Jenkins Build ${env.BUILD_NUMBER} testen."]
                        ]
            }
        }
    }

    // ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- ---- --- Send post messages
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
