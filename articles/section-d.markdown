### Access APIs after authorization

In this section, you will learn how to use the user' authorization code to get an access token which is subsequently used to fetch the user's profile, all by leveraging the ReBLOC structure.

##### Architect your profile screen

Firstly, start by defining your profile state which accounts for values with possibility of changing in the profile screen. To do this, add the code snippet below to the `/lib/rebloc/state/profile_state.dart` file:

```dart
import '../models/model.dart';

class ProfileState {
  final User user;

  const ProfileState({
    this.user,
  });

  ProfileState.initialState()
      : user = User(name: "", nickname: "", pictureUrl: "");

  ProfileState copyWith({User user}) {
    return ProfileState(
      user: user ?? this.user,
    );
  }

  @override
  String toString() {
    return "ProfileState_$user";
  }
}
```

Then, add the profile state to the application state by modifying the `/lib/rebloc/state/app_state.dart` file as seen in the code below:

```dart 
import '../models/models.dart';

class AppState {
  final AuthState authState;
  final ProfileState profileState;

  const AppState({this.authState, this.profileState});

  AppState.initialState()
      : authState = AuthState.initial(),
        profileState = ProfileState.initialState();

  AppState copyWith({AuthState authState, ProfileState profileState}) {
    return AppState(
      authState: authState ?? this.authState,
      profileState: profileState ?? this.profileState,
    );
  }

  @override
  String toString() => "AppState_$authState$profileState";
}
```

After appending changes to the state classes, create a `ProfileBloc` class in the `/lib/rebloc/bloc/profile_bloc.dart` file by extending the `SimpleBloc` class as seen in the [Architect your login screen]() section. Then add the `ProfileBloc()` object to the list of blocs in `/lib/rebloc/store.dart`. 

Finally, modify the `build()` method in your `/lib/screens/profile.dart` file as seen in the code below:

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text("User Profile"),
    ),
    body: Center(
      child: FirstBuildDispatcher<AppState>(
        action: isAuthCode == true
            ? GetTokensFromAuthAction(authCode: code)
            : GetAccessFromRefreshTokenAction(refreshToken: code),
        child: ViewModelSubscriber<AppState, ProfileState>(
          converter: (state) => state.profileState,
          builder: (context, dispatch, viewModel) {
            if (viewModel.user.name == "") {
              return CircularProgressIndicator();
            }
            return Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: <Widget>[
                Container(
                  width: 150,
                  height: 150,
                  decoration: new BoxDecoration(
                    border: Border.all(color: Colors.blue, width: 4.0),
                    shape: BoxShape.circle,
                    image: new DecorationImage(
                      fit: BoxFit.fill,
                      image: new NetworkImage(
                        viewModel.user.pictureUrl,
                      ),
                    ),
                  ),
                ),
                SizedBox(height: 24.0),
                Text(
                  'Name: ${viewModel.user.name}',
                  style:
                      TextStyle(fontWeight: FontWeight.w600, fontSize: 18.0),
                ),
                SizedBox(height: 24.0),
                Text(
                  'Nickname: ${viewModel.user.nickname}',
                  style:
                      TextStyle(fontWeight: FontWeight.w600, fontSize: 18.0),
                ),
                SizedBox(height: 48.0),
                RaisedButton(
                  onPressed: () {},
                  child: Text("Click to Logout"),
                ),
              ],
            );
          },
        ),
      ),
    ),
  );
}
```

In the code above, on initial build of the profile screen, the application checks if the code received as a paramter to the profile screen is an authorization code or a refresh token, and dispatches necessary actions based on the check. Also in the `build()` method, the application checks if user details have been retrieved. If retrieved, the details will be displayed from the view model, otherwise a loading indicator is rendered.

#### Exchange authorization code for access tokens

As seen in the diagram in the [OAuth 2.0 Flow and Mobile Applications]() section, when securing apps with Auth0 using the Authorization Code Grant Flow with PKCE, a request is made to get the access token after the user authorizes your application.

As seen in the [documentations](https://auth0.com/docs/api-auth/tutorials/authorization-code-grant-pkce#4-exchange-the-authorization-code-for-an-access-token), the request is a simple POST request that takes in `grant_type`, `client_id`, `code_verifier`, `code` and `redirect_uri` parameters, then returns the response containing the `access_token` and `refresh_token` parameters.

To make this request, append the code snippet below to the `/lib/rebloc/bloc/profile_bloc.dart` file, and add necessary imports:

```dart
/// To get the access token and refresh token using authorization code
Future<String> getAccessFromAuthCode(String authCode) async {
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
```

The function above makes a `POST` request to the token endpoint, gets the access token and refresh token, and persists the refresh token for subsequent logins using the `storeRefreshTokens()` function.

To create the `storeRefreshTokens()` function, add the following code to your `/lib/utils/persistence.dart` file:

```dart
import 'dart:async';

import 'package:meta/meta.dart';
import 'package:shared_preferences/shared_preferences.dart';

Future<Null> storeRefreshToken({@required String refreshToken}) async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  await sharedPreferences.setString('refresh_token', refreshToken);
}
```

Then, import `/lib/utils/persistence.dart` in `/lib/rebloc/bloc/profile_bloc.dart`.

> **Note:** In building applications for production, it is considered unsafe to persist critical values such as access or refresh tokens with shared preferences. A minimal security step is to at least obfuscate the token with an algorithm before persisting.

#### Fetch user profile with access token

After getting the access token, it can be used to make API calls. To get the user details using the access token, append the code below to the `/lib/rebloc/bloc/profile_bloc.dart` file

```dart
/// To get the user details from userinfo API of identity provider
Future<User> getUserDetails(String accessToken) async {
  User user;
  var url = "https://$DOMAIN/userinfo";
  final response = await http.get(
    url,
    headers: {"authorization": "Bearer $accessToken"},
  );
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
```

The `getUserDetails()` function above takes in an accessToken parameter and passes it as part of the authorization header when calling the `/userinfo` endpoint. It then constructs a `User` object from the retrieved response.

To call the `getUserDetails()` and `getAccessFromAuthCode()` actions, modify the `middleware()` and `reducer()` functions in the `/lib/rebloc/bloc/profile_bloc.dart` file as seen in the code below:

```dart
@override
FutureOr<Action> middleware(
    DispatchFunction dispatch,
    AppState state,
    Action action,
) async {
    Function showUserDetails = (accessToken) async {
      await getUserDetails(accessToken).then((user) {
        dispatch(DisplayDetailsAction(user: user));
      });
    };

    if (action is GetTokensFromAuthAction) {
      await getAccessFromAuthCode(action.authCode).then(showUserDetails);
    }
    return super.middleware(dispatch, state, action);
}

@override
AppState reducer(AppState state, Action action) {
    final _profileState = state.profileState;
    if (action is DisplayDetailsAction) {
      return state.copyWith(
          profileState: _profileState.copyWith(user: action.user));
    }
    return state;
}
```

The `middleware()` function in the code above retrieves the access token using the authorization code whenever `GetTokensFromAuthAction` is dispatched. It then sends the access token to a `showUserDetails()` function which gets the user' details and displays the retrieved details by dispatching the `DisplayDetailsAction` action. Upon triggering the `DisplayDetailsAction`, the application state is update with the user' details in the `reducer()` function.

To ease the process of wiring the login and profile screens together, create `MainBloc` and `MainState` classes. The function of the `MainBloc` class is to perform initialzation actions such as setting the in checking if the user was previously logged in (as you'll see in the [Persist refresh tokens for subsequent access.]() section). On the other hand, the function of the `MainState` class is to render the profile or login screen based on whether or not the user is logged in.

To create a `MainBloc` class, add the code snippet below to the `/lib/rebloc/bloc/main_bloc.dart` file:

```dart
import '../models/models.dart';

class MainState {
  final ScreenType screenType;
  final String refreshToken;
  const MainState({this.screenType, this.refreshToken});

  MainState.initialState()
      : screenType = ScreenType.loggedOut,
        refreshToken = "";

  MainState copyWith({
    ScreenType newScreenType,
    String refreshToken,
  }) {
    return MainState(
      screenType: newScreenType,
      refreshToken: refreshToken,
    );
  }

  @override
  String toString() => "MainState_$screenType";
}
```

The code above creates the `MainState` class with two parameters - a `ScreenType` object that defines whether the user is logged in or not, and a refreshToken for instances where the user was previously logged in.

To create a `MainBloc`, add the code snippet below to the `/lib/rebloc/bloc/main_bloc.dart` file:

```dart
import 'dart:async';

import '../actions/actions.dart';
import '../models/models.dart';
import '../state/app_state.dart';
import 'package:rebloc/rebloc.dart';

class MainBloc extends SimpleBloc<AppState> {
  @override
  FutureOr<Action> middleware(
    DispatchFunction dispatcher,
    AppState state,
    Action action,
  ) async {
    return super.middleware(dispatcher, state, action);
  }

  @override
  AppState reducer(AppState state, Action action) {
    final _mainState = state.mainState;
    if (action is OnInitAction) {
      return state.copyWith(
        mainState: _mainState.copyWith(newScreenType: ScreenType.loggedOut),
      );
    }
    return state;
  }
}
```

In the `reducer()` function in the code above, the main state is updated to reflect that the user is initially logged out such that a login screen is displayed to the user.

Add the `MainBloc` and `MainState` to the list of blocs and app state in the `/lib/rebloc/store.dart` and `/lib/rebloc/states/app_state.dart` files respectively.

Finally, wire the existing blocs and states together by modifying the `/lib/main.dart` file to be as shown in the code below.

```dart
void main() {
  runApp(StoreProvider<AppState>(
    store: appStore,
    child: MyApp(),
  ));
}

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
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

class MyHomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return FirstBuildDispatcher<AppState>(
      action: OnInitAction(),
      child: ViewModelSubscriber<AppState, AppState>(
        converter: (state) => state,
        builder: (context, dispatcher, viewModel) {
          bool isAuthenticated = viewModel.authState.authModel.isAuthenticated;
          String code = viewModel.authState.authModel.authCode;
          if (isAuthenticated) {
            return Profile(
              code: code,
              isAuthCode: true,
            );
          } else if (isAuthenticated == false) {
            return Login(
              loginError: code,
            );
          }
          return Login();
        },
      ),
    );
  }
}
```

The code above replaces `Login` with `MyHomePage` as the base class of the application. The `MyHomePage` class triggers an `OnInitAction` using the `FirstBuildDispatcher` from the ReBLOC package, it then checks if the user is authenticated. If authenticated, it returns the profile by passing the authorization code as a parameter to enable retrieval of access tokens from the profile screen. If not authenticated, it implies that the user failed to authorize the application, and as a result, the login screen is re-rendered with the login error sent as a parameter to be displayed. Otherwise, it just renders the login screen.

#### Using refresh tokens for subsequent access
<!-- Validate the bad practice shii finally -->

For most modern day applications, it is widely considered a bad practice to make users have to login anew everytime they open an application. The refresh token retrieved as part of the response allows you to emulate the practice of preserving user' login until the user decides to voluntarily logout in the application being developed in this article. The refresh token which is persisted using `SharedPreferences` as seen in the [Exchange authorization code for access tokens]() section can be subsequently retrieved and used in getting a new access token which is in turn used in making API calls.

To use refresh tokens to retrieve access tokens, make the following modifications to the files below:

<!-- /token or /oauth/token endpoint? -->
In the `/lib/rebloc/bloc/profile_bloc.dart` file, append the code below to make a request to the `/token` endpoint with a `refresh_token` parameter sent:

```dart
//To get access token from refresh token
Future<String> getAccessFromRefreshTokens(String refreshToken) async {
  String accessCode = "";
  var url = "https://$DOMAIN/oauth/token";
  final response = await http.post(url, body: {
    "grant_type": "refresh_token",
    "client_id": CLIENT_ID,
    "refresh_token": refreshToken,
  });
  
  if (response.statusCode == 200) {
    Map jsonMap = json.decode(response.body);
    accessCode = jsonMap['access_token'];
  } else {
    throw Exception('Failed to get access token');
  }
  return accessCode;
}
```

Still in the `/lib/rebloc/bloc/profile_bloc.dart` file, modify the `middleware()` function to be as shown in the code snippet below:

```dart
FutureOr<Action> middleware(
    DispatchFunction dispatch,
    AppState state,
    Action action,
) async {
    Function showUserDetails = (accessToken) async {
      await getUserDetails(accessToken).then((user) {
        dispatch(DisplayDetailsAction(user: user));
      });
    };

    if (action is GetTokensFromAuthAction) {
      await getAccessFromAuthCode(action.authCode).then(showUserDetails);
    } else if (action is GetAccessFromRefreshTokenAction) {
      await getAccessFromRefreshTokens(action.refreshToken)
          .then(showUserDetails);
    }

    return super.middleware(dispatch, state, action);
}
```

The modification ensures that the `getAccessFromRefreshTokens()` function is called when trying to get the access token using a refresh token instead of an authorization code.

Then, update the `middleware()` and `reducer()` functions in your `/lib/rebloc/bloc/main_bloc.dart` file to account for instances where the user has been previously logged in, as seen in the code below:

```dart
class MainBloc extends SimpleBloc<AppState> {
  @override
  FutureOr<Action> middleware(
    DispatchFunction dispatcher,
    AppState state,
    Action action,
  ) async {
    if (action is OnInitAction) {
      await getRefreshToken().then((refreshToken) {
        if (refreshToken != null && refreshToken.isNotEmpty) {
          dispatcher(SilentLoginAction(refreshToken: refreshToken));
        }
      });
    }
    return super.middleware(dispatcher, state, action);
  }

  @override
  AppState reducer(AppState state, Action action) {
    final _mainState = state.mainState;
    if (action is OnInitAction) {
      return state.copyWith(
        mainState: _mainState.copyWith(newScreenType: ScreenType.loggedOut),
      );
    }
    if (action is SilentLoginAction) {
      return state.copyWith(
        mainState: _mainState.copyWith(
            newScreenType: ScreenType.loggedIn,
            refreshToken: action.refreshToken),
      );
    }
    return state;
  }
}
```

In the code above, when the `OnInitAction` is dispatched, the presence of a refresh token is checked for in the middleware. If a refresh token is found, the `SilentLoginAction` is dispatched to update the status of the application screen as `ScreenType.loggedIn`, thus eliminating the need for a login prompt. Otherwise, the login screen is rendered.

To create the `getRefreshToken()` function, append the following code to your `/lib/utils/persistence.dart` file:

```dart
Future<String> getRefreshToken() async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  String refreshToken = sharedPreferences.getString('refresh_token');
  return refreshToken;
}
```

Then, import `/lib/utils/persistence.dart` in `/lib/rebloc/bloc/main_bloc.dart`.

Finally, modify the `MyHomePage` class in the `/lib/main.dart` to reflect the code below in a bid to take logged in users into consideration when determining whether to render the login screen or the profile screen:

```dart
class MyHomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return FirstBuildDispatcher<AppState>(
      action: OnInitAction(),
      child: ViewModelSubscriber<AppState, AppState>(
        converter: (state) => state,
        builder: (context, dispatcher, viewModel) {
          if (viewModel.mainState.screenType == ScreenType.loggedIn) {
              String refreshToken = viewModel.mainState.refreshToken;
              return Profile(
                code: refreshToken,
                isAuthCode: false, 
              );
            } else {
              bool isAuthenticated =
                  viewModel.authState.authModel.isAuthenticated;
              String code = viewModel.authState.authModel.authCode;
              if (isAuthenticated) {
                return Profile(
                  code: code,
                  isAuthCode: true,
                );
              } else if (isAuthenticated == false) {
                return Login(
                  loginError: code,
                );
              }
            }
            return Login();
        },
      ),
    );
  }
}
```

The code above dispatches an `OnInitAction()` which was updated to check for the existence of a refresh token. If a refresh token exists, it returns the profile screen by sending the refresh token as the code parameter. It also assigns a `false` value to the `isAuthCode` parameter, signifying that the login screen should treat the received code as a refresh token, and not as an authorization code.

