# github-app-user-auth

Provide fine-grained push access to GitHub from a JupyterHub.

## Goals

1. Allow users on a JupyterHub to grant push access to only *specific
   repositories* rather than *all* the repositories they have access to.
2. Do not store long-term credentials (like personal access tokens or
   ssh-keys) on disk, as they may get archived / fall into the wrong
   hands in the future.
3. Allow GitHub organization admins visibility and control over what
   repos users can push to from remote systems (like JupyterHub or a
   shared cluster), where other admins of the remote system might
   be able to access the files of users with push access to repos. This
   has serious implications for supply chain security, as credentials
   might be stolen or lost and serious vulnerabilities be pushed to
   the repo.

These goals are accomplished by:

1. Creating a [GitHub app](https://docs.github.com/en/developers/apps)
   specific to the remote service (JupyterHub, HPC cluster, etc). Users
   and GitHub organization admins can then provide fine grained, repo
   level access to this GitHub app - Users can only push to repos that have the
   app installed.
2. A commandline tool (`github-app-user-auth`) that lets specific users
   authorize push access to the selected repositories temporarily - a token
   that expires after 8 hours.

In the future, an optional web app might also be provided to aid in
authentication.

## Installation

You can install `github-app-user-auth` from PyPI.

```bash
pip install github-app-user-auth
```

## GitHub App configuration

1. Create a [GitHub app](https://docs.github.com/en/developers/apps) for
   use by the service (JupyterHub, HPC cluster, etc). You can either create
   it under your [personal account](https://github.com/settings/apps/new),
   or preferably under a GitHub organization account (Go to Settings ->
   Developer Settings -> GitHub Apps -> New GitHub app from the organization's
   GitHub page).

2. Give it a descriptive name and description, as your users will see this
   when they authenticate. Provide a link to a descriptive page explaining your
   service (if you are using a JupyterHub, this could be just your JupyterHub URL).

3. Disable webhooks (uncheck the 'Active' checkbox under 'Webhooks'). All other
   textboxes can be left empty.

4. Under 'Repository permissions', select 'Read & write' for 'Contents'. This
   will provide users authenticating via the app just enough permissions to push
   and pull from repositories.

5. Under 'Where can this GitHub App be installed?', select 'Any account'. This will
   enable users to push to their own user repositories or other organization repositaries,
   rather than just the repos of the user or organization owning this GitHub app.

6. Save the `Client ID` provided in the information page of the app. You'll need this
   in the client. Save the `Public link` as well, as users will need to use this to grant
   access to particular repositories.

## Client configuration

1. `github-app-user-auth` uses `git-credentials-store` to provide appropriate authentication,
    by writing to a `/tmp/github-app-git-credentials` file. This makes sure we don't override
	the default `~/.git-credentials` file someone might be using. `git` will have to be configured to use
	the new file.

	You can put the following snippet in `/etc/gitconfig` (for containers) or in
	`~/.gitconfig`:

	```ini
	[credential]
        helper = store --file=/tmp/github-app-git-credentials
	```

	Or you can run the following command (this puts it in `~/.gitconfig`)

	```
	git config --global credential.helper "store --file=/tmp/github-app-git-credentials"
	```

2. `github-app-user-auth` will need to know the "Client ID" of the created GitHub app to
    perform authentication. This can be either set with the environment variable
	`GITHUB_APP_CLIENT_ID`, or be passed in as a commandline parameter `--client-id` to
	the `github-app-user-auth` script when users use it to authenticate.

## Usage

### Grant access to the GitHub app

Users will first need to go to the public page of the GitHub app, and
'Install' the app on their account and in organizations with repos they
want to push to. We *highly* recommend allowing access only to selected
repositories, and explicitly select the repositories this hosted service
(JupyterHub, HPC cluster, etc) should be able to push to. You can modify
this list afterwards, to make sure you only grant the required permissions.

Given the common usage pattern where you are only pushing to a limited
set of repositories from a particular hosted service, this should hopefully
not be too cumborsome.

### Authenticate to GitHub

The hosted service must have `github-app-user-auth` installed.

1. Open a terminal, and type `github-app-user-auth`.
2. It should give you a link to go to, and a code to input into the web
   page when that link is opened. Open the link, enter the code there.
3. Grant access to the device in the web page, and you're done!

Authentication is valid for **8 hours**, and once it expires, this
process will need to be repeated. In the future, we might have a
web app or other process to make this less painful. However, keeping
the length of this session limited drastically helps with security too.