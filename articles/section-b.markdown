### Setup an authorization URL

User authorization with Auth0 involves redirecting the user to an authorization URL which provides an authorization dialog. After authorization, the user is redirected to a callback URL which contains details about the user' authorization. An authorization URL requires parameters including `redirect_uri`, `client_id` and a few other parameters as seen [here](https://auth0.com/docs/protocols/oauth2#authorization-endpoint). Specifically, when authorizing using the Authorization Code Grant Flow with PKCE, the `code_challenge` and `code_challenge_method` parameters are required.

To get started with this, create a file named `url_utils.dart` in the `/lib/utils` directory and add the code below: 

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

The code snippet above declares parameters necessary for the creation of an Authorization URL. The `REDIRECT_URI` variable specifies where the user will be redirected to after authorization. The value of `REDIRECT_URI` must match with one of the **Allowed Callback URLs** defined in the **Settings** tab of your Auth0 application.

On the other hand, the `SCOPES` variable specifies the kind of access required by your application. The `offline_access` option ensures that a `refresh_token` is retrieved alongside the `access_token` while exchanging the authorization code for access token. As you will see in latter parts of this article, the `refresh_token` eliminates the need for unneccessary repetition of the authorization process.

> **Note:** For the <AUTH0_DOMAIN> and <CLIENT_ID> placeholders, replace them with the **Domain** and **Client ID** fields that can be found in the **Settings** tab of your application.

#### Create a code verifier

A code verifier is a cryptographically random key used to generate a code challenge. To create a code verifier, append the snippet below to the `lib/utils/url_utils.dart` file:

```dart
///To create code verifier
String _createVerifier() {
  var generator = Random.secure();
  var verifier = List.generate(32, (x) => generator.nextInt(256));
  return base64UrlEncode(verifier).replaceAll("=", "");
}
```

<!-- Check what's here really -->
In the code above, a random list is generated and encoded as a Base64 String to get a code verifier.

> **Note:** The `.replaceAll("=","")` operation performed on the Base64 String removes the unrequired padding which leads to error when verifying the code challenge.

#### Create a code challenge

A code verifier is required when creating a code challenge to be passed to the authorization endpoint. As you'll see in [Exchange authorization code for access tokens]() section, the code verifier passed must be the same with the code verifier used in generating a code challenge. To create a code challenge, append the snippet below to the `lib/utils/url_utils.dart` file:

```dart
///To create code challenge
String _createChallenge(String verifier) {
  var enc = utf8.encode(verifier);
  var challenge = sha256.convert(enc).bytes;
  return base64UrlEncode(challenge).replaceAll("=", "");
}
```

The code snippet above encodes the code verifier using UTF8 and converts the result to a SHA256 byte before encoding again, this time into a Base64 String to generate the code challenge.

> **Note:** The SHA256 conversion method used in generating the digest byte determines what algorithm is used in verifying the code challenge in [Exchange authorization code for access tokens]() section.

#### Construct the authorization URL.

To wrap things up on setting up the authorization, add the code snippet below to the `lib/utils/url_utils.dart` file:

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

The `getAuthorizationUrl()` function initializes the `codeVerifier` and `codeChallenge` variables. It then uses them alongside previously initialized variables such as `DOMAIN` and `SCOPES` to build the `authorizationUrl`.
