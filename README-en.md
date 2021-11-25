# CI CD with Jenkins
This guide provides you with everything you need to know in details to be able to set up a CI/CD pipeline for your iOS or Android development projects. This guide covers setting up and install Jenkins, fastlane and their components. You can also find, at the end, an example Jenkinsfile and Fastfile for a flutter project here at VNPT.
## Setup
#### Jenkins Installation
Install Jenkins with homebrew on Mac OS
```sh
brew install jenkins-lts
```
#### Web interface
Jenkins's default web interface listens on localhost. To access Jenkins via LAN change the httpListenAddress field in `~/Library/LaunchAgents/homebrew.mxcl.jenkins-lts.plist` from `127.0.0.1` to `0.0.0.0`. This file is generated automatically during Jenkins installation process.

```sh
brew services start jenkins-lts
file=~/Library/LaunchAgents/homebrew.mxcl.jenkins-lts.plist
cp $file ~/tempfile
brew services stop jenkins-lts
perl -0777 -pe 's/(<string>--httpListenAddress=)(.*)(<\/string>)/$1"0.0.0.0"$3/' ~/tempfile
mv ~tempfile $file
```

Then start the jenkins service (the configuration file requires `root:wheel` permission): 
```sh
file=~/Library/LaunchAgent/homebrew.mxcl.jenkins-lts.plist
sudo chown root $file
sudo chgrp wheel $file
sudo systemctl load $file
```

Jenkins is now accessible at `http://localhost:8080`. Follow the instructions on the web GUI to login as admin and install plugins (I suggest installing recommended plugins).

#### Jenkins configuration
Add environment variables at **Manage Jenkins** > **Configure System** > **Global properties** > **Environment variables** , `Note that these variables are crucial for the operation of Jenkins when building Android and iOS apps` : 
```
ANDROID_SDK_ROOT = ~/Library/Android/sdk 
PATH = (run echo $PATH on your terminal and copy the result) 
```
Example : 
![envar](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/envar.png)

Jenkins plugins:
- Generic web-hook trigger
- Locale
- ANSI color
- Rebuilder

#### Fastlane 
Ruby and bundler installation: 

```sh
# Install rbenv for ruby version management
brew install rbenv
rbenv init
# Check if everything is ok
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor| bash
# Install ruby 2.7.3
rbenv install 2.7.3
# Switch to version 2.7.3
rbenv global 2.7.3
# Install bundler, note that ruby is required to run gem
gem install bundler
```
Create a Gemfile at project root:
```sh
touch Gemfile
```
Then add: 
```ruby
# Gemfile
source "https://rubygems.org"

gem "fastlane"
```
fastlane installation with Gemfile
```sh
bundle install
```
Now we create a Fastfile in fastlane/Fastfile, VNPT's iOS and android projects already have a Fastfile with its custom actions. Note that everything fastlane-related is in `fastlane/`

### Flutter
Flutter 2.2.1 macos installation :  [link](https://storage.googleapis.com/flutter_infra_release/releases/stable/macos/flutter_macos_2.2.1-stable.zip)

Unzip and add flutter to PATH
```sh
cd ~/Documents
unzip ~/Downloads/flutter_macos_2.2.1-stable.zip
flutter_root=$(pwd)
echo export PATH="$PATH:$flutter_root/flutter/bin" >> ~/.zshrc
source ~/.zshrc
```

Check if flutter works
```sh
flutter doctor
```

### Testing
1. Verify that Jenkins is running on `http://localhost:8080`
2. Check that bundler and fastlane is working and are of correct version
```sh
bundle --version
bundle update
bundle exec fastlane --version
```

3. Setting up build environement for iOS and Android apps

### Set up iOS certificate and provisioning profile with match
Required file in order to run match: 
- certificate (.cer)
- private key (.p12)
- profile     (.mobileprovision)

On mac, download your certificate from the Apple Developper Portal, double click to add it to your keychain keychain, then right-click to export the .cer and .key files.

![keychain](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/keychain.png)

Put your downloaded provisioning profile in the same folder as match

```sh
bundle exec fastlane match init
```

Press enter when prompted for a git url, then edit Matchfile to save your certificates to a separate branch of your project (Files will be encrypted before being pushed to git)

```ruby
git_url("<git_url>")
git_branch("<git_branch>")

storage_mode("git")

type("appstore")
# type("adhoc") # The default type, can be: appstore, adhoc, enterprise or development

app_identifier("<bundle id>")
username("<username>")
```
replace git_url, git_branch with your project's git url and the branch where you want to store your certificates (For example `certificates`), also replace bundle_id by your app's bundle id and username by your git username

Find the absolute path to the .cer .p12 .mobileprovision files that you downloaded and run: 
```sh
bundle exec fastlane match import
```
Provide the path to the files as required by match

#### Note: 
Change `type("appstore")` to `type("adhoc")` for OTA distribution. You will need ofcourse a different set of certificates and provisioning profiles.

### Set up keystore and JSON key on android
A service account is required to run CI for android apps. Users with **Admin** permission need to go to Google Play Console > API Access > Add Service Account and grant **Google Service Account** permission to create a service account that is connected to the google play console. Then go to the service account's settings and create a JSON key to login to google play with the service account. Download this JSON key to your computer and save the file's absolute path.

Create a keystore to sign android app using android studio. This keystore should also be encrypted and pushed to git.

#### Note : 
* To upload to the appstore, the app must be signed with a iOS Distribution certificate
* Provisioning profile must link with the app's bundle id and iOS Distribution certificate
* Service accounts needs to be granted service account permission to appear on Google Play Console

# Pipeline
## Important passwords to save
* Login name and password of bitbucket or api key
* Apple App-specific Password to deploy to testflight
* JSON key of service account to login to google play console
* path to keystore file and keystore password

## Secret text
Jenkins has an encrypted password storage environment at **Manage Jenkins** > **Credentials**. Add the passwords and set the key to save as an environment variable to be fetched in Jenkinsfile

## Fastfile
Fastlane uses small script blocks called `lane` along with `action` to do tasks according to user requirements. The lanes are declared in the fastfile and then called with the command
```
bundle exec fastlane <lane>
```
with `<lane>` the declared name of the lane in Fastfile.

#### Lane: Bump build number 
#### Note: flutter app
For flutter apps, the version code and build number are stored in the pubspec.yaml file and synced with the mobile app when flutter build is run. Currently, fastlane does not support version handling for flutter and does not recommend using the `increment_build_number`, `increment_version_number` actions because it will lose version synchronization between platforms. To manage the version and build number for projects using flutter, add a custom action at the link below [here](https://gitlab.com/anhdv282/one_farm/-/raw/cicd_dev_1.0_new/one_farm/ios/fastlane/ actions /increment_version_code_android.rb) to the fastlane/actions directory. This action is then called with the parameter being the path to the pubspec.yaml file and the current build number (optional: if there is no build_number parameter, the build number found in the pubspec.yaml file will be incremented by 1). The example below finds the largest build number on the Appstore and Google Play and then increases the build number by 1.



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

replace `<bundle_id>` with the bundle id of the app. lane will be called with the parameters is_ci, version, ci_track, ci_package_name

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

#### Lane: deploy to Testfilght 
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

#### Note: The lanes must be called consecutively in a wrapper lane to be able to share the `lane_context` variables, if calling separately, the `release` lane will not find the `IPA_OUPUT_PATH variable. ` generated by gym:

```ruby
lane :beta do |options|
    # pre_build_match
    build(is_ci: true)
    test(lane: post_build)
    release(is_ci: true)
end
```


#### Lane: flutter build android 
Flutter needs the key.properties file at root to automatically build and sign the apk. Since this file is not allowed to be tracked on git, Jenkins needs to create this file in a stage of the pipeline before building with flutter.

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
Replace `<package name>` with the package name of the app. lane will be called with the parameters type, track, release_status
```sh
bundle exec fastlane upload type:apk track:internal release_status:draft
```

#### Note:
* App must be uploaded manually before using fastlane for the package name of the app to be registered on google play console
* For unreleased apps, release_status must be `draft`
* Apps that are not released can't be uploaded to internal_app_sharing
* If the app has been released on a track, the apk on that track cannot be changed

## Jenkinsfile
Create a Jenkinsfile at the root of the repo, add basic stages like clone from git, build and release

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
On Jenkins, go to **New item** > **Pipeline** to create a new pipeline. 

## Generic Webhook Trigger 
This Jenkins plugin uses a webhook to run the pipeline when it receives a post request from git. Webhook definition (`https://developer.github.com/webhooks/`):
> Webhooks allow you to build or set up integrations, which subscribe to certain events. When one of those events is triggered, we'll send a HTTP POST payload to the webhook's configured URL. Webhooks can be used to update an external issue tracker, trigger CI builds, update a backup mirror, or even deploy to your production server. You're only limited by your imagination.

### Post content parameter
Webhooks from SCMs contain a lot of information regarding the repo and the latest commits. Jenkins uses JSON Path to save that information as environment variables. In the **Build Trigger** > **Generic Webhook Triggr** > **Post Content Parameter** > **Add** section of the pipeline settings. Add a variable `message` like this:

![postparam](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/postparam.png)

Some example JSON Path for bitbucket :
* Author of the latest commit: `$.push.changes[0].new.target.author.raw`
* remote repository: `$.repository.full_name`
* branch: `$.push.changes[0].new.name`

### Filter
In the **Optional filter** section, add a filter for the message variable to ignore webhooks from commits to increase fastlane's build number

```
Expression: ^((?!\[fastlane\]|\[skip ci]).)*$
Text : $message
```

![filter_message](https://raw.githubusercontent.com/Thanhphan1147/CI-CD-with-Jenkins/master/filter_message.png)

After completing this step, press **Build now** to test the Pipeline then push to git to test the webhook.

# Example Jenkinsfile and Fastfile for android and iOS deployment
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
