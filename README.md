## This repo stores images used in this markdown document ##
# Jenkins
#### Download jenkins
MacOS
```sh
brew install jenkins-lts
brew service start jenkins-lts
```
Jenkins will be available at http://localhost:8080
#### First-time Jenkins configuration
A number of environment variables need to be set in order for jenkins to work properly, go to **Manage Jenkins** > **Configure System** > **Global properties** > **Environment variables** and set the following:
```
ANDROID_SDK_ROOT = <location of your android sdk> 
PATH = <your shell's path variable> (echo $PATH)
```
Plugins for jenkins:
- Generic web-hook trigger
- Locale
- On windows: powershell
- ANSI color
- Rebuild

If you are not using fastlane:

- Google play Android publisher
- Google Play OAuth Credentials
- Android app signing

## Build a pipeline for iOS
### Setup
#### Pre-requisites
1. Make sure jenkins is running by navigate to `http://localhost:8080` on your browser.
2. Check your fastlane installation: 
```shell
bundle update
bundle exec fastlane --version
```
3. Have a reade-to-build and deploy iOS app.
4. Have a valid appstore distribution and development certificate their corresponding provisioning profiles.

It is recommended to manualy sign your code and use `match` to handle your certificates and profiles. A guide to match can be found [here](https://docs.fastlane.tools/actions/match/)

### Jenkins-ios
#### Create a new pipeline
> Follow the official jenkins tutorial: https://www.jenkins.io/doc/pipeline/tour/getting-started/
Note that the official tutorial focus on creating a `pipeline` which requires the use of a `Jenkinsfile`. However, the commands that we in a **build step** of a `freestyle project` can be easily put into a `Jenkinsfile` and vice-versa  

Go to `http://localhost:8080` on your browser, click on **New item** > **Freestyle project** and enter a name.
Fastlane uses ANSI encoding for colored output, therefore you should enable ANSI color console output in **Build Enviroment**.
#### Setup a webhook
> Follow the bitbucket tutorial for managing webhooks : https://confluence.atlassian.com/bitbucketserver/managing-webhooks-in-bitbucket-server-938025878.html

Using a webhook causes an infinite loop of build triggers when you push from jenkins to the remote in the middle of the pipeline, see this [section](#build-loop) to see how to fix this problem.

#### Alternatives to webhook
If your git server doesn't support webhooks (Outdated, privacy policy, ...) you can create a simple `post-receive` hook script in your remote repository that will run after every push to the remote repository.

> Explanation on git hooks: https://git-scm.com/docs/githooks
> Required decent knowledge about shell scripting

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
![Fast](https://raw.githubusercontent.com/Thanhphan1147/images/master/fastlane.png)

In fastlane you use `increment_build_number` for versioning, `gym` to build and sign your ipa and `pilot` to deploy the app to testflight. They can be run in the CLI but it is recommended to put them in a `Fastfile`, full example can be found [here](https://github.com/Thanhphan1147/images/blob/master/fastlane/Fastfile).

#### Bump the build number 
The `increment_build_number` action are called before `gym` and `push_to_git_remote` is called after the build completed successfully
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
![jsonpath](https://raw.githubusercontent.com/Thanhphan1147/images/master/jsonpath.png)

To prevent commits by fastlane to trigger a build loop, use a custom git `username` and `email` for jenkins, and filter out this jenkins username using `regex` in the **Optional filter** section
```
expression: ^((?!(jenkins|placeholder)).)*$
text: <your variable>
```
![filter](https://raw.githubusercontent.com/Thanhphan1147/images/master/filter.png)

#### Useful tools
* [Regexp] tester
* [JSON path] finder


[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)
   
   [icon]: <https://raw.githubusercontent.com/Thanhphan1147/images/master/icon.png>
   [ngrok]: <https://ngrok.com>
   [Regexp]: <https://www.regextester.com/15>
   [JSON path]: <https://jsonpathfinder.com>
