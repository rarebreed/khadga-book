# Authenticating a user

For khadga, we will be using the Google OAuth mechanism.

## Setting things up

There are actually 2 parts to this:

- Setting up a google account with OAuth Credentials
- The front end client

## Setting up google account for OAuth

The first thing we will need to do is to set up a google account.  Go to
[console.developers.google.com][-console] and select or create a project.  Since the UI is always
changing, it's hard to say what to look for, but currently, in the left hand sidebar, there is a
link that says `Credentials`.

If you click on `Credentials`, a new layout appears that shows your current credentials for your
project:

- API Keys
- OAuth2 credentials
- Service Accounts

At the top, there is a button to `+ Create Credentials`.  In the drop down that appears, select the
OAuth client ID option.  In the next screen, you can select `Web Option`.  A new section appears and
you can optionally give it a name.

There will also be a field that specifies authorized javascript origins.  For demo purposes, we can
put in `http://localhost:7001` which is where our khadga backend, and therefore where our web page's
`window.location` will be.


## Setting up the front end client

The first thing we need to do is load the javascript.  Unfortunately, google does not release an npm
package, so we need to load this in a `<script>` tag in our index.html.

```html
  <head>
    <meta charset="UTF-8" name="viewport" content="width=device-width, initial-scale=1">
    <title>Grid test web app!</title>
    <script defer src="https://use.fontawesome.com/releases/v5.3.1/js/all.js"></script>
    <script src="https://apis.google.com/js/api.js"></script>
    <link rel="stylesheet" href="./styles.css"><link>
  </head>
```

Once we do this, we will have access to the `gapi` object from our window.navigator.  Since we are
using typescript, adding objects to the global `window` object is a bit hacky.  We will do a simple
approach which is not type-safe, and then improve on it later.

```javascript
function latestName(conn, user) {
	let baseName = user;
	let re = new RegExp(`${baseName}-(\\d+)`);
	let index = 0;
	for (let name of conn) {
		if (name === baseName) {
      console.log(`${name} in list matches current of ${baseName}`)
			let matched = name.match(re);
			if (matched) {
				index = parseInt(matched[1]) + 1;
				console.log(matched);
				console.log(`Got match, index is ${index}`);
				baseName = baseName.replace(/\d+/, `${index}`);
			} else {
				index += 1;
				baseName = `${baseName}-${index}`;
			}
		} else {
      console.log(`${name} does not equal ${baseName}`)
		}
		console.log(`baseName is now ${baseName}`)
	}
	return baseName;
}

latestName(["sean", "sean-1", "sean-2"], "sean")
```

[-console]: https://console.developers.google.com