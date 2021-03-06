# `shibboleth-idp-docker`

## Shibboleth v3 Identity Provider Deployment using Docker

This project is a workspace in which I am experimenting with deploying the
[Shibboleth](http://shibboleth.net)
[v3 Identity Provider](https://wiki.shibboleth.net/confluence/display/IDP30/Home)
software using the [Docker](http://www.docker.com) container technology.

Although this is what I'm using to deploy my own, rather minimal,
identity provider in "production", there's no guarantee that anything
here actually works. If you find
something useful you're welcome to take advantage of it.

## Base Image and Java

This Docker build is based on [Amazon Corretto][] 11, an OpenJDK distribution
with long term support. This is produced by Amazon and used for many of their
own production services.

[Amazon Corretto]: https://aws.amazon.com/corretto/

If you want to replace this with another Java distribution, change the definition
of `JAVA_VERSION` in `VERSIONS`:

```
#
# Java
#
# Base image to use for the build.
#
JAVA_VERSION=amazoncorretto:11
```

Any JDK from Java 7 onwards will _probably_ work, with the proviso that if you
use something without the
[Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)
or equivalent the IdP won't able to use some useful encryption algorithms with "long" keys.
One important example is 256-bit AES, which is eligible for use in XML encryption
of messages sent to service providers.
This functionality is built in to all current versions of Java, though.


## Fetching the Jetty Distribution

You should execute the `./fetch-jetty` script to pull down a copy of the Jetty distribution
into `jetty-dist/dist`. The variables `JETTY_VERSION` and `JETTY_DATE in the `VERSIONS` file
control the version acquired.

Some minimal validation is performed of the downloaded file, but at present it's on a "leap of faith"
basis as Jetty's approach to distribution signing has been a little hit and miss. Feel free to submit
a pull request if you have a better way of handling this.

## Jetty 9.4 Configuration

Prior to 2020-02-04, the Jetty configuration used was part of the IdP installation mounted into
the running container. The actual configuration used was derived from the Jetty base used by the
Shibboleth project's Windows installer, and then locally edited. One advantage of this setup was
that the keystore passwords were not made part of the container image. One disadvantage is that
the installer mechanisms used to do this were not part of the supported API.

In the current iteration, the Jetty configuration has been moved inside the container image.
As part of the build, the `jetty-base-9.4` directory in this repository is copied to `/opt/jetty-base`
in the image. This is still _derived_ from the same source, but no longer depends on undocumened
features of the Shibboleth installer, and comes pre-customised for the container environment.
Additionally, it lives outside the `/opt/shibboleth-idp` directory, which gives a cleaner
separation between Jetty and the IdP.

This default configuration uses default keystore passwords as follows.

In `jetty-base-9.4/start.d/idp.ini`:

```
## Keystore password
jetty.sslContext.keyStorePassword=changeit
## Truststore password
jetty.sslContext.trustStorePassword=changeit
## KeyManager password
jetty.sslContext.keyManagerPassword=changeit
```

In `jetty-base-9.4/start.d/idp-backchannel.ini`:

```
## Backchannel keystore password
# idp.backchannel.keyStorePassword=changeit
```

Arguably, there's little point in changing these values in the obvious way, as whatever you do
will end up on disk and additionally in a container image. I do want to use Docker secrets, or
Vault, or some other secrets management system to acquire and inject secrets like this. Until I get
round to that, though, I'd be delighted to get a pull request in this area.

If you do want to change these or other values, or make any other local customisations to the
Jetty configuration, you can of course just make a private branch of this repository and change
the files in `jetty-base-9.4` directly. I have also provided an overlay system to make this a
bit cleaner.

If you create, for example, `overlay/jetty-base-9.4/start.d/idp.ini`, then that file will overwrite
the one taken from `jetty-base-9.4`. Anything under `overlay` is ignored by Git so it can be a local
repository unconnected with this one. I have also made it possible for `overlay/jetty-base-9.4` to be
a symbolic link so that it can link to somewhere _inside_ another local repository.

See [`overlay/README.md`](overlay/README.md) for more detail on the overlay system.

## Jetty 9.3 Configuration

If you're using v3 of the Shibboleth Identity Provider, it's possible to use Jetty 9.3. The remarks
above for Jetty 9.4 apply with obvious changes: the build will copy in the appropriate Jetty base
and overlay if you set appropriate values in VERSIONS.

Jetty 9.3 can't be used with v4 of the identity provider, so I will eventually remove this support.

## Building the Image

Execute the `./build` script to build a new container image. This new image will be
tagged as `shibboleth-idp` and incorporate the Jetty distribution fetched earlier,
the `jetty-base` from this repository and any `overlay/jetty-base` you have created.

It will *not* include
the contents of `shibboleth-idp`; instead, they will be mounted into the container at `/opt/shibboleth-idp` when
a container is run from the image.

One important result of this approach is that the container image does not incorporate any secrets that are
part of the Shibboleth configuration, such as passwords. On the other hand, the container image doesn't really
contain much of the IdP, just a tailored environment for it.

**Note:** If a new version of Jetty is released and you wish to incorporate it, simply change the
version components in `VERSIONS`, and then execute `./fetch-jetty` and `./build`. Then,
terminate and re-create your container. You don't need to reinstall Shibboleth for this, as it's
not part of the image.

## Fetching the Shibboleth Distribution

You should execute the `./fetch-shib` script to pull down a copy of the Shibboleth IdP distribution
into `fetched/shibboleth-dist`. A variable at the top of the script controls the version acquired.

Some minimal validation is performed of the downloaded file using a file of PGP keys published by
the Shibboleth project and included here to avoid taking a complete "leap of faith" approach.

## Shibboleth "Install"

Before attempting the next step, you should edit the `install-idp` script to change the critical
parameters at the top:

* `UFPASS` and `SEALERPASS` are passwords to use if the user-facing TLS credential or data sealer keystores,
respectively, need to be generated. It's arguable whether changing the default `X-changethis` values
really adds any security given that the values are just put in the clear in property files anyway.
* `SCOPE` should be your organizational scope.
* `HOST` is built from `SCOPE` by prepending `idp2.`, which probably won't suit you.
* `ENTITYID` is built from `HOST`. The default here is the same as the interactive install would suggest.

Executing the `./install` script will now run the Shibboleth install process in a container based on the
configured Docker Java image. If you do not have a `shibboleth-idp` directory, this will act like a first-time
install using the parameters you set before, resulting in a basic installation in that directory.

If `shibboleth-idp` already exists, `./install` will act to upgrade it to the latest distribution. This should
be idempotent; you should be able to just run `./install` at any time without changing the results. In this
case, the variables set at the top of the `install` script won't have any effect as the appropriate values
are already frozen into the configuration.

## Container Configuration

A rudimentary mechanism is provided to allow configuration of the scripts provided to interact with the Docker
container. Each relevant script defers configuration to `script-functions`, which sets defaults and then in turn
invokes `CONFIG` (if present) to override those defaults. An example `CONFIG` file is provided as
`CONFIG.example`.

By default, the container's port 443 and 8443 are bound to the Docker host's same-numbered ports on all
available interfaces. This is probably the right choice for most people. If you need to override this, you
can set `IPADDR` to a specific IP address in the `CONFIG` file.

Setting `IPADDR=127.0.0.1`, for example, might be useful to allow access to the IdP from only the
Docker host itself during testing. Another use for `IPADDR` would be to single out a specific host
interface on a multi-homed host.

## Executing the Container

Start a randomly named container from the image using the `./test` script. This is set up to be an interactive
container; you will see a couple of lines of logging and then it will appear to pause. Use ^C to stop the
container; it will be automatically removed when you do so.

All state, such as logs, will appear at appropriate locations in the `shibboleth-idp` directory tree.

## Other Lifecycle Scripts

Also included are:

* `./run` is a more conventional script to start a container called `shibboleth-idp` from the image
and run it in the background.
* `./stop` stops the `shibboleth-idp` container.
* `./terminate` stops the `shibboleth-idp` container and removes the container. This is useful if
you want to build and run another container version.
* `./cleanup` can be used at any time to remove orphaned
containers and images, which Docker tends to create in abundance during
development. Use `./cleanup -n` to "dry run" and see what it would remove.
Docker has got a fair bit better at doing this itself over time, but you may still want
to run this once in a while to clear out dead wood.

## OpenSSL Tips

You will often find that you have keys and certificates in a format other than the one you'd like. Here are
some recipes I've found useful in this work.

### Self-signed Certificate and Key to PKCS12

If you have a pair of PEM files (normally a self-signed certificate and the corresponding private key) and
need them in PKCS12 form for use with Jetty, you can try something like this:

    $ openssl pkcs12 -export -out X.p12 -inkey X.key -in X.crt

You'll be prompted for the output password, and the input password if one is required.

### Commercial Certificate and Key to PKCS12

If you have a certificate from a commercial CA, it will normally come with a bundle of intermediates
which need to be presented by the server in addition to the end entity certificate. If you need to
convert a private key, certificate and bundle to PKCS12 form for use with Jetty, try this:

    $ openssl pkcs12 -export -out X.p12 -inkey X.key -in X.crt -certfile chain.pem

Again, you'll be prompted for any relevant passwords.


## Copyright and License

The entire package is Copyright (C) 2014&ndash;2020, Ian A. Young.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

