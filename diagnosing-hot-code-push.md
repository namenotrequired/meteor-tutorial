# Diagnosing Hot Code Push issues in Meteor Cordova

Meteor’s [Hot Code Push](https://guide.meteor.com/cordova#hot-code-push) is a [great power](https://blog.meteor.com/meteor-hot-code-push-with-great-power-comes-great-responsibility-7e9e8f7312d5), but it doesn’t always work seamlessly.

So your `.meteor/versions` file includes `hot-code-push`, yet your cordova app is not getting the updates you’re deploying? Here’s some things you can try.

If you get stuck, feel free to get in touch.

## Override compatability versions

Did the app suddenly stop getting new code after you changed cordova version or plugins?

The client may log: `Skipping downloading new version because the Cordova platform version or plugin versions have changed and are potentially incompatible`.

Cordova and its plugins cannot be updated through Hot Code Push. So Meteor by default disables Hot Code Push to app versions that have different versions than the server. This avoids crashing a user’s app, for example, when new JS calls a plugin that his app version doesn’t yet have.

You can [override this behavior](https://guide.meteor.com/cordova#controlling-compatibility-version). Just make sure you deal with potentially incompatible versions in your JS instead.

## Set an AUTOUPDATE_VERSION

Explained pretty clearly in the [QA document](https://github.com/meteor/meteor/blob/devel/packages/autoupdate/QA.md#autoupdate_version) of the [autoupdate](https://github.com/meteor/meteor/tree/devel/packages/autoupdate) package.

If adding this variable when you `run` or `deploy` causes HCP to suddenly work, it means for some reason, your server thinks the client code and assets didn’t actually change.

You can set it for every deploy and change its value every time you want to update your clients. Or you can debug further, and try to figure out why [WebApp.calculateClientHash](https://github.com/meteor/meteor/blob/devel/packages/webapp/webapp_server.js#L267) isn’t generating new hashes for your new code.

## Cordova doesn’t hot reload CSS separately

Are you seeing your web app incorporating changes without reload, yet your cordova app reloads each time?

For CSS-only changes, this is the expected behaviour. Browsers update the layout without reload, but in cordova, [any change reloads the whole app](https://docs.meteor.com/packages/autoupdate.html#Cordova-Client).

I don't know of a way to change this without making big changes to the underlying packages (which I'll help with below).

## Outdated custom reload code and packages

There are [several reload packages](https://atmospherejs.com/?q=reload), and maybe your app includes some custom reload code. Of course, these may have bugs.

Specifically, when you push an update, does the app reload but use the old code anyways?

Probably, the code hasn't been updated to work with Meteor 1.8.1 or later. [Meteor recommends](https://docs.meteor.com/changelog.html#v18120190403) you call `WebAppLocalServer.switchToPendingVersion` before forcing a browser reload.

Alternatively, use the built-in behavior to reload. Instead of, say, `window.location.reload()`, call the `retry` function passed to the `Reload._onMigrate()` callback. For example:

```js
Reload._onMigrate((retry) => {
  if (/* not ready */) {
    window.setTimeout(retry, 5 * 1000); // Check again in 5 seconds
    return [false];
  }
  // ready
  return [true];
});
```

## Avoid hash fragments

Cordova doesn’t show the URL bar, but the user is still on some URL or other, which may have a hash (`#`). HCP [works better if it doesn't](https://github.com/meteor/meteor/blob/devel/packages/reload/reload.js#L224).

If you can, remove the hash before the reload.

## Avoid big files not included in the app

While [remotely debugging an android app](https://developers.google.com/web/tools/chrome-devtools/remote-debugging#remote-debugging-on-android-with-chrome-devtools), I was seeing HCP fail with errors like:

```
Error: Error downloading asset: /
  at http://localhost:12472/plugins/cordova-plugin-meteor-webapp/www/webapp-local-server.js:51:21
  at Object.callbackFromNative (http://localhost:12472/cordova.js:287:58)
  at <anonymous>:1:9
```

It turns out that [cordova-plugin-meteor-webapp](https://github.com/meteor/cordova-plugin-meteor-webapp) threw this error because some files in `public` were too big. Downloading would fail depending on connection speed and available space.

Does HCP start working if you delete the content of your `public` folder? Then this may be your issue.

I ran [`du -a public | sort -n -r | head -n 20`](https://www.cyberciti.biz/faq/linux-find-largest-file-in-directory-recursively-using-find-du/) to find the biggest files. In my case, these were used only on rarely accessed routes, so I could safely move them to a CDN. This solved the issue.

## If it is only broken locally

If you notice HCP works in production but not when you test locally, you may need to enable clear text or set a correct `--mobile-server`. Both are [explained in the docs](https://docs.meteor.com/packages/autoupdate.html#Cordova-Client).

## Still having issues?

If none of that solved your issues and you’d like to dive deeper, here’s some tips to get you started.

## Where does hot code push live?

Hot code push is included in `meteor-base` through a web of [official meteor packages](https://github.com/meteor/meteor/tree/devel/packages), most importantly [`reload`](https://github.com/meteor/meteor/tree/devel/packages/reload) and [`autoupdate`](https://github.com/meteor/meteor/tree/devel/packages/autoupdate).

In the case of cordova, a lot of the heavy lifting is done by [`cordova-plugin-meteor-webapp`](https://github.com/meteor/cordova-plugin-meteor-webapp).

To oversimplify, `autoupdate` decides *when* to refresh the client, the plugin then downloads the new client code and assets, and `reload` then refreshes the page to start using them.

## What are the steps it takes?

We can break it down a bit more:

- whenever the server thinks the client side may have changed, it calculates a hash of your entire client bundle
- it [publishes](https://docs.meteor.com/api/pubsub.html) this hash to all clients
- the clients subscribe to this publish
- when a new hash arrives, the client compares it to its own hash
- if it’s different, it starts to download the new client bundle
- when it’s done, the client saves any data and announces that it will reload
- the app and packages get a chance to [save their data or to deny the reload](https://forums.meteor.com/t/is-there-an-official-documentation-of-reload--onmigrate/16974/2)
- if/when allowed, it reloads

## How to spy on it?

To figure out where the issue is, we can log the various steps HCP takes.

First, make sure you can [see client-side logs](https://guide.meteor.com/cordova#logging-and-remote-debugging) (or print them on some screen of your app). You may start seeing some default logs already.

A few more useful values to print, and events to listen to, might be:

- The version hashes: `__meteor_runtime_config__.autoupdate.versions['web.cordova']`

- The reactive [`Autoupdate.newClientAvailable()`](https://github.com/meteor/meteor/blob/devel/packages/autoupdate/QA.md#autoupdatenewclientavailable): if this turns into `true` and then doesn’t refresh, you know the client does receive the new version but something goes wrong trying to download or apply it.

```js
Tracker.autorun(() => {
  console.log(‘new client available:’, Autoupdate.newClientAvailable());
});
```

- To check the client’s subscription to the new versions, check `Meteor.default_connection._subscriptions`. For example, to log whether the subscription is `ready` and `inactive` (using lodash):

```js
const { ready, inactive } = _.chain(Meteor)
  .get('default_connection._subscriptions', {})
  .toPairs()
  .map(1)
  .find({ name: 'meteor_autoupdate_clientVersions' })
  .pick(['inactive', 'ready']) // comment this to see all options
  .value();
console.log(‘ready:’, ready);
console.log(‘inactive:’, inactive);
```
Or, to log `ready` as soon as the subscription changes:

```js
const hcpSub = _.chain(Meteor)
  .get('default_connection._subscriptions', {})
  .toPairs()
  .map(1)
  .find({ name: 'meteor_autoupdate_clientVersions' })
  .value(); // no .pick() this time; return whole subscription object

Tracker.autorun(() => {
  hcpSub.readyDeps.depend(); // Rerun when something changes in the subscription
  console.log('hcpSub.ready', hcpSub.ready);
});
```
Should print `false` and then `true` less than a second later.

- To see if we finish downloading and preparing the new version, listen to `WebAppLocalServer.onNewVersionReady`;

```js
WebAppLocalServer.onNewVersionReady(() => {
  console.log('new version is ready!');
  // Copied from original in autoupdate/autoupdate_cordova.js because we overwrite it
  if (Package.reload) {
    Package.reload.Reload._reload();
  }
});
```

- To see if permission to reload is being requested, listen to `Reload._onMigrate()`. Be sure to return `[true]` or the reload may not happen. (I believe that if this is run in your app code, it means all packages allowed the reload. But I didn’t find my source on this.)

```js
Reload._onMigrate(() => {
  console.log('going to reload now');
  return [true];
});
```

- To know if your `startup` was the result of a HCP reload or not, we can take advantage of the fact that `Session`s (and `ReactiveDict`s) are preserved.

```js
Meteor.startup(() => {
  console.log('Was HCP:', Session.get('wasHCP'));
  Session.set('wasHCP', false);

  Reload._onMigrate(() => {
    Session.set('wasHCP', true);
    return [true];
  });
});
```

## How to edit the source

Finally, if you want to change some of the package and plugin code, you can.

## Editing the packages

Say we want to edit the `autoupdate` package.

In the root of your project, create a folder named `packages`, then add a folder `autoupdate`. Here we put the code from the original package (found in `~/.meteor/packages`), then we edit it.

Meteor will now use the local version instead of the official one.

## Editing the plugin

To install a modified version of a plugin,

- from another folder, download the original code e.g. `git clone https://github.com/meteor/cordova-plugin-meteor-webapp.git`
- install it into your meteor project with [`meteor add cordova:cordova-plugin-meteor-webapp@file://path/to/cordova-plugin-meteor-webapp`](https://stackoverflow.com/a/35941588/5786714)
- modify it as you like

Meteor will start using the local version instead of the official one. But note you will have to rerun `meteor build` or `meteor run` every time you change the plugin.

## Good luck!

Did it help? Are you stuck? I'd love to hear.

Bart Sturm, co-founder & head of engineering at [TutorMundi](https://tutormundi.com/)
