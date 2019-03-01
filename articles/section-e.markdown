### Handle logouts

In this section, you will learn how to log users out of your application - both out of Auth0 and your client application.

#### Handle Auth0-layer logouts

Logging users out of your Auth0 application emulates a number of processes from login operation, from launching a logout URL to redirecting users to a logout callback URL. 

As is the case with login, add a logout callback URL to the **Allowed Logout URLs** field in the [**Advanced** tab of your **tenant** settings](https://manage.auth0.com/#/tenant/advanced).

After setting a logout callback URL in your Auth0 application, you need to modify the `/manifest` and `/plist` files as seen in the [Login]() section to allow deeplinking of the logout callback URL into your application.

To do this on the Android end, add a new data tag within the `<intent-filter>` tag of your `/android/app/src/main/AndroidManifest.xml` file as seen in the code below:

```xml
<manifest ...>
  <!-- ... other tags -->
  <application ...>
    <activity ...>
    <!-- ... other tags -->
    <intent-filter android:autoVerify="true">
        <!-- other tags -->
        <!-- login callback url data tag -->
        <data
            android:host="logoutcallback"
            android:pathPattern=".*"
            android:scheme="myapp" /> 
    </intent-filter>
    </activity>
  </application>
</manifest>
```

<!-- Then, add the following .... on the iOS end. (not tested yet. Leave empty for now) -->

To launch the logout URL, modify the `onPressed()` function of the logout button in `/lib/screens/profile.dart` as seen in the code below:

```dart
onPressed: () {
    launchURL(
        context,
        url: "https://$DOMAIN/v2/logout?returnTo=myapp%3A%2F%2Flogoutcallback",
    );
}
```

The code above launches the logout URL witha `returnTo` parameter which expects your logout callback URL in an [encoded form](https://www.urlencoder.org/). Although logged out of Auth0, the profile screen is rerendered. In the next section, [Handling application-layer logouts](), you will learn how to fix this.

#### Handling application-layer logouts

Logging users out of the application layer involves deleting the persisted refresh token.

To do this, start by appending the `deleteRefreshToken()` function below to the `/lib/utils/persistence.dart` file:

```dart
Future<Null> deleteRefreshToken() async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  await sharedPreferences.remove('refresh_token');
}
```

Then, append the code snippet below in the `onPressed()` function of the logout button in `/lib/screens/profile.dart` file:

```dart
dispatch(GetReceivedURLAction());
```

After modifying the `middleware()` function in `/lib/rebloc/bloc/auth_bloc.dart` to account for instances where the deeplink received in the app is the logout callback URL and dispatch an appropriate action as seen in the code below:

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
                    //parse and dispatch login
                } else if (receivedLink.startsWith("myapp://logoutcallback")) {
                    dispatch(LogoutAction());
                }
            },
            onError: (err) {
                receivedLink = err;
            },
            onDone: () {
            _sub.cancel();
            },
        );
        } on PlatformException {}
    }

    if (action is LogoutAction) {
        await deleteRefreshToken();
    }

    return super.middleware(dispatch, state, action);
}
```

Still in the `/lib/rebloc/bloc/auth_bloc.dart` file, add the code below to the `reducer()` function to finish the login process:

```dart
if (action is LogoutAction) {
    return AppState.initialState();
}
```

The code above resets the application to its initial state - with user logged out, refresh token absent, and a login prompt rendered to the user.

After that, you're done securing your Flutter application with Auth0! You can now run your app using the `flutter run` command.







