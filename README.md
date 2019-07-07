# datasette-auth-github

[![PyPI](https://img.shields.io/pypi/v/datasette-auth-github.svg)](https://pypi.org/project/datasette-auth-github/)
[![CircleCI](https://circleci.com/gh/simonw/datasette-auth-github.svg?style=svg)](https://circleci.com/gh/simonw/datasette-auth-github)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://github.com/simonw/datasette-auth-github/blob/master/LICENSE)

Datasette plugin and ASGI middleware that authenticates users against GitHub.

This requires datasette master - it uses unreleased plugin hooks.

## Setup instructions

* Install the plugin - `pip install datasette-auth-github`
* Create a GitHub OAuth app: https://github.com/settings/applications/new
* Set the Authorization callback URL to `http://127.0.0.1:8001/-/auth-callback`
* Create a `metadata.json` file with the following structure:

```json
{
    "title": "datasette-auth-github demo",
    "plugins": {
        "datasette-auth-github": {
            "client_id": {"$env": "GITHUB_CLIENT_ID"},
            "client_secret": {"$env": "GITHUB_CLIENT_SECRET"}
        }
    }
}
```
Now you can start Datasette like this, passing in the secrets as environment variables:

    $ GITHUB_CLIENT_ID=XXX GITHUB_CLIENT_SECRET=YYY datasette \
        fixtures.db -m metadata.json

Note that hard-coding secrets in `metadata.json` is a bad idea as they will be visible to anyone who can navigate to `/-/metadata`. Instead, we use a new mechanism for [adding secret plugin configuration options](https://datasette.readthedocs.io/en/latest/plugins.html#secret-configuration-values).

By default, the plugin will redirect signed-out users directly to GitHub.

If you would rather they saw a "You are logged out" screen with a button first, you can change this behaviour by adding the "disable_auto_login" setting to your configuration:

```json
{
    "plugins": {
        "datasette-auth-github": {
            "client_id": "...",
            "client_secret": "...",
            "disable_auto_login": true
        }
    }
}
```

## Restricting access to specific users

By default the plugin will allow any GitHub user to log in. You can restrict allowed users to a specific list using the `allow_users` configuration option:

```json
{
    "plugins": {
        "datasette-auth-github": {
            "client_id": "...",
            "client_secret": "...",
            "allow_users": ["simonw"]
        }
    }
}
```
You can list one or more GitHub usernames here.

## Restricting access to specific GitHub organizations or teams

You can also restrict access to users who are members of a specific GitHub organization:

```json
{
    "plugins": {
        "datasette-auth-github": {
            "client_id": "...",
            "client_secret": "...",
            "allow_orgs": ["datasette-project"]
        }
    }
}
```

If your organization is [arranged into teams](https://help.github.com/en/articles/organizing-members-into-teams) you can restrict access to a specific team like this:

```json
{
    "plugins": {
        "datasette-auth-github": {
            "client_id": "...",
            "client_secret": "...",
            "allow_teams": ["your-organization/engineering"]
        }
    }
}
```

## Using this with the 'datasette publish' command

`allow_orgs`, `allow_users` and `allow_teams` can both be single strings rather than lists. This means you can publish data and configure the plugin entirely from the command-line like so:

    $ datasette publish nowv1 fixtures.db \
        --alias datasette-auth-demo \
        --install=datasette-auth-github \
        --plugin-secret datasette-auth-github client_id 86e397f7fd7a54d26a3a \
        --plugin-secret datasette-auth-github client_secret ... \
        --plugin-secret datasette-auth-github allow_user simonw

## Cookie expiration

The cookies set by this plugin default to expiring after 24 hours. Users with expired cookies will be automatically redirected back through GitHub to login, so they are unlikely to notice that their cookies have expired.

You can change the cookie expiration policy in seconds using the `cookie_ttl` setting. Here's how to reduce that timeout to an hour:

```json
{
    "plugins": {
        "datasette-auth-github": {
            "client_id": "...",
            "client_secret": "...",
            "cookie_ttl": 3600
        }
    }
}
```

## Forced cookie expiration

If you are using GitHub organizations or teams with this plugin, you need to be aware that users may continue to hold valid cookies even after they have been removed from a team or organization - generally for up to 24 hours unless you have changed the `cookie_ttl`.

If you need to revoke access to your instance immediately, you can do so using the `cookie_version` setting. Simply modify your metadata to add a new value for `cookie_version` and restart or redeploy your Datasette instance:

```json
{
    "plugins": {
        "datasette-auth-github": {
            "client_id": "...",
            "client_secret": "...",
            "cookie_version": 2
        }
    }
}
```

All existing cookies will be invalidated. Users who are still members of the organization or team will be able to sign in again - and in fact may be signed in automatically. Users who have been removed from the organization or team will lose access to the Datasette instance.

## Using this as ASGI middleware without Datasette

While `datasette-auth-github` is designed to be used as a [Datasette plugin](https://datasette.readthedocs.io/en/stable/plugins.html), it can also be used as regular ASGI middleware to add GitHub authentication to any ASGI application.

Here's how to do that:

```python
from datasette_auth_github import GitHubAuth
from your_asgi_app import asgi_app


app = GitHubAuth(
    asgi_app,
    client_id="github_client_id",
    client_secret="github_client_secret",
    # Other options:
    # cookie_ttl=60 * 60,
    # disable_auto_login=True,
    # allow_users=["simonw"],
    # allow_orgs=["my-org"],
    # allow_teams=["my-org/engineering"],
)
```

The keyword arguments work in the same way as the Datasette plugin settings documented above.
