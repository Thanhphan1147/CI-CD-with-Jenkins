# Jenkins
## Setup
#### Tải jenkins
Trên macOS tải Jenkins bằng homebrew
```sh
brew install jenkins-lts
```
#### Giao diện web
Giao diện web của Jenkins mặc định nghe trên localhost. Để vào được Jenkins qua mạng LAN thì sửa trường httpListenAddress ở `~/Library/LaunchAgents/homebrew.mxcl.jenkins-lts.plist` từ `127.0.0.1` sang `0.0.0.0`. File này được tạo ra khi khởi động service jenkins bằng homebrew.

```sh
brew services start jenkins-lts
file=~/Library/LaunchAgents/homebrew.mxcl.jenkins-lts.plist
cp $file ~/tempfile
brew services stop jenkins-lts
perl -0777 -pe 's/(<string>--httpListenAddress=)(.*)(<\/string>)/$1"0.0.0.0"$3/' ~/tempfile
mv ~tempfile $file
```

Sau đó chạy jenkins (file config cần thuộc user root và nhóm wheel): 
```sh
file=~/Library/LaunchAgent/homebrew.mxcl.jenkins-lts.plist
sudo chown root $file
sudo chgrp wheel $file
sudo systemctl load $file
```

Truy cập jenkins tại `http://localhost:8080` và làm theo hướng dẫn để thiết lập lần đầu, tạo tài khoản admin và cài plugin (cài những plugin được recommended)

#### Cấu hình Jenkins lần đầu
Thêm biến môi trường ở **Manage Jenkins** > **Configure System** > **Global properties** > **Environment variables** , `Chú ý jenkins sẽ ko chạy được nếu ko có 2 biến này` : 
```
ANDROID_SDK_ROOT = Thông thưởng ở trên mac nó sẽ ở ~/Library/Android/sdk 
PATH = Chạy lệnh echo $PATH trên terminal và copy kết quả vào đây 
```
Ví dụ : 
![envar](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/envar.png)

Plugin cho jenkins:
- Generic web-hook trigger
- Locale
- ANSI color
- Rebuilder

#### Fastlane 
Tải ruby và bundler 

```sh
# Tải rbenv để quản lý phiên bản ruby
brew install rbenv
rbenv init
# Script check xem mọi thứ có ok không 
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor| bash
# Cài bản ruby 2.7.3
rbenv install 2.7.3
# Chuyển sang dùng bản 2.7.3
rbenv global 2.7.3
# Cài bundler, nếu có ruby thì mới chạy được lệnh gem
gem install bundler
```
Thêm Gemfile để dùng bundler (Trên repo của OneFarm đã có sẵn Gemfile):
```ruby
# Đặt tên file là Gemfile
source "https://rubygems.org"

gem "fastlane"
```
Chạy lệnh để cài fastlane sau khi đã có Gemfile
```sh
bundle install
```
Tạo một Fastfile ở trong fastlane/Fastfile, onefarm ios và android đã có sẵn Fastfile cùng các custom action. tất cả các file cần thiết đều nằm trong thư mục `fastlane\`

#### Flutter

### Kiểm tra
1. Kiểm tra jenkins đang chạy trên `http://localhost:8080`
2. Kiểm tra bundler và fastlane
```sh
bundle --version
bundle update
bundle exec fastlane --version
```

3. Kiểm tra flutter
```sh
flutter doctor
```

4. Chuẩn bị app iOS và android

### Set up certificate và provisioning profile
Các file cần có trước khi dùng match: 
- certificate (.cer)
- private key (.p12)
- profile     (.mobileprovision)

Ở trên mac, tải certificate xuống từ Apple Developper Portal, nhấn đúp để thêm vào keychain, rồi ấn chuột phải để export 2 file .cer và .key ra ngoài

![keychain](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/keychain.png)

Tải provisioning profile xuống cùng thư mục và khởi tạo match

```sh
bundle exec fastlane match init
```

Ấn enter khi được yêu cầu nhập url git, sau khi chạy xong lệnh thì sửa file Matchfile để đặt url git và branch để lưu các file trên 1 nhánh của repo (các file này sẽ được mã hoá trước khi push lên), ví dụ của onefarm. 

```ruby
git_url("https://stc.vnpt:afrrYW5GtbrxfN9b9DXE@gitlab.com/anhdv282/one_farm.git")
git_branch("certificates")

storage_mode("git")

type("appstore")
# type("adhoc") # The default type, can be: appstore, adhoc, enterprise or development

app_identifier("vn.vnpt-technology.ONEFarm")
username("stc.vnpt@gmail.com")
```
sửa trường type("appstore") sang type("adhoc") khi cần import provisioning profile để build adhoc 

### Set up keystore và JSON key trên android
Bắt buộc phải có service account để chạy CI cho app android. User có quyền **Admin** cần vào Google Play Console > API Access > Add Service Account và cấp quyền **Google Service Account** để tạo 1 service account được kết nối với google play console. Sau đó vào cài đặt của service account tạo 1 JSON key dùng để login vào google play bằng service account. Tải JSON key này xuống máy và lưu lại địa chỉ file.

Tạo một keystore để sign app android bằng android studio. keystore này cũng nên được mã hoá và đẩy lên git.

#### Lưu ý : 
* Để upload được lên appstore thì app phải được ký bằng certificate loại iOS Distribution
* Provisioning profile phải link với bundle id của app và iOS Distribution certificate
* Đối với onefarm chỉ cần chạy `bundle exec fastlane match appstore` để cài cả 3 vì certificate và profile đã được import lên git.
* Service account cần được cấp quyền service account mới hiện lên trên Google Play Console

# Pipeline
## Các password quan trọng cần lưu
* Tên login và password của bitbucket hoặc api key
* Apple App-specific Password để deploy lên testflight
* JSON key của service account để đăng nhập vào google play console
* đường dẫn đến file keystore và password của keystore 

## Secret text
Jenkins có môi trường lưu trữ mật khẩu mã hoá tại **Manage Jenkins** > **Credentials**. Thêm các mật khẩu và đặt key để lưu thành biến môi trường trong Jenkinsfile

## Jenkinsfile
Tạo 1 Jenkinsfile ở root của repo, ví dụ như Jenkinsfile của Onefarm : 
```sh
pipeline {
    agent any
    
    options {
    	# Load các plugins
        ansiColor('xterm')
    }
    
    environment {
    	# Load các secret text và chuyển thành biến môi trường
        KEYSTORE_PASSWORD = credentials('keystore_password')
        KEY_STORE_LOCATION = credentials('key_store_location')
        JSON_KEY_FILE = credentials('json_key_file')
        APPLEID_APP_PASSWORD = credentials('apple-application-password')
        GITLAB_API_KEY = credentials('gitlab_api_key')
        SCM_BRANCH = 'stc_vnpt_dev_0.9'  

    }   
    stages {
        stage('Fetch repo') {
            steps {
                git branch: "$SCM_BRANCH", url: "https://stc.vnpt:$GITLAB_API_KEY@gitlab.com/anhdv282/one_farm.git"            
                sh('pwd')
                script {
                    currentBuild.description = "branch: $SCM_BRANCH"
                    currentBuild.displayName = "ONE Farm CI: Pending..."
                }
            }
        }
        stage('Bump build number') {
            steps {
                sh '''cd one_farm/ios
                bundle update
                bundle update fastlane
                bundle exec fastlane ios increment_flutter_version_code is_ci:true json_key:$JSON_KEY_FILE track:internal
                bundle exec fastlane write_label
                '''
                script {
                    def label = readFile(file: 'one_farm/buildlabel.txt')
                    println(label)
                    currentBuild.displayName = label
                    currentBuild.description = "branch: $SCM_BRANCH"
                }
            }
        }
        stage('Build') {
            parallel {
                stage('Build and deploy iOS') {
                    stages {
                        stage('Pre build iOS') {
                            steps {
                                sh '''cd one_farm/ios
                                bundle exec fastlane ios pre_build is_ci:true'''   
                            }
                        }
                        stage('Flutter build iOS') {
                            steps {
                                sh'''cd one_farm/ios
                                flutter build ios --release --no-codesign'''
                            }
                        }
                        stage('Build and deploy to test flight') {
                            steps {
                                sh '''cd one_farm/ios
                                # bundle exec fastlane beta is_ci:true
                                '''
                            }    
                        }
                    }
                }
                
                stage('Build and deploy Android') {
                    stages {
                        stage('set up') {
                            steps {
                                sh'''cd one_farm/android
                                bundle --version
                                # bundle update'''
                            }
                        }
                        stage('Write keystore config') {
                            steps {
                                sh '''cd one_farm/android
                                echo "storePassword=$KEYSTORE_PASSWORD" > key.properties
                                echo "keyPassword=$KEYSTORE_PASSWORD"  >> key.properties
                                echo "keyAlias=upload" >> key.properties
                                echo "storeFile=$KEY_STORE_LOCATION" >> key.properties
                                cat key.properties'''
                            }
                        }
                        stage('Flutter build aab') {
                            steps {
                                sh '''pwd
                                cd one_farm/android
                                bundle exec fastlane build_aab is_ci:true'''
                            }
                        }
                        stage('Flutter build apk') {
                            steps {
                                sh '''pwd
                                cd one_farm/android
                                bundle exec fastlane build_apk is_ci:true'''
                            }
                        }
                        stage('Upload google play') {
                            steps {
                                sh '''pwd
                                cd one_farm/android
                                # bundle exec fastlane upload is_ci:true type:apk'''
                            }
                        }
                    }
                }
            }
        }
        
        stage('Build OTA for in-house distribution') {
            stages {
                stage('Pre build iOS') {
                    steps {
                        sh '''cd one_farm/ios
                        flutter build ios --release --no-codesign'''   
                    }
                }
                stage('Build OTA') {
                    steps {
                        sh '''cd one_farm/ios
                        bundle exec fastlane build_ota is_ci:true'''   
                    }
                }
            }
            
        }
        
        stage('Wrapping up') {
            steps {
                sh '''
                git status --porcelain
                git add one_farm/pubspec.yaml
                git commit -m "[skip-ci] [fastlane] update build number"
                git reflog
                git remote
                git branch
                # git push gitlab
                '''
            }
        }
    }
}

```

## Tạo pipeline
Trên Jenkins, nhấn vào **New item** > **Multibranch Pipeline** để Jenkins tự tạo các jobs dựa theo Jenkinsfile trên repo.

#### Setup a webhook
from `https://developer.github.com/webhooks/`:
> Webhooks allow you to build or set up integrations, which subscribe to certain events. When one of those events is triggered, we'll send a HTTP POST payload to the webhook's configured URL. Webhooks can be used to update an external issue tracker, trigger CI builds, update a backup mirror, or even deploy to your production server. You're only limited by your imagination.

> Follow the bitbucket tutorial for managing webhooks : https://confluence.atlassian.com/bitbucketserver/managing-webhooks-in-bitbucket-server-938025878.html

Using a webhook causes an infinite loop of build triggers when you push from jenkins to the remote in the middle of the pipeline, see this [section](#build-loop) to see how to fix this problem.

#### Alternatives to webhook
If your git server doesn't support webhooks (Outdated, privacy policy, ...) you can create a simple `post-receive` hook script in your remote repository that will run after every push to the remote repository.

> Explanation on git hooks: https://git-scm.com/docs/githooks
> Required knowledge about shell scripting

Sample script:
```sh
author=$(git log -1 --pretty=format:'%an')                                  # get the author of latest commit
repository=<"name of repository slug">
branch=$(git name-rev $(git log -1 --pretty='format:%C(auto)%h'))           # get the branch name that was pushed to

# processing (doesn't send trigger if branch != "master", check if author == "jenkins" , ...)

# Trigger the build
curl --location --request POST 'http://JENKINS_URL/generic-webhook-trigger/invoke?token=<your-token>' \
--header 'Content-Type: application/json' \
--data-raw '{"author", "$author", <post-content-parameter>": "<value>", ...'
```
In the bitbucket server's file system (You'll need admin priviledge on the server hosting bitbucket), name this script `21_post_receive` and store it in `shared/repositories/<slug-of-your-repo>/hooks/post_receive.d`

### Fastlane
> Follow the official fastlane tutorial for iOS : https://docs.fastlane.tools/getting-started/ios/setup/ 
If you are not an admin or account holder on App Store Connect, It is recommended to register a new bundle id (follow [this](https://subscription.packtpub.com/book/application_development/9781786464507/18/ch18lvl1sec87/creating-a-bundle-identifier) tutorial) and have the app created by an admin beforehand. 

After that go to your project's root directory and run: 
```sh
bundle exec fastlane init
```
To have your Fastfile configuration written in Swift (Beta):
```sh
bundle exec fastlane init swift
```
The command will automatically find the app that has the same bundle id as your project and create a `fastlane/` folder in your project's root directory:
![Fast](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/fastlane.png)

>fastlane is the easiest way to automate beta deployments and releases for your iOS and Android apps. 🚀 It handles all tedious tasks, like generating screenshots, dealing with code signing, and releasing your application.

For an IOS app you use `increment_build_number` for versioning, `gym` to build and sign your ipa and `pilot` to deploy the app to testflight. They can be run in the CLI but it is recommended to put them in a `Fastfile`, full example can be found [here](https://github.com/Thanhphan1147/CI-CD-with-Jenkins/blob/master/fastlane/Fastfile).

#### Bump the build number 
`increment_build_number` is called before `gym` and `push_to_git_remote` is called after the build completed successfully
```ruby
private_lane :<before-build-lane> do

    ensure_git_status_clean
    build = increment_build_number
    version = increment_version_number

end

private_lane :<after-build-lane> do |options|

    lane = options[:lane]
    build = Actions.lane_context[Actions::SharedValues::BUILD_NUMBER]
    
    commit_version_bump(
      message: "[fastlane] [skip ci] Incremented build number for build #{build}",
      no_verify: true
    )
    push_to_git_remote(
      remote: "origin",             # optional, default: "origin"
      local_branch: "HEAD",         # optional, aliased by "branch", default: "master"
      remote_branch: "master",      # optional, default is set to local_branch
      force: true,                  # optional, default: false
      tags: true                    # optional, default: true
    )
end
```
> Note: since jenkins works with git in a detached HEAD state, a local_branch option is required
#### Build and sign with gym
Note: The default build method for gym is app-store 
```ruby
desc "Build and sign the app using an ios distribution profile"
lane :build do
# pre-build actions (increment build and version number, git status check, setup variables to compute build time, ...)

# change codesigning setting to manual
update_code_signing_settings(path: "<path_to_xcodeproj>" ,use_automatic_signing: false)

gym(
  output_directory: "./build",
  clean: true,                                      # clean build directory before building
  export_options:{
    compileBitcode: false,
    signingStyle: "manual",
    provisioningProfiles:{
      "bundle_id": "provisioning profile"           # A mapping of bundle id -> provisioning profile
    }
  }
)
# post-build actions (commit changes and push back to remote, move your ipa to storage, send notifications, ...)
```
To run this lane in your CLI:
```sh
bundle exec fastlane 
```
In jenkins go to your pipeline **Configure** > **Build** and add a new build step `Execute shell`
```sh
#!/bin/bash
start=$(date "+%s")
bundle exec fastlane build
end=$(date "+%s")
echo "build finished in $((end-start)) s"
```
#### Testfilght distribution
```ruby
desc "upload the app to testflight"
lane :release do
pilot(
  ipa: "<path_to_ipa>",
  skip_waiting_for_build_processing: true # If you don't have admin access to the app setting this to false will cause your pipeline to fail due to insufficient permission
)
# post_release actions
end
```
run this lane in CLI
```sh
bundle exec fastlane release
```
Add another `Execute shell` build step in jenkins
```sh
#!/bin/bash
bundle exec fastlane release
```
## Build a pipeline for Android
### Setup
#### Pre-requisites
1. Make sure jenkins is running by navigate to `http://localhost:8080` on your browser.
2. Check your fastlane installation: 
```shell
bundle update
bundle exec fastlane --version
```
3. Have a ready-to-build and deploy android app in Android Studio
4. Have a service account with at least **Project Lead** permission in google play store
5. Have the json key of said service account stored as credentials in jenkins
> The process of adding service accounts and generate a JSON key can be followed [here](https://docs.fastlane.tools/getting-started/android/setup/) at **Setting up supply** section

#### Jenkins-android
Since jenkins configuration is similar for both android and ios, follow [this](#jenkins-ios) section of the iOS pipeline

### Fastlane
Go to your project's root directory and run: 
```sh
bundle exec fastlane init
```
To add plugins such as `firebase_app_distribution` run
```sh
fastlane add_plugin firebase_app_distribution
```
### Sign APK
#### Generate .jks key file
> Follow the official tutorial on code signing: https://developer.android.com/studio/publish/app-signing

After this process you should have a key file `<key-file-name>.jks` in your file system. Make sure to also store the `keystore-password`, the `key-alias` and the `key-password`
#### Sign using Gradle action
With fastlane you don't need to convert your jks key to p12 to store on Jenkins if it's available locally
```ruby
desc "build a release apk"
lane :build_release do
  gradle(
    task: "assemble",
    build_type: "Release",
    print_command: false,
    properties: {
      "android.injected.signing.store.file" => "<path-to-key>",
      "android.injected.signing.store.password" => "<keystore-password>",
      "android.injected.signing.key.alias" => "<key-alias>",
      "android.injected.signing.key.password" => "<key-password>",
    }
  )
end
```
If you are not using fastlane you need to install jenkins's `Android_signing` plugin and store your key as a jenkins credential (under p12 file extension). A tutorial can be found [here](https://github.com/jenkinsci/android-signing-plugin)
### Upload to Google Play Store
```ruby
desc "Upload to goolge play store"
lane :upload do
    sh("pwd")
    Dir.chdir ".." do
      sh("cp", "app/build/outputs/apk/release/app-release.apk", "myapplication.apk")
    end
    upload_to_play_store(track: 'internal', apk: 'myapplication.apk', package_name: '<package_name>', release_status: 'draft')
    # upload_to_play_store_internal_app_sharing(apk: 'myapplication.apk')
end
```
Note: 
- You must upload manually for the first time onto any track for the `<package_name>` of your app to be registered and discoverable by fastlane. 
- If an app is marked as `draft` on Google Play console you must specify `draft` as the release_status.
- You cannot publish to Google Play app sharing if your app is not published
- To publish your app you need to follow the **Getting Started** process on the app dashboard (Fill app informations, take screenshots, ...)
- If an app is published on a given track you cannot make further changes to the apk on that track, you can still make changed to other tracks if your app is not released there.

# Git 

#### Build loop
In Jenkins, use JSONpath to get push related information. They can be use to filter out commits that were not meant to trigger builds (Like version bump commits generated by fastlane). Some examples using `bitbucket web hooks`:
* Author of the latest commit: `$.push.changes[0].new.target.author.raw`
* remote repository: `$.repository.full_name`
* current branch: `$.push.changes[0].new.name`
* Information on the POST request sent to jenkins from a SCM can be obtain by reading their docs / using a proxy or a tunnel like [ngrok]

The values need to be assigned to an environment variable (**Build Trigger** > **Generic webhook trigger** > **Add post content parameter**) and the variables can be used across the build process. 
![jsonpath](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/jsonpath.png)

To prevent commits by fastlane to trigger a build loop, use a custom git `username` and `email` for jenkins, and filter out this jenkins username using `regex` in the **Optional filter** section
```
expression: ^((?!(jenkins|placeholder)).)*$
text: <your variable>
```
![filter](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/filter.png)

#### Useful tools
* [Regexp] tester
* [JSON path] finder


[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)
   
   [icon]: <https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/icon.png>
   [ngrok]: <https://ngrok.com>
   [Regexp]: <https://www.regextester.com/15>
   [JSON path]: <https://jsonpathfinder.com>
