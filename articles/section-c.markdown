### Get the user's authorization

<!-- Remember to add the 'Google, Github and Database connection stuff to the referenced section -->
As highlighted in the [Creating an Auth0 project]() section, the application built throughout this article implements social login with Google and Github, in addition to database login. 

However, Google has a bit of history with embedded browsers - you simply cannot make OAuth authorization requests to Google via an embedded browser [any longer](https://auth0.com/blog/google-blocks-oauth-requests-from-embedded-browsers/). This constraint leaves mobile developers wth two major alternatives; using AppAuth or Chrome Custom Tabs. With AppAuth yet to have a solid Flutter implementation, the latter provides a suitable approach to making requests.

> **Note:** Although beyond the scope of this article, in using ChromeCustomTabs, you are advised to provide some form of fallback mechanism as ChromeCustomTabs is largely dependent on the user's installation of Chrome browser.

#### Launch the authorization URL within your app


 