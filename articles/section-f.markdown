## Conclusion and Recommendations

In this post you learned how to secure a Flutter application with Auth0 using the authorization code grant flow with PKCE, and the basics of state management with ReBLOC.

Althought Dart's default HTTP client library was used in making network requests throughout this article, the [Dio package]() is a robust alternative, providing you with features such as interceptors and ability to handle timeouts.

Finally, while its usage is limited to fetching user details in the course of this article, the access token should be kept alive throughout the lifecycle of large applications where it's needed to make frequent API calls.

I do hope that you enjoyed this tutorial. Happy hacking!