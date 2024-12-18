---
date:
  created: 2024-12-18
slug: reddit-auth
tags:
  - reddit
  - authorization
  - oauth2
---

# Implementing Reddit Auth for Flask App

Every year we organize a Gift Exchange event at [r/IndiaSocial](https://www.reddit.com/r/indiasocial/) just before Christmas. It's our version of Secret Santa. We have been organizing this even for the past 4 years, last year I thought we should get it more streamlined and so created an app. This year I fine-tuned the app a little, and the end result is a simple Flask app with Reddit authorization using OAuth2. Here's a little blog post on how to integrate Reddit authorization in your Python app.
<!-- more -->

## Why do we need a blog post?

Because Reddit authorization documentation is absolute garbage, and even then the one that exists is [8 years old](https://github.com/reddit-archive/reddit/wiki/OAuth2). A lot has changed since then, so it's time we get an update tutorial.

## How do I get started?

First, you need to create a Reddit app which will give you an application ID and a secret. You would need these to create your app. To create a Reddit app, you can head over to the [https://www.reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) page and click the **Create an app** button.

Then you need to give your app a name, select **Web app** as the type of application, and fill in a suitable description with your Reddit username in it. Finally in the **Redirect URI** field fill in a call route for your app, it should look something like this: https://www.myredditapp.com/callback.

## ...done! Now what?

Now it's time to setup the authorization flow in your app. Let's say you are creating a Flask app like I did, you need to provide an entry point for the app. Here, we have chosen the home page for this, which can be `index.html`. The app begins the authorization process when a user lands on the homepage. Define a route in your app for this entry point, it can be something as simple as the following code:

```
@app.route("/")
def homepage():
    authorize_url = make_authorization_url()
    return render_template("index.html", authorize_url=authorize_url)
```

The `make_authorization_url` function generates the URL for Reddit's authorization page dynamically to ensure flexibility and security. By constructing the URL on-the-fly, the app can incorporate unique parameters such as a randomly generated `state` for preventing CSRF attacks and a tailored `redirect_uri`. These parameters not only enhance the security of the authentication process but also ensure that the app can handle user-specific flows, such as redirecting users to appropriate locations after successful authorization.

```
def make_authorization_url():
    state = str(uuid4())  # Generate a random state to prevent CSRF attacks
    save_created_state(state)  # (Optional) Save the state for validation later

    params = {
        "client_id": CLIENT_ID,
        "response_type": "code",
        "state": state,
        "redirect_uri": REDIRECT_URI,
        "duration": "temporary",
        "scope": "identity",
    }
    url = "https://ssl.reddit.com/api/v1/authorize?" + urllib.parse.urlencode(params)
    return url
```

This function does the following:

- Generates a random `state` parameter to prevent cross-site request forgery (CSRF) attacks.
- Constructs the authorization URL with the required query parameters, including:
    - `client_id`: Your app's Reddit client ID.
    - `redirect_uri`: The URL where Reddit will send the user after authentication.
    - `scope`: The permissions being requested (in this case, "identity" to access the username).

When the user clicks the authorization link, they are redirected to Reddit's authorization page.

## Okay, but how do we handle the callback?

After the user grants or denies access, Reddit redirects them back to the app via the `redirect_uri`. This step is a critical part of the OAuth2 process, as it serves as the bridge between the user's action on Reddit and the app's ability to proceed with authorization securely.

The callback route, `/callback`, plays several key roles:

- Error Handling: It captures any errors Reddit might send back, such as the user denying access, and communicates them to the user.
- State Validation: The `state` parameter, generated during the authorization URL creation, is validated here to prevent cross-site request forgery (CSRF) attacks. Only requests with a matching state are allowed to proceed, ensuring the callback was initiated by the app.
- Code Exchange: The authorization code received from Reddit is exchanged for an access token. This token is essential for making authenticated API requests on behalf of the user.

By handling the callback securely and efficiently, the app ensures that only legitimate users can access its features and that sensitive data, such as the access token, is protected throughout the process.

After the user grants or denies access, Reddit redirects them back to the app via the `redirect_uri`. The app handles this in the `/callback` route.

```
@app.route("/callback")
def callback():
    error = request.args.get("error", "")
    if error:
        return "Error: " + error

    state = request.args.get("state", "")
    if not is_valid_state(state):
        abort(403)  # Reject invalid state values

    code = request.args.get("code")
    access_token = get_token(code)  # Exchange the code for an access token
    username = get_username(access_token)  # Fetch the Reddit username
    session["username"] = username

    return render_template("index.html", username=username)
```

## But how to get the access token?

An access token is a critical piece in the OAuth2 process. It acts as a secure credential that the app uses to interact with Reddit's API on behalf of the user. Without it, the app would not be able to fetch user-specific data, such as their Reddit username, or perform any authenticated actions.

The app exchanges the authorization code for an access token by making a POST request to Reddit's API.

```
def get_token(code):
    client_auth = requests.auth.HTTPBasicAuth(CLIENT_ID, CLIENT_SECRET)
    post_data = {
        "grant_type": "authorization_code",
        "code": code,
        "redirect_uri": REDIRECT_URI,
    }
    headers = base_headers()

    response = requests.post(
        "https://ssl.reddit.com/api/v1/access_token",
        auth=client_auth,
        headers=headers,
        data=post_data,
    )

    token_json = response.json()
    return token_json["access_token"]
```

This function uses basic authentication with the app's `CLIENT_ID` and `CLIENT_SECRET` to verify the app's identity. It then sends the `code` and `redirect_uri` as part of the payload to ensure the request aligns with Reddit's OAuth2 process. Finally, it returns the `access_token` from Reddit's response, enabling the app to access the user's data securely.

By retrieving an access token, the app ensures that user interactions with Reddit's API remain secure and specific to the authenticated user. The app exchanges the authorization code for an access token by making a POST request to Reddit's API.

## ...and the username?

With the access token, the app fetches the authenticated user's Reddit username.

```
def get_username(access_token):
    headers = base_headers()
    headers.update({"Authorization": "bearer " + access_token})

    response = requests.get("https://oauth.reddit.com/api/v1/me", headers=headers)
    me_json = response.json()
    return me_json["name"]
```

This function sends a GET request to Reddit's `/api/v1/me` endpoint with the `Authorization` header containing the access token. The username is extracted from the response JSON.

## But how to retain the username?

Simple, once the username is retrieved, store it in the user's session.

```
session["username"] = username
```

This post demonstrates a complete implementation of Reddit's OAuth2 flow:

- Generating an authorization URL.
- Handling the callback and exchanging the authorization code for an access token.
- Using the access token to fetch the user's Reddit username.
- Personalizing the user's experience based on their authentication status.

By leveraging Reddit's OAuth2 API, the app ensures a secure and seamless login process for users, making it an engaging platform for a Secret Santa event. For more information on scopes, error messages etc. you can refer to the archived page link given at the beginning of the post.