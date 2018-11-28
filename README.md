# Aptoide Gradle Deployer
Upload your APK directly from gradle

## Project Gradle

Add this information to your buildscript:

```gradle
buildscript {
    repositories {
        // ...
        maven { url "https://plugins.gradle.org/m2/" }
    }
    dependencies {
        // ...
        classpath "gradle.plugin.io.github.http-builder-ng:http-plugin:0.1.1"
    }
}
```

## Module Gradle

Add these imports:

```gradle
import groovy.json.JsonSlurper
import io.github.httpbuilderng.http.HttpTask
import static groovyx.net.http.MultipartContent.multipart
```

Get Aptoide data from local.properties (or another method):

```gradle
ext {
    Properties properties = new Properties()
    properties.load(project.rootProject.file('local.properties').newDataInputStream())
    
    propAptoideUsername = properties.getProperty('aptoideUsername')
    propAptoidePassword = properties.getProperty('aptoidePassword')
    propAptoideRepo = properties.getProperty('aptoideRepo')
    propAptoideApkName = properties.getProperty('aptoideApkName')
    propAptoideApkPath = properties.getProperty('aptoideApkPath')
}
```

Apply an HTTP plugin:

```gradle
apply plugin: "io.github.http-builder-ng.http-plugin"
```

Add this gradle task:

```gradle
task uploadToAptoide(type: HttpTask) {
    String accessToken = null
    config {
        request.uri = 'https://webservices.aptoide.com'
    }
    post {
        logger.info("Getting access token from Aptoide")
        request.uri.path = '/webservices/3/oauth2Authentication'
        request.contentType = 'application/json'
        request.body = [grant_type: 'password', client_id: 'Aptoide', mode: 'json', username: propAptoideUsername, password: propAptoidePassword]
        response.parser('application/json'){ cc, fs ->
            Map json = new JsonSlurper().parse(fs.inputStream) as Map
            logger.info("oauth2Authentication server response: $json")
            if(json.containsKey("error")) {
                logger.error("Aptoide ERROR - [${json.error}]: ${json.error_description}")
                logger.error("Could not log in to Aptoide. Please ensure that you have these values in your local.properties file: aptoideUsername, aptoidePassword, aptoideRepo, aptoideApkName")
            } else {
                accessToken = json.access_token
                logger.info("Access Token: ${accessToken}")
            }
        }
    }
    post {
        logger.info("Upload apk from: ${propAptoideApkPath}")
        request.uri.path = '/webservices/3/uploadAppToRepo'
        request.contentType = 'multipart/form-data'
        request.body = multipart {
            field 'access_token', accessToken
            field 'repo', propAptoideRepo
            field 'mode', 'json'
            field 'only_user_repo', 'false'
            part 'apk', propAptoideApkName, 'application/octet-stream', new File(propAptoideApkPath)
        }
        request.encoder 'multipart/form-data', groovyx.net.http.OkHttpEncoders.&multipart

        response.parser('application/json') { cc, fs ->
            Map json = new JsonSlurper().parse(fs.inputStream) as Map
            logger.info("uploadAppToRepo server response: $json")
            if(json.status == "OK") {
                logger.info("Aptoide upload status: ${json.status}")
            } else {
                json.errors.each { err -> logger.error("Aptoide ERROR - [${err.code}]: ${err.msg}") }
            }
        }
    }
}
```

Now you can run

```
gradlew uploadToAptoide
```
