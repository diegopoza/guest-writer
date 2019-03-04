### Get the user's authorization

As highlighted in the [What You'll Build]() section, the application built throughout this article implements social login with Google and GitHub, in addition to database login. 

However, Google has a bit of history with embedded browsers - you simply cannot make OAuth authorization requests to Google via an embedded browser [any longer](https://auth0.com/blog/google-blocks-oauth-requests-from-embedded-browsers/). This constraint leaves mobile developers wth two major alternatives - using AppAuth or Chrome Custom Tabs. With AppAuth yet to have a solid Flutter implementation, the latter provides a suitable approach to making requests.

> **Note:** Although beyond the scope of this article, in using ChromeCustomTabs, you are advised to provide some form of fallback mechanism as ChromeCustomTabs is largely dependent on the user's installation of Chrome browser.

#### Launch the authorization URL within your app

To launch the authorization URL, start by creating a `launchURL()` function that takes in a URL and opens the URL using ChromeCustomTabs. To do this, add the following code to your `/lib/utils/auth_utils.dart` file:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_custom_tabs/flutter_custom_tabs.dart';

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
```

To use the `launchURL()` function, replace the code in the `onPressed()` function of the login button in `/lib/screens/login.dart` with the code below and add necessary imports:

```dart
launchURL(context, url: getAuthorizationUrl());
```

The snippet above launches the constructed authorization URL from `/lib/utils/url_utils.dart` using ChromeCustomTabs.

#### Retrieve the authorization code

In this section, you will setup your app for deep-linking from an authorization URL, listen to the received link via a stream, and parse the received link to useful data. You will also learn how to use ReBLOC architecture to achieve the aforementioned processes.

##### Architect your login screen

In using ReBLOC, **actions** represent operations that you want to perform, **states** represent possible appearances of your app's screen, **blocs** contain logic code, and a **store** serves as the repository for the states and operations of an application.

Firstly, define your data models by adding the code snippet below to the `lib/rebloc/models.dart` file:

```dart
/// Authentication object model.
class AuthModel {
  final bool isAuthenticated;
  final String authCode;

  const AuthModel({this.isAuthenticated, this.authCode});
}

enum ScreenType {
  loggedOut,
  loggedIn,
}

/// User object model
class User {
  final String pictureUrl;
  final String name;
  final String nickname;

  const User({this.pictureUrl, this.name, this.nickname});
}
```

After defining your models, define all actions performable in the app by adding the code below to the `/lib/rebloc/actions.dart` file:

```dart
import 'package:rebloc/rebloc.dart';
import '../models.dart';

// MAIN ACTIONS
/// To initialize app and check if user
/// was previously logged in
class OnInitAction extends Action {}

// AUTH ACTIONS
/// To get the URL received during deeplinking
class GetReceivedURLAction extends Action {}

/// To log user in with [authCode]
class LoginAction extends Action {
  final String authCode;
  final bool isAuthenticated;

  const LoginAction({this.authCode, this.isAuthenticated});
}

class LogoutAction extends Action {}

/// To perform silent login without authorization
/// after initial login
class SilentLoginAction extends Action {
  final String refreshToken;

  const SilentLoginAction({this.refreshToken});
}

// PROFILE ACTIONS
///  To display user details
class DisplayDetailsAction extends Action {
  final User user;

  const DisplayDetailsAction({this.user});
}

/// To get access tokens during initial login
class GetTokensFromAuthAction extends Action {
  final String authCode;

  const GetTokensFromAuthAction({this.authCode});
}

/// To get access token during subsequent logins
class GetAccessFromRefreshTokenAction extends Action {
  final String refreshToken;

  const GetAccessFromRefreshTokenAction({this.refreshToken});
}
```

After specifying your app actions, define the state of your login screen by adding the code snippet below to `/lib/rebloc/state/auth_state.dart`:

```dart
import '../models.dart';

class AuthState {
  final AuthModel authModel;

  const AuthState({this.authModel});

  AuthState.initial()
      : authModel = AuthModel(
          isAuthenticated: false,
          authCode: "",
        );

  AuthState copyWith({AuthModel authModel}) {
    return AuthState(
      authModel: authModel ?? this.authModel,
    );
  }

  @override
  String toString() {
    return "AuthState_$authModel";
  }
}
```

Then, create the application's whole state which holds all other states in the application by adding the snippet below to your `/lib/rebloc/state/app_state.dart` file:

```dart
import '../state/auth_state.dart';

class AppState {
  final AuthState authState;

  const AppState({
    this.authState,
  });

  AppState.initialState() : authState = AuthState.initial();

  AppState copyWith({AuthState authState}) {
    return AppState(
      authState: authState ?? this.authState,
    );
  }

  @override
  String toString() => "AppState_$authState";
}
```

Later in the article, the `AppState` class will be modified to hold newly added states.

In ReBLOC, making the states and blocs accessible to all parts of the app is achieved by passing the store down the widget tree using a `StoreProvider` widget.

Before creating your store, create a class `LoginBloc` in the file `/lib/rebloc/bloc/auth_bloc.dart` by extending the `SimpleBloc` class provided by the `rebloc` package, and override the `middleware()` and `reducer()` functions as seen in the snippet below:

```dart
import 'dart:async';

import '../state/app_state.dart';
import 'package:rebloc/rebloc.dart';

class AuthBloc extends SimpleBloc<AppState> {
  @override
  FutureOr<Action> middleware(
    DispatchFunction dispatch,
    AppState state,
    Action action,
  ) async {
    return super.middleware(dispatch, state, action);
  }

  @override
  AppState reducer(AppState state, Action action) {
    return state;
  }
}
```

Middlewares in ReBLOC are the logical operations performed after an action has been triggered, while reducers are [pure functions](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-a-pure-function-d1c076bec976) that respond to actions that are intended to change the application's appearance.

Finally, create a store by adding the code below to the `/lib/rebloc/store.dart` file:

```dart
import './bloc/auth_bloc.dart';
import './state/app_state.dart';
import 'package:rebloc/rebloc.dart';

final appStore = Store<AppState>(
  initialState: AppState.initialState(),
  blocs: [
    AuthBloc(),
  ],
);
```

To pass the store down the widget tree, wrap the `MyApp` widget in the `main()` function of the `/lib/main.dart` file with a `StoreProvider` widget as seen in the code below, and add the necessary imports:

```dart
void main() {
  runApp(StoreProvider<AppState>(
    store: appStore,
    child: MyApp(),
  ));
}
```

##### Set up your app for deeplinking

Deeplinking in Flutter requires having to do some work on the native Android and iOS platforms. 

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
        <data
            android:host="logincallback"
            android:pathPattern=".*"
            android:scheme="myapp" />    
    </intent-filter>
    </activity>
  </application>
</manifest>
```

<!-- Then, add the following .... on the iOS end. (not tested yet. Leave empty for now) -->

In the above code snippets, a combination of the host and scheme seperated by a colon will produce the callback URL previously defined. While this doesn't have to be the case, the callback URL must at least start with the said combination to work as expected.

#### Listen to the received URL

To listen to the URL received during deeplinking, modify the `middleware()` and `reducer()` functions in the `/lib/rebloc/bloc/auth_bloc.dart` to setup a listening stream and update the app state respectively, as seen in the code snippet below:

```dart
@override
FutureOr<Action> middleware(
  DispatchFunction dispatch,
  AppState state,
  Action action,
) async {
  if (action is GetReceivedURLAction) {
    StreamSubscription _sub;
    String receivedLink;

    try {
      receivedLink = await getInitialLink();
      _sub = getLinksStream().listen(
        (String link) {
          receivedLink = link;

          if (receivedLink.startsWith("myapp://logincallback")) {
            AuthModel authDetails = parseUrlToValue(receivedLink);
            dispatch(LoginAction(
                authCode: authDetails.authCode,
                isAuthenticated: authDetails.isAuthenticated));
          }
        },
        onError: (err) {
          receivedLink = err;
        },
        onDone: () {
          _sub.cancel();
        },
      );
    } catch (err) {}
  }

  return super.middleware(dispatch, state, action);
}

@override
AppState reducer(AppState state, Action action) {
    final _authState = state.authState;
    if (action is LoginAction) {
        return state.copyWith(
            authState: _authState.copyWith(
                authModel: AuthModel(
                    authCode: action.authCode,
                    isAuthenticated: action.isAuthenticated),
            ),
        );
    }

    return state;
}
```

The code above sets up a subscription that listens to the `getLinksStream()` stream from the `uni_links` package. When the link changes after authorization (as expected), the method checks if the `receivedLink` parameter starts with your specified callback URL (`myapp://logincallback` in this case). It then parses the received link to check for whether an error or a code was returned by calling the `parseUrlToValue()` function on the link before triggering the `LoginAction`. Upon triggering the action, the application state is updated in the `reducer()` function.

Create the `parseUrlToValue()` function by appending the code below to your `/lib/rebloc/bloc/auth_bloc.dart` file, and add the necessary imports:

```dart
AuthModel parseUrlToValue(String receivedURL) {
  String value;
  bool isAuthenticated = false;

  /// checks if url contains the 'state' word. Logins with database
  /// connection don't return 'state' as part of the callback
  if (!receivedURL.contains("state")) { 
    if (receivedURL.contains("code")) {
        //assigns the auth code as the value
      value = receivedURL.substring(receivedURL.lastIndexOf("?code=") + 6);
      isAuthenticated = true;
    } else if (receivedURL.contains("error")) {
        //assigns the error message as the value
      value = receivedURL.substring(receivedURL.lastIndexOf("?error=") + 7);
    }
  } else {
    if (receivedURL.contains("code")) {
        //assigns the auth code as the value
      value = receivedURL.substring(
          receivedURL.lastIndexOf("?code=") + 6, receivedURL.indexOf("&state"));
      isAuthenticated = true;
    } else if (receivedURL.contains("error")) {
        //assigns the error message as the value
      value = receivedURL.substring(receivedURL.lastIndexOf("?error=") + 7);
    }
  }
  return AuthModel(isAuthenticated: isAuthenticated, authCode: value);
}
```

The last step in retrieving the user's authorization is to dispatch/trigger the action to start listening to the stream immediately the authorization URL is launched. To do this, wrap the scaffold in `/lib/screens/login.dart` with a `DispatchSubscriber` widget, and modify the `onPressed()` function of the login button as shown in the code below:

```dart
@override
Widget build(BuildContext context) {
  return DispatchSubscriber<AppState>(
    builder: (context, dispatch) {
      return Scaffold(
        appBar: AppBar(title: Text("Welcome")),
        body: Container(
          //widget tree
            RaisedButton(
              onPressed: () {
                launchURL(
                  context,
                  url: getAuthorizationUrl(),
                );
                dispatch(GetReceivedURLAction());
              },
              child: Text("Click to Login"),
            ),
        //widget tree
        ),
      );
    },
  );
}
```

A `DispatchSubscriber` widget in ReBLOC is used when you want to dispatch an action without the need for a view model. For instances that require an action dispatcher and a view model, the `ViewModelSubscriber` widget is used instead.

In the `onPressed()` function in the code above, the `dispatch()` function triggers the `GetReceivedURLAction` which in turn starts the subscription stream to detect new URLs deeplinked into the application.

> **Note:** If you're interested in seeing the values returned at any point within your app, you can log the values with print statements.