# TWA Steps

1. Initialize project with `No Activity` from Android Studio. Feel free to use Kotlin, as there should not be any code in Kotlin/Java for TWA.
2. Add this to `build.gradle` in **Project** level
    ```gradle
        allprojects {
            repositories {
                google()
                jcenter()
                maven { url "https://jitpack.io" }
            }
        }
    ```
3. Most tutorials (and the official guide) will recommend to add this to **Module**'s `build.gradle > dependencies` section:
    ```gradle
        implementation 'com.github.GoogleChrome.custom-tabs-client:customtabs:<VERSION>'
    ```
    where `<VERSION>` is the latest version of `customtabs` library (e.g.: `7a2c1374a3`, etc.)
4. However, the above may not work due to bug in chromium as referred in this [thread's  comment](https://bugs.chromium.org/p/chromium/issues/detail?id=983378#c11) since we're using `androidx`. So, we need to add these instead of `customtabs` library to **Module**'s `build.gradle > dependencies` section:
    ```gradle
        implementation 'androidx.browser:browser:1.0.0'
        implementation 'com.github.GoogleChrome:android-browser-helper:ff8dfc4ed3d4133aacc837673c88d090d3628ec8'
    ```
    This has been proven to work as of 27 Oct 2019. Also, add these to **Module**'s  `build.gradle > android` section:
    ```gradle
        compileOptions {
            sourceCompatibility JavaVersion.VERSION_1_8
            targetCompatibility JavaVersion.VERSION_1_8
        }
    ```
5. Then, go to `strings.xml` in `res` folder, and add the following:
    ```xml
        <string name="asset_statements">
            [{
                \"relation\": [\"delegate_permission/common.handle_all_urls\"],
                \"target\": {
                    \"namespace\": \"web\",
                    \"site\": \"https://<FE_URL>\"}
            }]
        </string>
    ```
    where `<FE_URL>` is the FE's deployment site. This **SHOULD** hide the URL bar in the TWA, **IF** the web provides `assetlinks.json` file in `.well-known` directory on the deployment folder.
6. Then, go to `AndroidManifest.xml` and replace with the following structure:
    ```xml
        <manifest xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:tools="http://schemas.android.com/tools"
            package="<PACKAGE_PATH>">

            <application
                android:allowBackup="true"
                android:icon="@mipmap/ic_launcher"
                android:label="@string/app_name"
                android:roundIcon="@mipmap/ic_launcher_round"
                android:supportsRtl="true"
                android:theme="@style/AppTheme"
                tools:ignore="GoogleAppIndexingWarning">

                <meta-data
                    android:name="asset_statements"
                    android:resource="@string/asset_statements" />

                <activity
                    android:name="com.google.androidbrowserhelper.trusted.LauncherActivity">

                    <meta-data
                        android:name="android.support.customtabs.trusted.DEFAULT_URL"
                        android:value="https://<FE_URL>" />

                    <intent-filter>
                        <action android:name="android.intent.action.MAIN" />
                        <category android:name="android.intent.category.LAUNCHER" />
                    </intent-filter>

                    <intent-filter>
                        <action android:name="android.intent.action.VIEW"/>
                        <category android:name="android.intent.category.DEFAULT" />
                        <category android:name="android.intent.category.BROWSABLE"/>

                        <!-- Edit android:host to handle links to the target URL-->
                        <data
                            android:scheme="https"
                            android:host="<FE_URL>"/>
                    </intent-filter>
                </activity>
            </application>
        </manifest>
    ```
    where `<PACKAGE_PATH>` is the application's package path, which may be `com.future.function.twa`, and `<FE_URL>` is the FE's deployment site.
7. After configuring association from app to FE_URL, we need to ensure FE_URL has connection to app, **so that URL bar from Google Chrome does not show on app**. Follow these steps:
	- Go to `Build > Generate Signed Bundle/APK`
	- Select create new `.jks` file, specify key store path and its name (usually `upload-keystore.jks`)
	- Set password to `functiontwa`, key alias `function-twa`, key password `functionkeypass`, and fill the rest of the field as instructed by Android Studio
	- Open bash/cmd on the app's root folder in the project, and type the following:
		```shell
			keytool -list -v -keystore ./upload-keystore.jks -alias function-twa -storepass functiontwa -keypass functionkeypass
		```
	  this will produce something like this:
		```shell
			Alias name: function-twa
			Creation date: Nov 2, 2019
			Entry type: PrivateKeyEntry
			Certificate chain length: 1
			Certificate[1]:
			Owner: CN=Function, OU=Future, O=Blibli.com, L=Jakarta Pusat, ST=DKI Jakarta, C=ID
			Issuer: CN=Function, OU=Future, O=Blibli.com, L=Jakarta Pusat, ST=DKI Jakarta, C=ID
			Serial number: 754aa563
			Valid from: Sat Nov 02 12:25:46 ICT 2019 until: Wed Oct 26 12:25:46 ICT 2044
			Certificate fingerprints:
			         MD5:  DB:FF:7B:BB:8B:E2:21:2E:79:33:4F:44:F4:7D:4B:2A
			         SHA1: F7:61:59:47:F7:65:1F:F6:C0:A0:DA:22:CE:44:87:52:FD:64:6F:1C
			         SHA256: A6:2C:7D:E3:B9:9A:3E:39:37:CA:F5:A7:AE:1F:E8:B4:F4:74:5D:EE:7F:6D:1F:9B:BA:35:F5:FD:A9:30:2B:7A
			Signature algorithm name: SHA256withRSA
			Subject Public Key Algorithm: 2048-bit RSA key
			Version: 3

			Extensions:

			#1: ObjectId: 2.5.29.14 Criticality=false
			SubjectKeyIdentifier [
			KeyIdentifier [
			0000: 48 A9 96 6A 74 F5 5A F4   4D 51 51 94 5D D3 9F 41  H..jt.Z.MQQ.]..A
			0010: 9D 45 37 C2                                        .E7.
			]
			]
		```
	- Copy the value from `SHA256` from above code, and go to [https://developers.google.com/digital-asset-links/tools/generator](https://developers.google.com/digital-asset-links/tools/generator), and fill the host name, app package, and the value from `SHA256`. Then click `Generate statement`, and copy its value.
	- Go to `public` folder in the FE project, and save the value to `.well-known/assetlinks.json`. This is where the association from the FE_URL to the app is established. When running build, it will be automatically put into `dist` folder, as this path (`FE_URL/./wellknown/assetlinks.json`) must be accessible by the app. Do not worry if accessing the link for `assetlinks.json` does not produce anything, as it may be redirected by the server to homepage rather than the file.
8. For testing the application:
    - Ensure `adb` is registered in environment variable
        * Type `adb` in your command/bash/terminal
        * If not recognized, then register to environment variable path `C:\Users\<USERNAME>\AppData\Local\Android\Sdk\platform-tools`, depending on where you install the `adb` (`<USERNAME>` is your computer's username).
    - Then, connect your phone to the computer
    - Open `Chrome` in your phone, go to `chrome://flags`. Search for `Enable command line on non-rooted devices`, and set to `Enabled`. When prompted to, restart your `Chrome` app in your phone.
    - Then, open command/bash/terminal, then type `adb devices`, you should be able to see at least 1 device connected to adb.
    - Run app:
        - Run MongoDB
        - Run Redis
        - Run BE web app
        - Build FE web app (`npm run build`)
        - Run FE server (using `nginx` or any other tool)
        - Go to desktop's Chrome, open `chrome://inspect`
        - Enable port forwarding, and input the port and hostname for FE server
        - Go to device, check if the hostname and the port is accessible from device's Chrome
    - Run the following line in the command/bash/terminal:
        ```shell
            adb shell "echo '_ --disable-digital-asset-link-verification-for-url=\"https://<FE_URL>\"' > /data/local/tmp/chrome-command-line"
        ```
        where `<FE_URL>` is the FE's deployment site.
    - Finally, run the app using `Android Studio`'s `Shift + F10` or `Run` button, targeting your phone.

---

## **NOTE:**
Further research is required to understand what is happening on the inside of PWA & TWA.
