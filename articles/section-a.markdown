### Create a user interface

As seen in the [What You'll Build]() section, the application contains a login screen and a profile screen, with the launched authorization URL being the linkage between both screens.

#### Create a login screen

To create a login screen, add the code below to the `/lib/screeens/login.dart` file:

```dart
import 'package:flutter/material.dart';

class Login extends StatelessWidget {
  final String loginError;

  const Login({Key key, this.loginError}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Welcome"),
      ),
      body: Container(
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
              RaisedButton(
                onPressed: () {},
                child: Text("Click to Login"),
              ),
              Text(loginError ?? ""),
            ],
          ),
        ),
      ),
    );
  }
}
```

The code above creates a login screen that takes in a string    `loginError` as an optional parameter. As you will see later in the article, the `loginError` parameter is displayed when an error is encountered in the authorization process.


#### Create a profile screen

To create a profile screen, add the code snippet below to the `/lib/screeens/profile.dart` file:

```dart
import 'package:flutter/material.dart';

class Profile extends StatelessWidget {
  final String code;

  const Profile({Key key, this.code}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("User Profile"),
      ),
      body: Center(
        child: Column(
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
                    "https://image.flaticon.com/icons/svg/78/78373.svg", //dummy image URL
                  ),
                ),
              ),
            ),
            SizedBox(height: 24.0),
            Text(
              'Name: Dummy Name',
              style: TextStyle(fontWeight: FontWeight.w600, fontSize: 18.0),
            ),
            SizedBox(height: 24.0),
            Text(
              'Nickname: Dummy Nickname',
              style: TextStyle(fontWeight: FontWeight.w600, fontSize: 18.0),
            ),
            SizedBox(height: 48.0),
            RaisedButton(
              onPressed: () {},
              child: Text("Click to Logout"),
            ),
          ],
        ),
      ),
    );
  }
}
```

In a similar fashion to the login screen, the snippet above creates a dummy profile screen that takes in an optional parameter, `code`, which is the authorization code used to retrieve access token which is subsequently used in accessing the user's profile details.

Finally, clear the boilerplate code in your `/lib/main.dart` file excluding the `MyApp` class and `main()` method. Then, set `Login()` as the value for the home attribute. This should leave you with the code below in the `/lib/main.dart` file:

```dart
import 'package:flutter/material.dart';
import './screens/login.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: Login(),
    );
  }
}
```

In the absence of the authorization process, add the code below to the `onPressed` function of the login button in the `/lib/screens/login.dart` to establish a navigation flow between the login screen and profile screen.

```dart
Navigator.push(context, MaterialPageRoute(builder: (context) => Profile()));
```

In the `onPressed` function of the logout button in `/lib/screens/profile.dart` file, add the following snippet:

```dart
Navigator.pop(context);
```

Add necessary imports and run your app using the `flutter run` command.