---
layout: post
title: "Securing Flutter Apps with Auth0"
description: "Learn how to secure Flutter apps with Auth0 while using the Authorization Code Grant Flow with PKCE"
date: "2019-02-18 08:30"
author:
  name: "Fabusuyi Ayodeji"
  url: "roscoefab"
  mail: "dejifab@outlook.com"
  avatar: "https://twitter.com/roscoefab/profile_image?size=original"
tags:
- flutter
- auth0
- mobile
- pkce
- authorizations
related:
- 2017-11-15-an-example-of-all-possible-elements
--- 

**TL;DR:** [Flutter](https://flutter.io/) is Google's cross-platform SDK created to help developers build expressive and beautiful mobile applications. In this article, you will learn how to build and secure a Flutter application with Auth0 using the Authorization Code Grant Flow with PKCE. You can checkout the code developed throughout the article [in this GitHub repository](https://github.com/thedejifab/flutter_auth0).

## Prerequisites

Before getting started with this article, you need a working knowledge of Flutter. If you need help getting started, you can follow the codelabs on the [Flutter website](https://flutter.io/docs/codelabs). 

You also need to have the installations outlined below on your machine.

* [Flutter SDK](https://flutter.io/docs/get-started/install), version 1.0 or later. (This comes with a [Dart SDK](https://www.dartlang.org/install) installation)
* A Development Environment, one of:
  * [Android Studio](https://developer.android.com/studio), version 3.0 or later, or
  * [IntelliJ IDEA](https://www.jetbrains.com/idea/download/), version 2017.1 or later, or
  * Visual Studio Code.
  
  with Dart and Flutter plugins installed.

## OAuth 2.0 Flow and Mobile Applications

OAuth 2.0 is an industry-standard protocol for authorization. It allows the delegation of user **authorization** (not authentication) responsibilities to other services. A typical example of OAuth 2.0 in action is seen when trying to sign up for a third-party app using Facebook. OAuth 2.0 helps the third-party app delegate user authorization tasks to Facebook without having to bear the weight of securing user credentials. 
 
OAuth 2.0 provides [different flows](https://auth0.com/docs/api-auth/which-oauth-flow-to-use) for user authorization, with the [Authorization Code Grant Flow with PKCE](https://auth0.com/docs/api-auth/tutorials/authorization-code-grant-pkce) being the recommended approach for securing mobile applications. A detailed illustration of how this is used in the application developed in the article is shown below:

![Create application screenshot](images/flow.png)

## What You'll Build

Throughout this article, you'll implement an application that uses social login (with GitHub and Google) and database login. Upon logging in, the application will fetch and render the user' profile details as seen in the screenshots below.

<p><img src="images/prompt.png" width="300px" height="auto"/> <img src="images/login.png" width="300px" height="auto"/> <img src="images/profile.png" width="300px" height="auto"/></p>

## Project Setup

In this section, you will set up the application to be used throughout the article. More specifically, you will:

* Create an Auth0 application to represent your Flutter app,
* Scaffold a new Flutter app, and
* Install packages that the app is dependent on.

### Creating an Auth0 project

Auth0 is an Identity-as-a-Service (IDaaS) platform that provides enterprises with features such as [**Social Login**](https://auth0.com/learn/social-login/) and [**Passwordless Login**](https://auth0.com/passwordless) amongst many others, aimed at easing the process of online identity management.

To integrate Auth0 into your Flutter app, you need an Auth0 account. If you have an existing account, you can use it. If you don't, [click here to create one](https://auth0.com/signup).

After creating an Auth0 account, follow the steps below to set up an application:

* Go to the [**Applications**](https://manage.auth0.com/#/applications) section of your dashboard
* Click on the [**Create Application**](https://manage.auth0.com/#/applications/create) button.
* Enter a name for your application (e.g "Flutter Application").
* Finally, select **Native App** as the application type and click the **Create** button.

![Create application screenshot](images/auth0.png)

With database login and Google social login enabled by default, click on the **Connection > Social** menu on your dashboard. Then, toggle on **GitHub** as one of the enabled social connections. In the displayed dialog prompt, use the default details and click the **SAVE** button to finish.

Finally, navigate to the **Settings** tab of your application to set a callback URL in the **Allowed Callback URLs** field. This could be any value ranging from normal web URLs with an HTTP scheme (E.g `https://myapp.com`) to URIs with custom schemes (E.g `myapp://logincallback`). In my case, I used the latter and made it unique. If you don't know the purpose of the callback URL, don't worry, the article will explain this concept into details later.

### Scaffolding a Flutter project

To facilitate the process of creating a new Flutter project you are going to use the Flutter CLI tool. To do this, open a terminal and navigate to your projects directory to run the following command:

```bash
flutter create flutter_app
```

The CLI tool generates a template project within a couple of seconds to get you started. The tool requires an internet connection to download dependencies during project creation, except an `--offline` option is passed to the `create` command to defer the downloading of dependencies. After project generation, you can now open the project in your preferred IDE.

### Installing dependencies

As you will see in the course of the article, the project requires five main dependencies - [`http`](https://pub.dartlang.org/packages/http) for performing network requests, [`crypto`](https://pub.dartlang.org/packages/crypto) for generating cryptographically secure codes used in the authorization process, [`flutter_custom_tabs`](https://pub.dartlang.org/packages/flutter_custom_tabs) for launching URLs, [`shared_preferences`](https://pub.dartlang.org/packages/shared_preferences) for persistence, and [`uni_links`](https://pub.dartlang.org/packages/uni_links) which affords the ability to deep-link from web URLs into a Flutter application.

With the project open in your IDE, navigate to your `/pubspec.yaml` file to add the dependencies by modifying the `dependencies` section as seen below:

```yaml
dependencies:
  flutter:
    sdk: flutter
  crypto: ^2.0.6
  uni_links: ^0.1.4
  http: ^0.12.0+1
  flutter_custom_tabs: ^0.4.0
  shared_preferences: ^0.4.3
```

Then, run `flutter packages get` command in your project's root directory with a stable internet connection to download the dependencies.

## Flutter and Auth0 in Practice

In this section, you will learn how to authorize users, fetch user details, and log users out with Auth0 using the Authorization Code Grant Flow with PKCE. 

Before getting started, create the `/utils`, `/screens`, and `/operations` folder in your project's `/lib` directory.

### Define models and utilities

Firstly, start by defining data models and utilities for your application. To define data models, create a file `models.dart` in the `/lib/operations/` folder and add the code below:

```dart
/// Authentication object model.
class AuthModel {
  final bool isAuthenticated;
  final String authCode;

  const AuthModel({this.isAuthenticated, this.authCode});
}

/// User object model
class User {
  final String pictureUrl;
  final String name;
  final String nickname;

  const User({this.pictureUrl, this.name, this.nickname});
}
```

The application built throughout this article uses a simple layer of persistence using the `shared_preferences` package. This includes operations such as storing the user' details locally for subsequent logins, retrieving the details, and deleting them. To define persistence utils, create a file `persistence.dart` in the `/lib/utils/` folder and add the code below:

```dart
import 'dart:async';

import 'package:meta/meta.dart';
import 'package:shared_preferences/shared_preferences.dart';

Future<Null> storeRefreshToken({@required String refreshToken}) async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  await sharedPreferences.setString('refresh_token', refreshToken);
}

Future<String> getRefreshToken() async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  String refreshToken = sharedPreferences.getString('refresh_token');
  return refreshToken;
}

Future<Null> deleteRefreshToken() async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  await sharedPreferences.remove('refresh_token');
}
```

Authenticating users involves a number of utilities ranging from launching an authorization URL which is constructed in [Setup an authorization URL](#setup-an-authorization-url), to parsing the response received after authorization. To define the authentication utilities, create a file `auth_utils.dart` in the `/lib/utils/` directory, and add the code snippet below:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_custom_tabs/flutter_custom_tabs.dart';

import '../operations/models.dart';

/// To launch the authorization URL using ChromeCustomTabs
void launchURL(BuildContext context, {String url}) async {
  try {
    await launch(
      url,
      option: new CustomTabsOption(
        toolbarColor: Theme.of(context).primaryColor,
        enableDefaultShare: true,
        enableUrlBarHiding: true,
        showPageTitle: true,
        animation: new CustomTabsAnimation.slideIn(),
        extraCustomTabs: <String>[
          'org.mozilla.firefox',
          'com.microsoft.emmx',
        ],
      ),
    );
  } catch (e) {
    // An exception is thrown if browser app is not installed on Android device.
    debugPrint(e.toString());
  }
}

AuthModel parseUrlToValue(String receivedURL) {
  String value;
  bool isAuthenticated = false;
  if (!receivedURL.contains("state")) {
    if (receivedURL.contains("code")) {
      value = receivedURL.substring(receivedURL.lastIndexOf("?code=") + 6);
      isAuthenticated = true;
    } else if (receivedURL.contains("error")) {
      value = receivedURL.substring(receivedURL.lastIndexOf("?error=") + 7);
    }
  } else {
    if (receivedURL.contains("code")) {
      value = receivedURL.substring(
          receivedURL.lastIndexOf("?code=") + 6, receivedURL.indexOf("&state"));
      isAuthenticated = true;
    } else if (receivedURL.contains("error")) {
      value = receivedURL.substring(receivedURL.lastIndexOf("?error=") + 7);
    }
  }
  return AuthModel(isAuthenticated: isAuthenticated, authCode: value);
}
```

In the code snippet above, the `launchURL()` function takes in a URL and opens the URL using ChromeCustomTabs. The application built throughout this article implements social login with Google and GitHub, in addition to database login, all of which can be assessed through an authorization URL launched in the browser. However, Google has a bit of history with embedded browsers - you simply cannot make OAuth authorization requests to Google via an embedded browser [any longer](https://auth0.com/blog/google-blocks-oauth-requests-from-embedded-browsers/). This constraint leaves mobile developers with two major alternatives - using AppAuth or Chrome Custom Tabs. With AppAuth yet to have a solid Flutter implementation, the latter provides a suitable approach to making requests.

On the other hand, the `parseUrlToValue()` function accepts the URL deeplinked into the application and parses it to determine if the authorization was successful with an authorization code, or if an error was encountered. To get better insights into how the `parseUrlToValue()` function is structured to work, below are sample formats of the URLs deeplinked after successful and failed authorizations respectively:

```text
//failed authorization using social login
LOGIN_CALLBACK_URL?error=access_denied&error_description=ERROR_DESCRIPTION&state=g6Fo2SBEcDY4ZmZIVW1VS2RkVDE5dHNVemk0WUVtbW5Zci1Id6N0aWTZIDBELTRLbEZRVDJ4dnpMQVdlMzY4N05XRklYWEM0RmV6o2NpZNkgMFZ4b0tZYUtxdE96SlNRMzVQdUs0dFFMdDU3cHpPblc

//failed authorization using database login
LOGIN_CALLBACK_URL?error=access_denied&error_description=ERROR_DESCRIPTION

//successful authorization using social login
LOGIN_CALLBACK_URL?code=AUTH_CODE&state=g6Fo2SBwMkVpeHpmemtQLUZuQlUzT1c3LV8xbExvTHY0OXBxS6N0aWTZIHQ4SkRTN1l0bkswRHRTN3RwVng1T1lnTXdlMzdVd1h3o2NpZNkgMFZ4b0tZYUtxdE96SlNRMzVQdUs0dFFMdDU3cHpPblc

//successful authorization using database login
LOGIN_CALLBACK_URL?code=AUTH_CODE
```

> **Note:** Although beyond the scope of this article, in using ChromeCustomTabs, you are advised to provide some form of fallback mechanism as ChromeCustomTabs is largely dependent on the user's installation of Chrome browser.

### Setup an authorization URL

User authorization with Auth0 involves redirecting the user to an authorization URL which provides an authorization dialog. After authorization, the user is redirected to a callback URL which contains details about the user' authorization. 

An authorization URL requires parameters including `redirect_uri`, `client_id` and a few other parameters as seen [here](https://auth0.com/docs/protocols/oauth2#authorization-endpoint). Specifically, when using the Authorization Code Grant Flow with PKCE, the `code_challenge` and `code_challenge_method` parameters are required.

To get started with this, create a file `url_utils.dart` in the `/lib/utils` directory and add the code below: 

```dart
import 'dart:math';
import 'dart:convert';

import 'package:crypto/crypto.dart';

const String DOMAIN = "<AUTH0_DOMAIN>";
const String CLIENT_ID = "<CLIENT_ID>";
const String AUDIENCE = "https://$DOMAIN/api/v2/";
const String SCOPES = "openid profile email offline_access";
const String REDIRECT_URI = "myapp://logincallback";

String codeVerifier;
String codeChallenge;
```

The code snippet above declares parameters necessary for the creation of an authorization URL. The `REDIRECT_URI` variable specifies where the user will be redirected to after authorization. The value of `REDIRECT_URI` must match with one of the **Allowed Callback URLs** defined in the **Settings** tab of your Auth0 application.

On the other hand, the `SCOPES` variable specifies the kind of access required by your application. The `offline_access` option ensures that a `refresh_token` is retrieved alongside the `access_token` while exchanging the authorization code for an access token. As you will see in the latter parts of this article, the `refresh_token` eliminates the need for unnecessary repetition of the authorization process.

> **Note:** For the <AUTH0_DOMAIN> and <CLIENT_ID> placeholders, replace them with the **Domain** and **Client ID** fields found in the **Settings** tab of your Auth0 application.

#### Create a code verifier

A code verifier is a cryptographically random key used to generate a code challenge. To create a code verifier, append the snippet below to the `/lib/utils/url_utils.dart` file:

```dart
///To create code verifier
String _createVerifier() {
  var generator = Random.secure();
  var verifier = List.generate(32, (x) => generator.nextInt(256));
  return base64UrlEncode(verifier).replaceAll("=", "");
}
```

In the code above, a list is randomly generated and encoded as a Base64 string to get a code verifier.

> **Note:** The `.replaceAll("=","")` operation performed on the Base64 string removes the unrequired padding which leads to an error when verifying the code challenge.

#### Create a code challenge

When creating a code challenge to be passed to the authorization endpoint, a code verifier is required. As you'll see in the [Exchange authorization code for access tokens](#exchange-authorization-code-for-access-tokens) section, the code verifier passed must be the same with the code verifier used in generating a code challenge. To create a code challenge, append the code snippet below to the `/lib/utils/url_utils.dart` file:

```dart
///To create code challenge
String _createChallenge(String verifier) {
  var enc = utf8.encode(verifier);
  var challenge = sha256.convert(enc).bytes;
  return base64UrlEncode(challenge).replaceAll("=", "");
}
```

The code snippet above encodes the code verifier using UTF8 and converts the result to a SHA256 byte before encoding again, this time into a Base64 string to generate the code challenge.

> **Note:** The SHA256 conversion method used in generating the digest byte determines what algorithm is used in verifying the code challenge in the [Exchange authorization code for access tokens](#exchange-authorization-code-for-access-tokens) section.

#### Construct the authorization URL.

To wrap things up on setting up the authorization, add the code snippet below to the `/lib/utils/url_utils.dart` file:

```dart
///To create authorization URL
String getAuthorizationUrl() {
  codeVerifier = _createVerifier();
  codeChallenge = _createChallenge(codeVerifier);
  String authorizationUrl =
      "https://$DOMAIN/authorize?scope=$SCOPES&audience=$AUDIENCE&response_type=code&client_id=$CLIENT_ID&code_challenge=$codeChallenge&code_challenge_method=S256&redirect_uri=$REDIRECT_URI";
  return authorizationUrl;
}
```

The `getAuthorizationUrl()` function initializes the `codeVerifier` and `codeChallenge` variables. It then uses them alongside previously initialized variables such as `DOMAIN` and `SCOPES` to build the authorization URL.

### Diving deeper: User Interfaces and Operations

As seen in the [What You'll Build](#what-youll-build) section, the application contains a login screen and a profile screen, with the launched authorization URL being the linkage between both screens.

#### Create a login screen

To create a login screen, create a file named `login.dart` in the `/lib/screens/` directory, and add the code snippet below:

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:uni_links/uni_links.dart';
import '../operations/models.dart';
import '../screens/profile.dart';
import '../utils/auth_utils.dart';
import '../utils/url_utils.dart';

class Login extends StatefulWidget {
  @override
  LoginState createState() {
    return new LoginState();
  }
}

class LoginState extends State<Login> {
  String loginError;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Container(
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
              RaisedButton(
                onPressed: () {
                  launchURL(
                    context,
                    url: getAuthorizationUrl(),
                  );
                  getReceivedURL();
                },
                child: Text("Click to Login"),
              ),
              loginError != null ? Text(loginError) : Text(""),
            ],
          ),
        ),
      ),
    );
  }
}
```

The code snippet above creates a simple screen with a login button. In the `onPressed()` function of the button, two operations are performed - a call is made to the `launchURL()` function from `/lib/utils/auth_utils.dart` to launch the URL returned by `getAuthorizationUrl()` from `/lib/utils/url_utils.dart` using ChromeCustomTabs; and a `getReceivedURL()` function is called to leverage on the `uni_links` package in setting up a stream whose function is to listen to incoming URLs that will be deeplinked into the application.

Now, define the `getReceivedURL()` function by adding the code snippet below to the `LoginState` class in `/lib/screens/login.dart`:

```dart
void getReceivedURL() async {
    StreamSubscription _sub;
    String receivedLink;

    try {
      _sub = getLinksStream().listen(
        (String link) {
          receivedLink = link;

          if (receivedLink.startsWith("myapp://logincallback")) {
            AuthModel authDetails = parseUrlToValue(receivedLink);
            if (!authDetails.isAuthenticated) {
              setState(() {
                loginError = authDetails.authCode;
              });
            } else {
              Navigator.of(context).push(MaterialPageRoute(builder: (context) {
                return Profile(
                  isAuthCode: true,
                  code: authDetails.authCode,
                );
              }));
            }
          }
        },
        onDone: () {
          _sub.cancel();
        },
      );
    } catch (err) {}
}
```

The code above checks if the incoming link contains the login callback URL defined in the [Creating an Auth0 project](creating-an-auth0-project) section. It then subsequently parses the URL using the `parseUrlToValue()` function from `/lib/utils/auth_utils.dart` to determine whether to render a login error or route the user to the profile page.

Albeit `getReceivedURL()` doesn't throw any error, deeplinking into a Flutter app still requires making some modifications on the native Android and iOS platforms which are unaccounted for by the `uni_links` package. 

On the Android end, add an intent filter within the `<activity>` tag of your `/android/app/src/main/AndroidManifest.xml` as seen in the code below:

```xml
<manifest ...>
  <!-- ... other tags -->
  <application ...>
    <activity ...>
    <!-- ... other tags -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- For login callback URL -->
        <data
            android:host="logincallback"
            android:pathPattern=".*"
            android:scheme="myapp" />  
        <!-- For logout callback URL -->
        <data
            android:host="logoutcallback"
            android:pathPattern=".*"
            android:scheme="myapp" />   
    </intent-filter>
    </activity>
  </application>
</manifest>
```

On the iOS end, add an `<array>` tag within the `<key>CFBundleURLTypes</key>` tag of your `ios/Runner/Info.plist` file as seen in the code below:


<!-- This doesn't work yet. I'm unable to test. -->
```xml
<?xml ...>
<!-- ... other tags -->
<plist>
<dict>
  <!-- ... other tags -->
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleTypeRole</key>
      <string>Editor</string>
      <key>CFBundleURLName</key>
      <string>[ANY_URL_NAME]</string>
      <key>CFBundleURLSchemes</key>
      <array>
        <string>[YOUR_SCHEME]</string>
      </array>
    </dict>
  </array>
  <!-- ... other tags -->
</dict>
</plist>
```


In the above code snippets, a combination of the host and scheme separated by a colon will produce the callback URL previously defined. While this doesn't have to be the case, the callback URL must at least start with the said combination to work as expected. 

#### Create a profile screen

Before creating the profile screen, start by defining the actions that can be performed therein by creating a file `operations.dart` in the `/lib/operations` folder, and add the code below:

```dart
import 'dart:async';
import 'dart:convert';

import 'package:http/http.dart' as http;
import '../operations/models.dart';
import '../utils/persistence.dart';
import '../utils/url_utils.dart';

/// To get the access token and refresh token using authorization code
Future<String> getNewTokens(String authCode) async {
  String accessToken = "";
  String refreshToken = "";
  var url = "https://$DOMAIN/oauth/token";
  final response = await http.post(url, body: {
    "grant_type": "authorization_code",
    "client_id": CLIENT_ID,
    "code_verifier": codeVerifier,
    "code": authCode,
    "redirect_uri": REDIRECT_URI,
  });
  if (response.statusCode == 200) {
    Map jsonMap = json.decode(response.body);
    accessToken = jsonMap['access_token'];
    refreshToken = jsonMap['refresh_token'];
    await storeRefreshToken(refreshToken: refreshToken);
  } else {
    throw Exception('Failed to get access token');
  }
  return accessToken;
}

//To get access token from refresh token
Future<String> getAccessFromRefreshTokens(String refreshToken) async {
  String accessCode = "";
  var url = "https://$DOMAIN/oauth/token";
  final response = await http.post(url, body: {
    "grant_type": "refresh_token",
    "client_id": CLIENT_ID,
    "refresh_token": refreshToken,
  });
  print(response);
  if (response.statusCode == 200) {
    Map jsonMap = json.decode(response.body);
    accessCode = jsonMap['access_token'];
  } else {
    throw Exception('Failed to get access token');
  }
  return accessCode;
}

/// To get the user details from userinfo API of identity provider
Future<User> getUserDetails(String accessToken) async {
  User user;
  var url = "https://$DOMAIN/userinfo";
  final response = await http.get(
    url,
    headers: {"authorization": "Bearer $accessToken"},
  );
  print(response);
  if (response.statusCode == 200) {
    Map jsonMap = json.decode(response.body);
    var name = jsonMap['name'];
    var pictureUrl = jsonMap['picture'];
    var nickname = jsonMap['nickname'];
    user = User(
      name: name,
      pictureUrl: pictureUrl,
      nickname: nickname,
    );
  } else {
    throw Exception('Failed to get user details');
  }
  return user;
}

void logoutAction() async {
  await deleteRefreshToken();
}
```

In the code above, the `getNewTokens()` function takes in the authorization code, and exchanges it for an access token used in accessing APIs, and a refresh token which is persisted to subsequently avoid repetitive logins. The `getAccessFromRefreshTokens()` function accepts a previously persisted refresh token and uses it in getting a new access token without the user having to reauthorize the application. The `getUserDetails()` function above takes in an `accessToken` parameter and passes it as part of the authorization header when calling the `/userinfo` endpoint. It then constructs a `User` object from the retrieved response. Lastly, the `logoutAction()` function calls `deleteRefreshToken()` from `/lib/utils/persistence.dart` to delete the stored refresh token such that the user will need to login to authorize the application again.

To create the profile screen, create a file `profile.dart` in the `/lib/utils/persistence.dart` directory and add the code below:

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:uni_links/uni_links.dart';
import '../screens/login.dart';
import '../operations/models.dart';
import '../operations/operations.dart';
import '../utils/auth_utils.dart';
import '../utils/url_utils.dart' show DOMAIN;

class Profile extends StatefulWidget {
  final String code;
  final bool isAuthCode;

  const Profile({Key key, @required this.code, @required this.isAuthCode})
      : super(key: key);

  @override
  ProfileState createState() {
    return new ProfileState();
  }
}

class ProfileState extends State<Profile> {
  User user;
  String accessToken;

  @override
  void initState() {
    initAction();
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: user == null
            ? CircularProgressIndicator()
            : Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: <Widget>[
                  Container(
                    width: 150,
                    height: 150,
                    decoration: BoxDecoration(
                      border: Border.all(color: Colors.blue, width: 4.0),
                      shape: BoxShape.circle,
                      image: DecorationImage(
                        fit: BoxFit.fill,
                        image: NetworkImage(user.pictureUrl),
                      ),
                    ),
                  ),
                  SizedBox(height: 24.0),
                  Text('Fullname: ${user.name}'),
                  SizedBox(height: 24.0),
                  Text('Nickname: ${user.nickname}'),
                  SizedBox(height: 48.0),
                  RaisedButton(
                    onPressed: () {
                    },
                    child: Text("Click to Logout"),
                  ),
                ],
              ),
      ),
    );
  }

  void initAction() async {
    if (widget.isAuthCode == true) {
      await getNewTokens(widget.code).then((accessToken) {
        getUserDetails(accessToken).then((receivedUser) {
          setState(() {
            user = receivedUser;
          });
        });
      });
    } else {
      accessToken =
          await getAccessFromRefreshTokens(widget.code).then((accessToken) {
        getUserDetails(accessToken).then((receivedUser) {
          setState(() {
            user = receivedUser;
          });
        });
      });
    }
  }
}
```

In the code above, the `Profile` class accepts two parameters - a `code` parameter which is used in getting an access token, and an `isAuthCode` parameter which signifies whether `code` is a refresh token (for previously logged in users) or an authorization code (for newly logged users).  In the `initState()` method of the `ProfileState` class, a function `initAction()` is called to use the `code` parameter in getting the access token, and subsequently, use the access token to get the user details.

At this point, your application is almost complete. To see the authentication and profile fetching operations in action, modify the `home` attribute in the `build()` method of  the `MyApp` class in `main.dart` from `MyHomePage()` to `Login()`, leaving the `build()` method as seen in the code below:

```dart
@override
Widget build(BuildContext context) {
return MaterialApp(
    title: 'Flutter Demo',
    theme: ThemeData(
    primarySwatch: Colors.blue,
    ),
    home: Login(), //previously MyHomePage()
);
}
```

Then, run the app using the `flutter run` command.

### Handle logouts

Logging users out of your application involve 2 layers (Auth0 layer and application layer) and emulates a number of processes from the login operation, from launching a logout URL to redirecting users to a logout callback URL. 

As is the case with login, add a logout callback URL to the **Allowed Logout URLs** field in the [**Advanced** tab of your **Tenant** settings](https://manage.auth0.com/#/tenant/advanced). The logout callback URL should be the same as the combination derived from the second `<data>` tag within the `<intent-filter>` of your `/android/app/src/main/AndroidManifest.xml` file and the second `<array>` tag within the `<key>CFBundleURLTypes</key>` tag in the `ios/Runner/Info.plist` file as seen in the [Create a login screen](#create-a-login-screen) section.

After adding the logout callback URL to your Auth0 application, modify the `onPressed()` function of the logout button in the `ProfileState` class in `/lib/screens/profile.dart` to handle Auth0-layer logout by launching the logout url, and application-layer logout by calling `logoutAction()` method from `/operations/operations.dart`. Hence, leaving your `onPressed()` function as seen in the code snippet below:

```dart
onPressed: () {
    launchURL(
    context,
    url:
        "https://$DOMAIN/v2/logout?returnTo=myapp%3A%2F%2Flogoutcallback",
    );
    logoutAction();
    getReceivedURL();
}
```

The code above launches the logout URL with a `returnTo` parameter which expects your logout callback URL in an [encoded form](https://www.urlencoder.org/). In a similar fashion to the login screen, it also makes a call to a `getReceivedURL()` function whose purpose is to set up a stream to listen to URLs deeplinked into the application - more specifically, in this case, to listen to the logout callback URL and perform UI operations signifying that the user is logged out.

Still in the `/lib/screens/profile.dart` file, define the `getReceivedURL()` function in the `ProfileState` class by adding the code snippet below:

```dart
void getReceivedURL() async {
    StreamSubscription _sub;
    String receivedLink;

    try {
      _sub = getLinksStream().listen(
        (String link) {
          receivedLink = link;

          if (receivedLink.startsWith("myapp://logoutcallback")) {
            Navigator.of(context).push(MaterialPageRoute(builder: (context) {
              return Login();
            }));
          }
        },
        onDone: () {
          _sub.cancel();
        },
      );
    } catch (err) {}
}
```

### Additional: Checking for previously logged-in users

The process of checking if the user is already logged in involves checking if a refresh token exists locally on shared preferences. If it exists, the application renders the profile screen, sending the refresh token as the `code` parameter. Otherwise, it renders the login screen.

To do this, modify the `MyHomePage` class in `/main.dart` and set the `home` attribute back to `MyHomePage()`, leaving the `main.dart` file as seen in the code snippet below:

```dart
import 'package:auth0/screens/login.dart';
import 'package:auth0/screens/profile.dart';
import 'package:auth0/utils/persistence.dart';
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(),
    );
  }
}

class MyHomePage extends StatefulWidget {
  @override
  MyHomePageState createState() {
    return new MyHomePageState();
  }
}

class MyHomePageState extends State<MyHomePage> {
  bool isLoggedIn = false;
  String refreshToken;

  @override
  void initState() {
    initAction();
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return refreshToken == null
        ? Login()
        : Profile(
            isAuthCode: false,
            code: refreshToken,
          );
  }

  void initAction() async {
    await getRefreshToken().then((receivedRefreshToken) {
      if (receivedRefreshToken != null && receivedRefreshToken.isNotEmpty) {
        setState(() {
          refreshToken = receivedRefreshToken;
        });
      }
    });
  }
}
```

In the code snippet above, the `MyHomePage` class calls an `initAction()` function in its `initState()` method. The `initAction()` function calls `getRefreshToken()` from `/lib/utils/persistence.dart` to check if a refresh token exists and renders the UI using `setState()` based on the outcome.

After that, you're done securing your Flutter application with Auth0! You can now run your app using the `flutter run` command.

## Conclusion and Recommendations

In this post, you learned how to secure a Flutter application with Auth0 using the authorization code grant flow with PKCE.

Although Dart's default HTTP client library was used in making network requests throughout this article, the [Dio package](https://pub.dartlang.org/packages/dio) is a robust alternative, providing you with features such as interceptors and the ability to handle request timeouts.

Additionally, In building applications for production, it is generally considered unsafe to plainly persist sensitive values such as access or refresh tokens with shared preferences. A minimal security step is to at least obfuscate the token with an algorithm before persisting.

Finally, while its usage is limited to fetching user details in the course of this article, the access token should be kept alive throughout the lifecycle of large applications where it's needed to make frequent API calls.

I do hope that you enjoyed this tutorial. Happy hacking!
