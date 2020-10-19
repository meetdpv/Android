def cleanup_workspace() {
  cleanWs()
  dir("${env.WORKSPACE}@tmp") {
    deleteDir()
  }
  dir("${env.WORKSPACE}@script") {
    deleteDir()
  }
  dir("${env.WORKSPACE}@script@tmp") {
    deleteDir()
  }
}

pipeline {
  agent any
  // only required if you actually use node in the build process
  tools {
    nodejs 'node-lts'
  }
  environment {
    APPSTORECONNECT_TEAMID = '1310680'// only required if build for iOS
  }

  stages {
    // 1
    stage('prepare') {
      steps {
        script {
          // get the package version
          PACKAGE_VERSION = sh(
            script: 'node --print --eval "require(\'./package.json\').version"',
            returnStdout: true
          ).trim();

          echo("Package version is '${PACKAGE_VERSION}'");

          // Define when to build the app
          BRANCH_IS_MASTER = env.BRANCH_NAME == 'master';
          BUILD_APP = BRANCH_IS_MASTER;

          // prepare dependencies to use when setting up the android/ios projects
          sh('npm install');
          stash(includes: 'node_modules/', name: 'node_modules');
        }
      }
    }

    // 2
    /**
      * just run a dev build if we're not on master, so that every commit gets
      * at least built once. we don't do a production build here, because
      * dev-builds are fast, while production builds take a lot of ressources
      */
    stage('test build') {
      when {
        expression {!BUILD_APP}
      }
      steps {
        unstash('node_modules');
        sh ('npm run build');
      }
    }

    stage('prepare build') {
      when {
        expression {BUILD_APP}
      }
      parallel {
        // 3
        stage("build base android app") {
          agent {
            label "master"
          }

          steps {
            // we need the platform so that the ng run app:ionic-cordova command
            // can produce platform sepcific cordova code in www, and correctly
            // setup this code in the platform-folder
            unstash('node_modules');
            sh('npm run add_android');
            sh('npm run prepare_android');

            // these folders are all we need to later build the actual app
            stash(includes: 'www/, platforms/', name: 'base_android_build');
          }
          post {
            always {
              cleanup_workspace();
            }
          }
        }

        stage("build base ios app") {
          agent {
            label "master"
          }

          steps {
            // we need the platform so that the ng run app:ionic-cordova command
            // can produce platform sepcific cordova code in www, and correctly
            // setup this code in the platform-folder
            unstash('node_modules');
            sh('npm run add_ios');
            sh('npm run prepare_ios');

            // these folders are all we need to later build the actual app
            stash(includes: 'www/, platforms/', name: 'base_ios_build');
          }
          post {
            always {
              cleanup_workspace();
            }
          }
        }
      }
    }

    stage('build and deploy') {
      when {
        expression {BUILD_APP}
      }
      parallel {

        // 4
        stage('Android app') {
          agent {
            label "docker"
          }
          stages {
            stage("setup build dependencies") {
              steps {
                unstash('base_android_build');
                // the docker plugin automatically mounts the current folder als working directory,
                // so we need to neither mount it ourselves, nor define the workdir ourselves.
                // see for mor info: https://github.com/jenkinsci/docker-plugin/issues/561
                dir("${env.WORKSPACE}/platforms/android") {
                  script {
                    // create gradlew file in the android project folder. This is needed by fastlane
                    docker
                      .image('thyrlian/android-sdk')
                      /**
                      * we run as root inside the docker container, otherwise this step
                      * will fail if additional android sdk components need to be
                      * installed. This makes some files in the android folder have
                      * incorrect ownership, but that will be corrected in the build step
                      */
                      .inside('--user=0:0') { c ->
                      sh "sdkmanager 'build-tools;28.0.3' && cd '${env.WORKSPACE}/platforms/android' && gradle wrapper";
                    }
                  }
                }
              }
            }
            stage("prepare project") {
              steps {
                script {
                  docker
                    .image('bigoloo/gitlab-ci-android-fastlane')
                    // we run as root inside the docker container, otherwise the installed tools won't be accessible
                    .inside('--user=0:0') { c ->
                      sh ("""
                        APP_VERSION=${PACKAGE_VERSION} \
                        BUILD_NUMBER=${BUILD_NUMBER} \
                        fastlane prepare_android
                      """);
                  }
                }
              }
            }
            stage("build") {
              steps {
                script {
                  // We need these later to correct file ownership
                  CURRENT_USER = sh (script: "id -u", returnStdout: true).trim();
                  CURRENT_GROUP = sh (script: "id -g", returnStdout: true).trim();
                  
                  withCredentials([
                    file(credentialsId: 'android_keystore', variable: 'KEYSTORE_FILE'),
                    string(credentialsId: 'android_keystore_password', variable: 'KEYSTORE_PASSWORD'),
                    string(credentialsId: 'android_signing_key_alias', variable: 'SIGNING_KEY_ALIAS'),
                    string(credentialsId: 'android_singing_key_password', variable: 'SIGNING_KEY_PASSWORD'),
                  ]) {
                    /**
                    * The ${KEYSTORE_FILE}-path points to a file in ${env.WORKSPACE}@tmp/secretFiles.
                    * fastlane or gradle don't seem to handle the "@" in that path correctly,
                    * so we copy it over to our regular workspace folder, that doesn't contain an "@"
                    */
                    sh "cp ${KEYSTORE_FILE} ${env.WORKSPACE}/android.keystore";
                    docker
                      .image('bigoloo/gitlab-ci-android-fastlane')
                      // we run as root inside the docker container, otherwise the installed tools won't be accessible
                      .inside('--user=0:0') { c ->
                        sh("""yes | sdkmanager --licenses &&
                          KEYSTORE_FILE="${env.WORKSPACE}/android.keystore" \
                          KEYSTORE_PASSWORD=${KEYSTORE_PASSWORD} \
                          SIGNING_KEY_ALIAS=${SIGNING_KEY_ALIAS} \
                          SIGNING_KEY_PASSWORD=${SIGNING_KEY_PASSWORD} \
                          fastlane build_android
                        """)
                        
                        // make the build is accessible for the user outside the docker container.
                        // without this, uploading and workspace-cleanup won't work
                        sh "chown -R ${CURRENT_USER}:${CURRENT_GROUP} ./platforms/android/*";
                        sh "chown -R ${CURRENT_USER}:${CURRENT_GROUP} ./platforms/android/.*";
                    }
                  }
                }
              }
            }
         
          post {
            always {
              cleanup_workspace();
            }
          }
        }

       
          post {
            always {
              cleanup_workspace();
            }
          }
        }
      }
    }

    // 6
    stage('cleanup') {
      steps {sh ':'}
    }
  }

  post {
    always {
      script {
        cleanup_workspace();
      }
    }
  }
}
