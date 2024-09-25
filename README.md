<p align="center">
<img alt="logo image" width="250" src="https://trapp-documentation.s3.eu-central-1.amazonaws.com/LogoMakr-7gMmq0.png"  />
</p>

# TrAPP Android Library

TrAPP is a platform that allow to manage all the translations of mobile and web apps. This repository
contains the library to integrate TrAPP with Android mobile apps.
The library handles the following features:

- Authentication
- Synchronization
- Translation

## Overview

Every development platforms comes with the support of localization, to be able to translate text based on the preferences of the user's device.</br>
The main issue is that every platform uses different formats and the localization files must be added inside the application bundles, requiring an
update every time a string is changed.</br>
Moreover, keeping the files of every platform up to date is a constant task that can be time consuming and prone to errors.</br>
TrAPP helps keeping all the translations in a single point, always up to date and available to every component of the team.</br>
This Kotlin library allows Android developers to retrieve and use the translations uploaded on the platform with a simple integration, without worrying about the technical implementation.

## Configuration

### Library

To import the project go to the `build.gradle.kts` of the module, add the dependency

``` kotlin
dependencies {
    implementation("dev.sevencircle.trappsync:core:{trappsyncVersion}")
}
```

If you are using TOML, these are the settings

``` toml
[versions]
trappsync = {trappsyncVersion}

[libraries]
trappsync = { group = "dev.sevencircle.trappsync", name = "core", version.ref = "trappsync" }
```

### Plugin

If you want to use the [backup file or the offline mode](#backup-file-and-offline-mode) of the library, you can just import the offline plugin in the `build.gradle.kts` of the module:

``` gradle
plugins {
    id("dev.sevencircle.trappsync.plugin.offline") version {trappsyncVersion}
}
```

If you are using the TOML file:

``` toml
[versions]
trappsync = {trappsyncVersion}

[plugins]
trappsync = { id = "dev.sevencircle.trappsync.plugin.offline", version.ref = "trappsync" }
```

``` gradle
plugins {
    alias(libs.plugins.trappsync)
}
```

> [!CAUTION]
> If you use the plugin, we strongly recommend to remove the manual import of the library because the plugin will take care of that, downloading the latest compatible version. If you don't do this, you are are at risk of experiencing weird behaviors or crashes.

## Quick start

To access the TrAPP services you need use the `Translator` object. This is a singleton that holds the instance of `TranslatorService`. This instance than needs to be configured by using the function `init`.

``` kotlin
try {
    Translator.init {
        apiKey = "your API Key"
        primaryLanguage = "your primary language tag"
    }
} catch (e: DoubleConfigurationException) {
    // handle exception
}
```

It's suggested to initialize the translator in the `Application` as it would remove the necessity of catching the `DoubleConfigurationException`.

### Retrieve the state of the TranslatorService

To allow the correct use of the service, it's exposed a `StateFlow<TranslatorState>` that denotes the current state of the service.
You should check the state of the service before doing any operation.
</br>Look up the in-code documentation for further info.

### Automatic Locale detection

The library is capable of automatically detect the device locale and adapt consequently its behavior. This feature is enabled by default.

> [!NOTE]
> To disable this feature, simply set the value `localeDetection` to `false` in the service initialization.
>
> ``` kotlin
> Translator.init {
>     apiKey = "your API Key"
>     primaryLanguage = "your primary language tag"
>     localeDetection = false
> }
> ```

### Retrieve the TranslatorService instance

You can retrieve the instance by accessing it through the `Translator`, after the initialization, from everywhere in the app.

``` kotlin
val translatorService = Translator.instance
```

### Sync

To synchronize the local database with the data in the server, the `sync` function must be used.

``` kotlin
translatorService.sync()
```

The synchronization operation should be done at least once every time the app is started, possibly as first operation, to ensure that the strings are available before using them. Depending on the behavior of the app, the `sync` operation can also be done multiple time during the lifecycle of the app, without issues.

> [!TIP]
> A good way to be sure that the language in the app is updated with the strings in the platform is to to the `sync` operation during the `onResume` of the activity.
>
> ``` kotlin
> override fun onResume() {
>     super.onResume()
>     lifecycleScope.launch {
>         Translator.instance.sync()
>     }
> }
> ```

### Localization

After the synchronization, the localization keys can be translated.

``` kotlin
translatorService.translate("test.plain")
```

#### Templated Strings

A translation can contain some placeholders that need to be substitute with some values. To achieve this, the translation method accepts an array of String that contains the substitutions to the placeholders. The order of the array will be used to order the substitutions. For example the translation for the following strings will be:

``` json
"test.substring1": "Lorem {{0}} dolor sit amet, {{1}} adipiscing elit."
"test.substring2": "Lorem {{1}} dolor sit amet, {{0}} adipiscing elit."
```

``` kotlin
val string1 = translatorService.translate("test.substring1", "ipsum", "consectetur")
// string1 = "Lorem ipsum dolor sit amet, consectetur adipiscing elit."

val string2 = translatorService.translate("test.substring2", "ipsum", "consectetur")
// string2 = "Lorem consectetur dolor sit amet, ipsum adipiscing elit."
```

> [!WARNING]
> If the key is not found in the dataset, the returned string is the key itself.

### Change language

If you want to programmatically change change the language of the translations, the function `setLanguage` will allow to set and synchronize a new language.

``` kotlin
translator.setLanguage(languageCode = "en-US")
```

> [!CAUTION]
> If the `setLanguage` function is successful, the automatic locale detection will be disabled.

This function needs a string indicating the `languageCode` that needs to be set. This string should follow the language ISO standard, with two characters for the language, four optional characters for the script, and two characters for the country, like `en-US` or `zh-HANS-CN`.
The same language must be set also in the TrAPP online platform.

### Backup file and offline mode

Since the connection to the network is not always guaranteed during the first launch of the app, there's the possibility to provide the library a static dataset of strings from a `JSON` file that can be downloaded from the platform. To use this functionality, you'll need to also import the offline plugin and setup the instance to use the strings extracted from the `JSON` file (see [Plugin configuration](#plugin)).

``` kotlin
Translator.init {
    apiKey = "your API Key"
    primaryLanguage = "your primary language tag"
    offlineLanguages = parsedLanguages
}
```

If you don't want to make any API call to the platform, you can enable the offline mode:

``` kotlin
Translator.init {
    apiKey = "your API Key"
    primaryLanguage = "your primary language tag"
    offlineMode = true
    offlineLanguages = parsedLanguages
}
```

### Compose usage

To use the Translator in Compose you can use the following extension function:

``` kotlin
@Composable
fun String.translate(vararg substitutions: String, translatorService: TranslatorService? = null): String {
    if (LocalInspectionMode.current) return this
    val service = translatorService ?: Translator.instance
    val state by service.state.collectAsState()
    return remember(this, Locale.current, state) {
        if (state == TranslatorState.UpdatingTranslations || state == TranslatorState.Ready) {
            service.translate(key = this, substitutions = substitutions)
        } else {
            this
        }
    }
}
```

This function will ensure that the translator service is ready before getting the value from the cached values and that the value is up to date.

## License

This library is released under the MIT license. See [LICENSE](LICENSE.txt) for details.

## Issues

To report bugs go to [the issues page](https://github.com/7Circle/7circle_trapp_android/issues) and check if the bug has already been reported. Otherwise, open a new issue and follow the provided reporting template.

## About

The names and logo are trademarks of Var Group S.p.A.
