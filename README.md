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
Tạo 1 Gemfile tại root của project:
```sh
touch Gemfile
```
Sau đó thêm vào: 
```ruby
# Đặt tên file là Gemfile
source "https://rubygems.org"

gem "fastlane"
```
Chạy lệnh để cài fastlane sau khi đã có Gemfile
```sh
bundle install
```
Tạo một Fastfile ở trong fastlane/Fastfile, onefarm ios và android đã có sẵn Fastfile cùng các custom action. tất cả các file cần thiết đều nằm trong thư mục `fastlane/`

### Flutter
Tải flutter 2.2.1 cho macos tại [dây](https://storage.googleapis.com/flutter_infra_release/releases/stable/macos/flutter_macos_2.2.1-stable.zip)

Sau đó unzip và thêm flutter vào path
```sh
cd ~/Documents
unzip ~/Downloads/flutter_macos_2.2.1-stable.zip
flutter_root=$(pwd)
echo export PATH="$PATH:$flutter_root/flutter/bin" >> ~/.zshrc
source ~/.zshrc
```

Kiểm tra flutter
```sh
flutter doctor
```

### Kiểm tra
1. Kiểm tra jenkins đang chạy trên `http://localhost:8080`
2. Kiểm tra bundler và fastlane
```sh
bundle --version
bundle update
bundle exec fastlane --version
```

3. Chuẩn bị app iOS và android

### Set up certificate và provisioning profile bằng match
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

Ấn enter khi được yêu cầu nhập url git, sau khi chạy xong lệnh thì sửa file Matchfile để đặt url git và branch để lưu các file trên 1 nhánh riêng của git (các file này sẽ được mã hoá trước khi push lên)

```ruby
git_url("<git_url>")
git_branch("<git_branch>")

storage_mode("git")

type("appstore")
# type("adhoc") # The default type, can be: appstore, adhoc, enterprise or development

app_identifier("<bundle id>")
username("<username>")
```
Thay git_url, git_branch bằng url và branch của git, thay bundle_id bằng bundle id của app và thay username bằng username của git (vd: thanhpt)

Sau đó tìm đường dẫn đến các file .cer .p12 .mobileprovision đã tải xuống ở bước trước rồi chạy lệnh 
```sh
bundle exec fastlane match import
```
Để import các file đó lên git, cung cấp đường dẫn đến các file như yêu cầu.

#### Lưu ý: 
Sửa `type("appstore")` sang `type("adhoc")` khi cần import provisioning profile để build adhoc cho OTA distribution.

### Set up keystore và JSON key trên android
Bắt buộc phải có service account để chạy CI cho app android. User có quyền **Admin** cần vào Google Play Console > API Access > Add Service Account và cấp quyền **Google Service Account** để tạo 1 service account được kết nối với google play console. Sau đó vào cài đặt của service account tạo 1 JSON key dùng để login vào google play bằng service account. Tải JSON key này xuống máy và lưu lại địa chỉ file.

Tạo một keystore để sign app android bằng android studio. keystore này cũng nên được mã hoá và đẩy lên git.

#### Lưu ý : 
* Để upload được lên appstore thì app phải được ký bằng certificate loại iOS Distribution
* Provisioning profile phải link với bundle id của app và iOS Distribution certificate
* Service account cần được cấp quyền service account mới hiện lên trên Google Play Console

# Pipeline
## Các password quan trọng cần lưu
* Tên login và password của bitbucket hoặc api key
* Apple App-specific Password để deploy lên testflight
* JSON key của service account để đăng nhập vào google play console
* đường dẫn đến file keystore và password của keystore 

## Secret text
Jenkins có môi trường lưu trữ mật khẩu mã hoá tại **Manage Jenkins** > **Credentials**. Thêm các mật khẩu và đặt key để lưu thành biến môi trường trong Jenkinsfile

## Fastfile
Fastlane sử dụng các block script nhỏ gọi là `lane` cùng với các `action` để làm các task theo yêu cầu của người dùng. Các lane được khai báo trong fastfile rồi được gọi bằng lệnh 
```
bundle exec fastlane <lane>
```
với `<lane>` là tên của lane đã khai báo trong Fastfile.

#### Lane: Bump build number 
#### Lưu ý : flutter app
Với app flutter, version code và build number được lưu trên file pubspec.yaml và được sync với app mobile khi chạy flutter build. Hiện tại fastlane không hỗ trợ xử lý phiên bản cho flutter và không khuyến khích dùng các action `increment_build_number`, `increment_version_number` vì sẽ mất đồng bộ phiên bản giữa các platform. Để quản lý phiên bản và build number cho project dùng flutter, thêm custom action ở link dưới [đây](www.google.com) vào thư mục fastlane/actions. Action này sau đó được gọi với parameter là đường dẫn đến file pubspec.yaml và build number hiện tại (optional: nếu không có parameter build_number thì build number tìm thấy trong file pubspec.yaml sẽ được tăng lên 1). Ví dụ ở dưới tìm build number lớn nhất trên Appstore và Google Play rồi tăng build number thêm 1.

```ruby
desc "increase build number"
lane :increment_flutter_version_code do |options| # specifying version, defaults to 1.0
    print "options: #{options}"
    if options[:is_ci] then
        json_key_location = options[:json_key]
    else
        json_key_location = ENV['JSON_KEY_LOCATION']
    end
    version = options[:version] ? options[:version] : "1.0.0" # defaults to 1.0.0
    ci_track = options[:track] ? options[:track] : "internal" # defaults to internal
    ci_package_name = options[:package_name] ? options[:package_name] : "<bundle_id>"
    if options[:is_ci] then
        sh("git status --porcelain")
        # ensure_git_status_clean   
    end 
    print "version: #{version}"

    ios_build = latest_testflight_build_number(version: version)
    android_build = google_play_track_version_codes(
        track: ci_track,
        json_key: json_key_location,
        package_name: ci_package_name)

    print "build: #{ios_build}, #{android_build}"
    current_dir = sh("pwd").strip

    increment_version_code_android(
      config_file_path: "#{current_dir}/../../pubspec.yaml",
      build_number: [Actions.lane_context[Actions::SharedValues::LATEST_BUILD_NUMBER], Actions.lane_context[Actions::SharedValues::LATEST_TESTFLIGHT_BUILD_NUMBER]].max
    )
end
```
thay `<bundle_id>` bằng bundle id của app. lane sẽ được gọi với các parameter is_ci, version, ci_track, ci_package_name

```sh
bundle exec fastlane increment_flutter_version_code is_ci:true version:1.0.0 ci_track:internal ci_package_name:<bundle_id>
```

#### Lane: build ios
```ruby
desc "Build and sign the app using an ios distribution profile"
lane :build do |options| # Lane is run AFTER flutter build ios  
    is_ci = options[:is_ci]
    update_code_signing_settings(path: "Runner.xcodeproj" ,use_automatic_signing: false)
    # match(type: "appstore", readonly: true)
    gym(
      output_directory: "./build",
      clean: true,
      export_options:{
        compileBitcode: false,
        signingStyle: "manual",
        provisioningProfiles: ENV['MATCH_PROVISIONING_PROFILE_MAPPING'],
        export_method: "app-store"
      }
    )

    print "IPA is at: #{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]}"
    post_build(lane: "build")
end
```

#### Lane: deploy lên Testfilght 
```ruby
desc "upload the app to testflight"
lane :release do |options|
    if options[:is_ci] then
        print "Requires appleid_app_password to be set"
    end
    print "IPA is at: #{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]}"
    
    pilot(
      ipa: lane_context[SharedValues::IPA_OUTPUT_PATH],
      skip_waiting_for_build_processing: true
    )

    post_release
end
```

#### Lưu ý: Các lane phải được gọi nối tiếp nhau ở trong một lane để có thể chia sẻ cho nhau những biến `lane_context`, nếu gọi 2 lane riêng biệt thì lane `release` sẽ không tìm thấy biến `IPA_OUPUT_PATH` được tạo ra bởi gym: 

```ruby
lane :beta do |options|
    # pre_build_match
    build(is_ci: true)
    test(lane: post_build)
    release(is_ci: true)
end
```


#### Lane: flutter build android 
Flutter cần file key.properties ở root để có thể tự động build và sign apk. Vì file này không được phép track trên git nên Jenkins cần phải tạo file này trong 1 stage của pipeline trước khi build bằng flutter.

```sh
steps {
    sh '''cd one_farm/android
    echo "storePassword=$KEYSTORE_PASSWORD" > key.properties
    echo "keyPassword=$KEYSTORE_PASSWORD"  >> key.properties
    echo "keyAlias=upload" >> key.properties
    echo "storeFile=$KEY_STORE_LOCATION" >> key.properties
    cat key.properties'''
}
```

```ruby
desc "build apk"
lane :build_apk do
    current_dir = sh("pwd").strip
    get_flutter_version_code(config_file_path: "#{current_dir}/../../pubspec.yaml")
    sh("flutter build apk")  
    post_build(lane: "build_apk")
end
```

#### Lane: Upload to Google Play Store
```ruby
desc "Upload to Google play"
lane :upload do |options|
    type = options[:type] ? options[:type] : "aab"
    track = options[:track] ? options[:track] : "internal"
    release_status = options[:release_status] ? options[:release_status] : "draft"

    if type == "aab"
      sh("cp", "../../build/app/outputs/bundle/release/app-release.aab", "release.aab")
          upload_to_play_store(track: track, aab: 'fastlane/release.aab', package_name: '<package_name>', release_status: release_status)

    elsif type == "apk"
      sh("cp", "../../build/app/outputs/flutter-apk/app-release.apk", "release.apk")
      upload_to_play_store(track: track, apk: 'fastlane/release.apk', package_name: '<package_name>', release_status: release_status)
    end
    # upload_to_play_store_internal_app_sharing(apk: 'myapplication.apk')
    post_release
end
```
Thay `<package name>` bằng package name của app. lane sẽ được gọi với các parameter type, track, release_status
```sh
bundle exec fastlane upload type:apk track:internal release_status:draft
```

#### Lưu ý: 
* App phải đuọc upload bằng tay trước khi dùng fastlane để package name của app được đăng kí trên google play console 
* App chưa release thì release_status bắt buộc phải là `draft`
* App chưa release thì không up được lên internal_app_sharing
* Nếu app đã release trên 1 track thì apk trên track đấy không thể bị thay đổi


## Jenkinsfile
Tạo 1 Jenkinsfile ở root của repo, thêm những stage cơ bản như là clone từ git, build và release
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
            }
        }
        
        stage('Build and deploy') {
            
            parallel {
                stage('Build and deploy iOS') {
                    stages {
                        stage('Pre build iOS') {
                            steps {
                                # Setup everything necessary for build
                            }
                        }
                        stage('build and sign iOS') {
                            steps {
                                # build and sign the ios app
                            }
                        }
                        stage('Build and deploy to test flight') {
                            steps {
                                # Deploy to beta testing
                            }    
                        }
                    }
                }
                
                stage('Build and deploy Android') {
                    stages {
                        stage('set up') {
                            steps {
                                # Setup everything necessary for build
                            }
                        }
                        stage('Build aab') {
                            steps {
                                # Build aab
                            }
                        }
                        stage('Build apk') {
                            steps {
                                # build  apk
                            }
                        }
                        stage('Upload google play') {
                            steps {
                                # Upload apk or aab to google play
                            }
                        }
                    }
                }
            }
        }
        
    }
}

```

## Tạo pipeline
Trên Jenkins, nhấn vào **New item** > **Pipeline** để tạo một pipeline mới. 

## Generic Webhook Trigger 
Plugin này của Jenkins dùng webhook để chạy pipeline khi nhận được post request từ git. Định nghĩa webhook (`https://developer.github.com/webhooks/`):
> Webhooks allow you to build or set up integrations, which subscribe to certain events. When one of those events is triggered, we'll send a HTTP POST payload to the webhook's configured URL. Webhooks can be used to update an external issue tracker, trigger CI builds, update a backup mirror, or even deploy to your production server. You're only limited by your imagination.

### Post content parameter
Webhook từ các scm chứa nhiều thông tin liên quan đến repo và các commit mới nhất. Jenkins dùng JSON Path để lưu những thông tin đấy dưới dạng biến môi trường. Trong phần **Build Trigger** > **Generic Webhook Triggr** > **Post Content Parameter** > **Add** của mục cài đặt pipeline. Thêm một biến `message` như sau : 

![postparam](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/postparam.png)

Một vài ví dụ JSON Path cho webhook của bitbucket :
* Author của commit mới nhất: `$.push.changes[0].new.target.author.raw`
* remote repository: `$.repository.full_name`
* branch: `$.push.changes[0].new.name`

### Filter
Ở mục **Optional filter**, thêm filter cho biến message để bỏ qua các webhook từ commit tăng build number của fastlane
```
Expression: ^((?!\[fastlane\]|\[skip ci]).)*$
Text : $message
```

![filter_message](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/filter_message.png)

Sau khi xong bước này thì ấn **Build now** để kiểm tra Pipeline sau đó push lên git để kiểm tra webhook.

# Ví dụ Jenkinsfile và Fastfile của android và iOS
Jenkinsfile : 
```ruby
pipeline {
    agent any
    
    options {
        ansiColor('xterm')
    }
    
    environment {
        KEYSTORE_PASSWORD = credentials('keystore_password')
        KEY_STORE_LOCATION = credentials('key_store_location')
        JSON_KEY_FILE = credentials('json_key_file')
        APPLEID_APP_PASSWORD = credentials('appleid_app_password')
        GITLAB_API_KEY = credentials('gitlab_api_key')
        SCM_BRANCH = 'dev_1.0_new'
    }   
    stages {
        stage('Fetch repo') {
            steps {
                git branch: 'dev_1.0_new', url: 'https://Thanhphan1147:$GITLAB_API_KEY@gitlab.com/anhdv282/one_farm.git'            
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
                                bundle exec fastlane upload is_ci:true type:apk'''
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
                git add one_farm/pubspec.yaml
                git commit -m "[skip-ci] [fastlane] update build number"
                git reflog
                git remote
                git branch
                git pull origin $SCM_BRANCH
                git status --porcelain
                # git push origin $SCM_BRANCH
                '''
            }
        }
    }
}

```
Fastfile android : 
```ruby
update_fastlane
default_platform(:android)

platform :android do

  lane :bump_major do
    current_dir = sh("pwd")

    flutter_version_manager(
      arguments: "-major",
      yml: "#{current_dir}/../../version.yml",
      pubspec: "#{current_dir}/../../pubspec.yaml")
  end

  lane :bump_minor do
    current_dir = sh("pwd")

    flutter_version_manager(
      arguments: "-minor",
      yml: "#{current_dir}/../../version.yml",
      pubspec: "#{current_dir}/../../pubspec.yaml")
  end

  lane :bump_patch do
    current_dir = sh("pwd")
    
    flutter_version_manager(
      arguments: "-patch",
      yml: "#{current_dir}/../../version.yml",
      pubspec: "#{current_dir}/../../pubspec.yaml")
  end
  
  desc "build aab"
  lane :build_aab do
    current_dir = sh("pwd").strip
    get_flutter_version_code(config_file_path: "#{current_dir}/../../pubspec.yaml")
    sh("flutter build appbundle")
    post_build(lane: "build_aab")  
  end

  desc "build apk"
  lane :build_apk do
    current_dir = sh("pwd").strip
    get_flutter_version_code(config_file_path: "#{current_dir}/../../pubspec.yaml")
    sh("flutter build apk")  
    post_build(lane: "build_apk")
  end

  desc "Upload to Google play"
  lane :upload do |options|
    type = options[:type] ? options[:type]:"aab"
    track = options[:track] ? options[:track]:"internal"
    release_status = options[:release_status] ? options[:release_status]:"draft"

    if type == "aab"
      sh("cp", "../../build/app/outputs/bundle/release/app-release.aab", "release.aab")
          upload_to_play_store(track: track, aab: 'fastlane/release.aab', package_name: '****', release_status: release_status)

    elsif type == "apk"
      sh("cp", "../../build/app/outputs/flutter-apk/app-release.apk", "release.apk")
      upload_to_play_store(track: track, apk: 'fastlane/release.apk', package_name: '****', release_status: release_status)
    end
    # upload_to_play_store_internal_app_sharing(apk: 'myapplication.apk')
    post_release
  end

  private_lane :post_build do |options|
    build_number = Actions.lane_context[SharedValues::FLUTTER_BUILD_NUMBER]
    version_code = Actions.lane_context[SharedValues::FLUTTER_VERSION_CODE]

    apache_root = "/usr/local/var/www"
    archive_dir_prefix = "#{apache_root}/download/android"
    app_prefix = "build-#{build_number}"
    current_dir = sh("pwd").strip
    sh("mkdir", "-p", "#{archive_dir_prefix}/apk/#{version_code}", "#{archive_dir_prefix}/aab/#{version_code}")

    if options[:lane] == "build_apk"
      sh("cp", "#{current_dir}/../../build/app/outputs/flutter-apk/app-release.apk", "#{archive_dir_prefix}/apk/#{version_code}/#{app_prefix}.apk")
    elsif options[:lane] == "build_aab"
      sh("cp", "#{current_dir}/../../build/app/outputs/bundle/release/app-release.aab", "#{archive_dir_prefix}/aab/#{version_code}/#{app_prefix}.aab")
    end

    print "Build status: Build #{version_code}+#{build_number} finished successfully"
    notification(
      title: "Build status",
      message: "Build '#{version_code}+#{build_number}' finished successfully",
      open: "http://localhost:8080"
    )
  end

  private_lane :post_release do |options|
    print "successfully uploaded to Google play"
    notification(
      title: "Google Play Upload status",
      message: "successfully uploaded to google play"
    ) # Mac OS X Notification
  end

  error do |lane, exception|
    print "error in lane #{lane}: #{exception}"
    notification(
      title: "Build error",
      message: "Error occured in lane : '#{lane}': #{exception}"
    )
  end
end
```
Fastfile iOs : 
```ruby
update_fastlane
default_platform :ios

platform :ios do

  ### PRE BUILD ###
  desc "Pull certificate information from app store connect"
  lane :pre_build_match do
    match(type: "appstore", readonly: true)
  end

  desc "increase build number"
  lane :increment_flutter_version_code do |options| # specifying version, defaults to 1.0
    print "options: #{options}"
    if options[:is_ci] then
        json_key_location = options[:json_key]
    else
        json_key_location = ENV['JSON_KEY_LOCATION']
    end
    version = options[:version] ? options[:version] : "1.0.0" # defaults to 1.0.0
    ci_track = options[:track] ? options[:track] : "internal" # defaults to internal
    ci_package_name = options[:package_name] ? options[:package_name] : "***" # default to onefarm
    
    if options[:is_ci] then
        sh("git status --porcelain")
        # ensure_git_status_clean   
    end 
    print "version: #{version}"
    
    ios_build = latest_testflight_build_number(version: version)
    android_build = google_play_track_version_codes(
        track: ci_track,
        json_key: json_key_location,
        package_name: ci_package_name)
    
    print "build: #{ios_build}, #{android_build}"
    current_dir = sh("pwd").strip
    
    increment_version_code_android(
      config_file_path: "#{current_dir}/../../pubspec.yaml",
      build_number: [Actions.lane_context[Actions::SharedValues::LATEST_BUILD_NUMBER], Actions.lane_context[Actions::SharedValues::LATEST_TESTFLIGHT_BUILD_NUMBER]].max
    )
  end

  desc "pre build setup"
  lane :pre_build do
    pre_build_match
  end

  desc "write version_code and build_number to jenkins"
  lane :write_label do
    current_dir = sh("pwd").strip
    get_flutter_version_code(config_file_path: "#{current_dir}/../../pubspec.yaml")
    File.open("#{current_dir}/../../buildlabel.txt", "w") { |f| f.write "ONE Farm CI: release-#{Actions.lane_context[SharedValues::FLUTTER_VERSION_CODE]}+#{Actions.lane_context[SharedValues::FLUTTER_BUILD_NUMBER]}" }
  end

  ### LANES ### 

  lane :test do |options|
    version = options[:version] ? options[:version] : "1.0.0"
    ci_track = options[:track] ? options[:track] : "internal" # defaults to internal
    ci_package_name = options[:package_name] ? options[:package_name] : "****" # default to onefarm

    ios_build = latest_testflight_build_number(version: version)
    android_build = google_play_track_version_codes(
        track: ci_track,
        json_key: json_key_location,
        package_name: ci_package_name)

    print "#{Actions.lane_context[Actions::SharedValues::LATEST_TESTFLIGHT_BUILD_NUMBER]}"
    print "#{[Actions.lane_context[Actions::SharedValues::LATEST_BUILD_NUMBER], Actions.lane_context[Actions::SharedValues::LATEST_TESTFLIGHT_BUILD_NUMBER]].max}"
  end

  lane :beta do |options|
    # pre_build_match
    build(is_ci: true)
    test(lane: post_build)
    release(is_ci: true)
  end

  desc "Build and sign the app using an ios distribution profile"
  lane :build do |options| # Lane is run AFTER flutter build ios  
    is_ci = options[:is_ci]
    update_code_signing_settings(path: "Runner.xcodeproj" ,use_automatic_signing: false)
    # match(type: "appstore", readonly: true)
    gym(
      output_directory: "./build",
      clean: true,
      export_options:{
        compileBitcode: false,
        signingStyle: "manual",
        provisioningProfiles: ENV['MATCH_PROVISIONING_PROFILE_MAPPING'],
        export_method: "app-store"
      }
    )

    print "IPA is at: #{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]}"
    post_build(lane: "build")
  end

  desc "Build and sign the app using an ios ad-hoc profile"
  lane :build_ota do |options| # Lane is run AFTER flutter build ios  
    current_dir = sh("pwd").strip
    get_flutter_version_code(config_file_path: "#{current_dir}/../../pubspec.yaml")

    is_ci = options[:is_ci]
    update_code_signing_settings(path: "Runner.xcodeproj" ,use_automatic_signing: false)
    match(type: "adhoc", readonly: true)
    gym(
      output_directory: "./build",
      clean: true,
      export_options:{
        compileBitcode: false,
        signingStyle: "manual",
        provisioningProfiles: ENV['MATCH_PROVISIONING_PROFILE_MAPPING'],
        export_method: "ad-hoc"
      }
    )

    print "IPA is at: #{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]}"
    post_build(lane: "build_ota")

  end

  #### distribution lane ####

  desc "upload the app to testflight"
  lane :release do |options|
    if options[:is_ci] then
        print "Requires appleid_app_password to be set"
    end
    print "IPA is at: #{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]}"
    pilot(
      ipa: lane_context[SharedValues::IPA_OUTPUT_PATH],
      skip_waiting_for_build_processing: true
    )

    post_release
  end

  ### POST BUILD ###

  # This lane is called, only if the executed lane was successful
  private_lane :post_build do |options|
    lane = options[:lane]

    sh("echo", "Running post build actions")
    build = Actions.lane_context[Actions::SharedValues::FLUTTER_BUILD_NUMBER]
    
    if lane == "build_ota" then
        build_number = Actions.lane_context[SharedValues::FLUTTER_BUILD_NUMBER]
        version_code = Actions.lane_context[SharedValues::FLUTTER_VERSION_CODE]
        ipa_name = "build-#{build_number}.ipa"
        release_dir = "/usr/local/var/www/download/ios/#{version_code}"
        
        sh("mkdir", "-p", "#{release_dir}")
        sh("cp", "#{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]}", "#{release_dir}/#{ipa_name}")

        apache_root = "/usr/local/var/www"
        manifest_file_path = "#{apache_root}/manifest.plist"
        ipa_path_regexp = /^(\s*)(<string>http:\/\/)(\d+\.\d+\.\d+\.\d+)\/download\/ios\/(\d+.\d+.\d+)\/(.+.ipa)<\/string>$/
        version_code_regexp = /^(\s*)(<string>)(\d+\.\d+\.\d+)(<\/string>)$/
        Tempfile.open(".#{File.basename(manifest_file_path)}", File.dirname(manifest_file_path)) do |tempfile|
            File.open(manifest_file_path).each do |line|
                if line[ipa_path_regexp] then
                    tempfile.puts line.gsub(ipa_path_regexp) { |s| $1 + $2 + $3 + "/download/ios/#{version_code}/#{ipa_name}</string>" }
                elsif line[version_code_regexp] then
                    tempfile.puts line.gsub(version_code_regexp) { |s| $1 + $2 + "#{version_code}" + $4 }
                else
                    tempfile.puts line
                end
            end
            tempfile.close
            FileUtils.mv tempfile.path, "#{release_dir}/manifest.plist"
        end 
    end

    notification(
      title: "Build status",
      message: "Build '#{build}' on lane #{lane} finished successfully",
      open: "http://localhost:8080"
    ) # Mac OS X Notification
  end

  private_lane :post_release do
    print "uploaded ipa: #{Actions.lane_context[SharedValues::IPA_OUTPUT_PATH]} to testflight"
    notification(
      title: "TestFlight upload status",
      message: "successfully uploaded to testflight",
      open: "http://localhost:8080"
    ) # Mac OS X Notification
  end

  error do |lane, exception|
    print "Build error in lane #{lane}: #{exception}"
    notification(
      title: "Build error",
      message: "Error occured in lane : '#{lane}', #{exception}"
    )
  end

end
```

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)
   
   [icon]: <https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/icon.png>
   [ngrok]: <https://ngrok.com>
   [Regexp]: <https://www.regextester.com/15>
   [JSON path]: <https://jsonpathfinder.com>
