N|Solid Cloud Foundry Buildpack
================================================================================

This buildpack is compatible with the existing Cloud Foundry Node.js buildpack,
but runs the application using the N|Solid Runtime instead of the open source
version of Node.js.  In addition, if the application is bound to a user-provided
service `nsolid-console`, pointing to an N|Solid Console server, that app will
have it's metrics tracked, etc, and will be visible in the N|Solid Console.

For more information on N|Solid, visit the [N|Solid documentation site][].

Note: this buildpack is a fork of the Cloud Foundry Node.js Buildpack available
here:

<https://github.com/cloudfoundry/nodejs-buildpack>

The original README for that buildpack is available here:

[README-cf-nodejs-buildpack.md](README-cf-nodejs-buildpack.md)

Differences between what's here and the existing Cloud Foundry Node.js Buildpack
are described in this document.

[N|Solid documentation site]: https://docs.nodesource.com/latest/docs


Usage
================================================================================

Typically you would select this buildpack by either using the command:

    cf push <push arguments / options> -b <buildpack>

or specifying the buildpack in your application's `manifest.yml` file:

    ---
    applications:
    - name:      hello-world
      buildpack: <buildpack>
      ...
      services:
       - nsolid-console

In both cases, the `<buildpack>` value is a URL to the git repo of this
buildpack:

    https://github.com/nodesource/nsolid-buildpack-cf.git

Optionally, you can specify a branch or tag at the end, starting with the `#`
character:

    https://github.com/nodesource/nsolid-buildpack-cf.git#experimental-branch

The buildpack operates the same as the existing Node.js buildpack, but arranges
for some additional code to be run before your app starts.  This code:

* computes configuration options for the N|Solid Runtime to be set as `NSOLID_*`
  environment variables

* accesses the user-provided service `nsolid-console`, that should to be bound
  to the application, which provides the connection coordinates to the N|Solid
  Console server

* optionally creates tunnels to allow the N|Solid Runtime to connect to the
  N|Solid Console server

* starts the app with the N|Solid Runtime

See below for more information on the user-provided service `nsolid-console`.


Selecting the version of Node.js to use
================================================================================

The buildpack currently only supports versions of the N|Solid Runtime
corresponding to LTS releases of Node.js - currently:

- 6.x (Boron)
- 8.x (Carbon)
- 10.x (Dubnium)

If your app will not run on these releases of Node.js, then this
buildpack cannot be used with your app.

You can select which of N|Solid Runtime to use, by setting the `engines`
property in your `package.json` to the Node.js version corresponding to the
N|Solid Runtime version.  This would either be a `6.x` or `8.x` styled semver
values - and those are good values to use, as they will select the most recent
available version of N|Solid Runtime for that Node.js release line.

For example, the following `engines` property selects the N|Solid Runtime
corresponding to Node.js 8.x (Carbon) LTS:

    "engines" : {
      "node" : "8.x"
    }

By default, the N|Solid Runtime corresponding to Node.js 10.x (Dubnium) LTS
will be used.


Customizing use of the buildpack
================================================================================

Cloud Foundry applications using this buildpack can customize the operation of
the buildpack by setting environment variables on the application.  The
following environment variables are available:

* `NSOLID_CF_RUN_TUNNELS`

  Set to `false` to indicate the socket tunnels should not be used to connect
  the N|Solid Agent to the N|Solid Console server.  Should only be needed for
  the N|Solid Console server when run as a Cloud Foundry application

* `NSOLID_CF_RUN_AGENT`

  Set to `false` to not start the N|Solid Agent for this app.  It will not
  connect to the N|Solid Console server or be visible in the N|Solid Console
  server.


Connecting to an N|Solid Console server
================================================================================

To monitor your application with the N|Solid Console, or access it via the
N|Solid Command Line Interface, the application will need to connect to the
N|Solid Console server.  There are currently two supported methods to run an
N|Solid Console server with this buildpack:

* running the server "on prem", within your private networking environment,
  which is accessible from apps running in Cloud Foundry

* running the server as a Cloud Foundry app itself, within the same Cloud
  Foundry installation

There are a number of limitations around running an N|Solid Console server
as a Cloud Foundry app, but it's also a very easy way to get started.

Limitations:

* The N|Solid Runtime will connect to the N|Solid Console server via `ssh`
  tunnels, using the `cf ssh` command.  This requires all the information
  required to run a `cf` command, including a valid userid and password.

* The N|Solid Console server persists data to disk, and so when restarted,
  will lose all persisted data.  This includes:

  * historical metrics captured
  * saved views and global notifications
  * CPU profiles and heap snapshots previously collected

Which method you use will be reflected in the user-provided service
`nsolid-console`, described below.

To use N|Solid Console "on prem", please follow the
directions available at the [N|Solid documentation site][].

To use  N|Solid Console as a Cloud Foundry app, please
follow the directions available in the
[nsolid-cf-v3 GitHub repo](https://github.com/nodesource/nsolid-cf-v3).


user-provided service `nsolid-console`
================================================================================

To connect an app using this buildpack to an N|Solid Console server, you will
need to bind a [user-provided service][] named `nsolid-console` to the app.

The `nsolid-console` service only needs to be created once per Cloud Foundry
space, and can then be bound to every app running in that space.

To create the user-provided service, the contents of the user-provided service
should be placed into a JSON file, and then you can run the following command to
create it:

    cf cups nsolid-console -p <file name>

The contents of the user-provided service can later be updated with the
command:

    cf uups nsolid-console -p <file name>

The contents of the JSON file contain the coordinates to connect to an
N|Solid Console server, and are structured differently depending on whether
your N|Solid Console server is running "on prem" or as a Cloud Foundry
app (see above).


### `nsolid-console` service for N|Solid Console server running "on prem"

The contents of the JSON file should be as follows:

```json
{
  "publicKey": "<N|Solid Console public key>",

  "tunnel":    null,

  "sockets": {
    "command": "192.168.0.2:9001",
    "data":    "192.168.0.2:9002",
    "bulk":    "192.168.0.2:9003"
  }
}
```

The `publicKey` property should be the public key configured for the N|Solid
Console server.  If no property is provided, the default N|Solid Console public
key is used.

The `tunnel` property should be either null, or not provided at all.  It's only
used when running N|Solid Console as a Cloud Foundry app.

The `sockets` property is an object which contains three other properties:
`command`, `data`, `bulk`.  These properties should contain the `host:port`
values of the N|Solid Console server's corresponding sockets.


### `nsolid-console` service for N|Solid Console server running as a Cloud Foundry app

The contents of the JSON file should be as follows:

```json
{
  "publicKey": "<N|Solid Console public key>",

  "tunnel":    "cf-ssh",

  "consoleApp": {
    "user":       "user",
    "password":   "pass",
    "cfapi":      "https://api.v3.pcfdev.io",
    "org":        "cfdev-org",
    "space":      "cfdev-space",
    "app":        "nsolid-console"
  }
}
```

The `publicKey` property should be the public key configured for the N|Solid
Console server.  If no property is provided, the default N|Solid Console public
key is used.

The `tunnel` property should be set to `cf-ssh`.

The `consoleApp` property contains the information required to run the `cf ssh`
command when the app is started, to tunnel connections to the N|Solid Console
app running as a Cloud Foundry app.  Before `cf ssh` is run, a `cf login` will
be run as follows, using the properties from the `consoleApp` object:

    cf login -u ${user} -p ${password} -o ${org} -s ${space} -a ${cfapi}

The `app` property of the `consoleApp` should be the app name of the N|Solid
Console server running as a Cloud Foundry app.

[user-provided service]: https://docs.cloudfoundry.org/devguide/services/user-provided.html


Installation
================================================================================

This buildpack can be installed as a system-available buildpack, using the
same technique as the Cloud Foundry Node.js buildpack, if your have admin
access to your Cloud Foundry instance.

* package the buildpack into the buildpack archives

      lib/vendor/nsolid/tools/build-buildpack-zip.sh

* the `bundled` version of the buildpack zip archive includes the N|Solid
  Runtimes and headers for use with `node-gyp`; the version without `bundled` in
  it's name will download the N|Solid Runtimes and headers from their
  distribution site on the web.

  * `nsolid_buildpack-vX.Y.Z.zip`
  * `nsolid_buildpack-bundled-vX.Y.Z.zip`

* install the buildpack archive in Cloud Foundry:

      cf create-buildpack nsolid_buildpack nsolid_buildpack-vX.Y.Z.zip 100
        or
      cf create-buildpack nsolid_buildpack nsolid_buildpack-bundled-vX.Y.Z.zip 100

As this buildpack will match the same applications as the Cloud Foundry Node.js
buildpack, you can choose which to use by default, by changing the position
parameter in the `cf create-buildpack` and `cf update-buildpack` commands.
To use the non-default buildpack, specify it explicitly in your `manifest.yml`
or on the `cf push` command line invocation.



Authors and Contributors
================================================================================

This code is a fork of the GitHub repo
<https://github.com/cloudfoundry/nodejs-buildpack>

Changes to that buildpack, in this repo, have been made by the following
people:

<table><tbody>
  <tr>
    <th align="left">Patrick Mueller</th>
    <td><a href="https://github.com/pmuellr">GitHub/pmuellr</a></td>
    <td><a href="https://twitter.com/pmuellr">Twitter/@pmuellr</a></td>
  </tr>
</tbody></table>


Contributing
================================================================================

Awesome!  We're happy that you want to contribute.

Make sure that you're read and understand the [Code of Conduct](CODE_OF_CONDUCT.md).


License & Copyright
================================================================================

This code is a fork of the GitHub repo
<https://github.com/cloudfoundry/nodejs-buildpack>

Changes to that buildpack have been made to this repo use the following
license and copyright.

**nsolid-buildpack-cf** is Copyright (c) 2016 NodeSource and licensed under the
MIT license. All rights not explicitly granted in the MIT license are reserved.

See the included [LICENSE.md](LICENSE.md) file for more details.
