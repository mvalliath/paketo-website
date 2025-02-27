---
title: "How To Configure Paketo Buildpacks"
weight: 340
menu:
  main:
    parent: howto
    identifier: configuration
    name: "Configure Paketo Buildpacks"
aliases:
  - /docs/buildpacks/configuration/
  - /docs/reference/configuration/
---

## About the Examples

Configuration examples will use the Paketo [sample applications][samples].

Examples assume that the root of this repository is the working directory:
{{< code/copyable >}}
git clone https://github.com/paketo-buildpacks/samples
cd samples
{{< /code/copyable >}}

The [pack CLI][pack] is used throughout the examples. `pack` is just one of several Cloud Native Buildpack [platforms][platforms] than can execute builds with Paketo Buildpacks.

Examples assume that the [Paketo Base builder][base builder] is the default builder:
{{< code/copyable >}}
pack config default-builder paketobuildpacks/builder:base
{{< /code/copyable >}}

## Types of Configuration

Paketo buildpacks can be configured via the following mechanisms:

* [Environment Variables]({{< relref "#environment-variables" >}}) - used for generic configuration at both **build-time** and **runtime**.
* [buildpack.yml]({{< relref "#buildpackyml" >}}) - used for generic configuration at **build-time**.
* [Bindings]({{< relref "#bindings" >}}) - used for **secret** configuration at both **build-time** and **runtime**.
* [Procfiles]({{< relref "#procfiles" >}}) - used to provide custom **process types** at **build-time**.

### Environment Variables

#### Build-time Environment Variables

Users may configure the build by setting variables in the buildpack environment. The names of variables accepted by the Paketo buildpacks at build-time are either prefixed with `BP_` or have well-known conventional meanings outside of Paketo (e.g. `http_proxy`).

The following example uses an environment variable to configure the JVM version installed by the Java Buildpack.

{{< code/copyable >}}
pack build samples/java  \
  --path java/jar \
  --env BP_JVM_VERSION=8
{{< /code/copyable >}}

During the build process, a buildpack may invoke other programs that accept configuration via the environment. Users may configure these tools as they would normally. For example, the command below configures the JVM memory settings for the JVM running Maven using [`MAVEN_OPTS`][maven opts].

{{< code/copyable >}}
pack build samples/java  \
  --path java/maven \
  --env "MAVEN_OPTS=-Xms256m -Xmx512m"
{{< /code/copyable >}}

#### Runtime Environment Variables

Users may configure runtime features of the app image by setting environment variables in the app container.  The names of variables accepted by buildpack-provided runtime components (e.g. profile scripts and processes types) are prefixed with `BPL_` or have well-known conventional meanings outside of Paketo (e.g `JAVA_TOOL_OPTIONS`).

The following example uses `JAVA_TOOL_OPTIONS` to set the server port of the sample application:
{{< code/copyable >}}
docker run --rm --publish 8082:8082 --env "JAVA_TOOL_OPTIONS=-Dserver.port=8082" samples/java
curl -s http://localhost:8082/actuator/health
{{< /code/copyable >}}

Programs invoked at runtime, including the application itself, will accept environment as they would normally.

#### Image Embedded Environment Variables

Users may embed environment variables into the images created by using the [Environment Variables buildpack][bp/environment-variables]. The Environment Variables buildpack looks for environment variables matching the pattern `$BPE_*`. When detected, the buildpack will modify the launch environment to adjust the specified variables. This is a good way to set non-sensitive configuration values such as defaults or modify environment variables that you do not need users to set.

The buildpack supports the following actions on environment variables:

| Environment Variable Name | Description                                                  |
| ------------------------- | ------------------------------------------------------------ |
| `$BPE_<NAME>`             | set `$NAME` to value (same as override)                      |
| `$BPE_APPEND_<NAME>`      | append value to `$NAME`                                      |
| `$BPE_DEFAULT_<NAME>`     | set default value for `$NAME`                                |
| `$BPE_OVERRIDE_<NAME>`    | set `$NAME` to value                                         |
| `$BPE_PREPEND_<NAME>`     | prepend value to `$NAME`                                     |

For more details on actions, you can refer to the [environment variable modification rules from the buildpacks spec](https://github.com/buildpacks/spec/blob/main/buildpack.md#environment-variable-modification-rules).

You can also change the delimiter used when appending or prepending by setting `$BPE_DELIM_<NAME>` for a particular variable name. It will default to an empty string (i.e. no delimiter). An example of using this would be to append to PATH or LD_LIBRARY_PATH, which are colon delimited.

**DO NOT** embed sensitive credentials or information using the environment variables buildpack. This information is added to the image generated by your build tool, so anyone with access to the image can see what you embed using this buildpack.

### buildpack.yml

Many Paketo buildpacks accept configuration from a `buildpack.yml` file if one is present at the root of the application directory.

For example, to configure the NodeJS version installed by the NodeJS Buildpack, create a file named `buildpack.yml` in the `nodejs/yarn` directory in the [samples repo][samples].
{{< code/copyable >}}
nodejs:
  version: 12.12.0
{{< /code/copyable >}}

Next, execute the build as normal and observe the that the specified version of NodeJs is installed.
{{< code/copyable >}}
pack build samples/nodejs --path nodejs/yarn
{{< /code/copyable >}}

### Bindings

#### Why bindings?

Some Paketo Buildpacks and components installed by the Paketo Buildpacks accept credentials and other secrets using bindings at build and runtime. Commonly, bindings provide the location and credentials needed to connect to external services.

 Some categories of external services one might want to bind at build-time include:

* Private artifact repositories.
* SaaS security scanning tools.

For example, the Maven buildpack accepts the location and credentials need to connect to a private Maven repository in a binding.

 Some categories of external services  one might want to bind at runtime include:

* APM servers.
* Data Services.
* OAuth2 providers.

For example, the Spring Boot Buildpack will install [Spring Cloud Bindings][spring cloud bindings] which is capable of auto-configuring Spring Boot application configuration properties to connect the application to a variety of external services, when a binding is provided at runtime.

#### What is a binding?

A Binding contains:

1. A **name**. Indentifies a particular binding. The name typically does not affect build or runtime behavior but may be used to reference a specific binding in output such as log messages.
1. A **type** or **kind**. Indicates what type of credentials the binding contains. For example, a binding of type `ApplicationInsights` contains the credentials needed to connect to Azure Application Insights.
1. An _optional_ **provider**. Indicates the source of the binding. For example, in a PaaS context, a specific service broker might provide the binding.
1. key-value pairs. These contain the configuration data. For example, an `ApplicationInsights` binding may contain a key-value pair with key `InstrumentationKey`.

Bindings must be presented to buildpacks as directories (typically volume mounted) on the container filesystem. The name of the directory provides the name of the binding. The contents of a binding can be provided using one of two specifications.

* [Service Binding Specification for Kubernetes][k8s service bindings]. This specification should be preferred over the CNB Bindings Specification when supported by the platform.
* [Cloud Native Buildpacks Bindings Specification][cnb bindings]. The original buildpacks bindings specification; this specification is en route to [deprecation][cnb bindings deprecation]

Paketo Buildpacks will look for bindings in the `/platform/bindings` directory at build-time and in `$SERVICE_BINDING_ROOT` or `$CNB_BINDINGS` directory at runtime.

For example, the Java Buildpack accepts a binding with `type` equal to `maven` containing a key named `settings.xml` containing [Maven settings][maven settings]. In the build container, the Maven Buildpack will use `settings.xml` if it finds either
{{< code/copyable >}}
/platform
└── bindings
    └── <name>
        ├── settings.xml
        └── type
{{< /code/copyable >}}
or
{{< code/copyable >}}
/platform
└── bindings
    └──<name>
        └── metadata
        |   └── kind
        └── secret
            └── settings.xml
{{< /code/copyable >}}
on the filesystem, where either the `type` or `kind` file contains the string `maven`.

#### How to use bindings?

The workflow for creating a binding and providing it to a build will depend on the chosen platform. For example, `pack` users should use the `--volume` flag to mount a binding directory into the build or app containers. Users of the `kpack` platform should store key value pairs in a Kubernetes Secret and provide that secret and associated metadata to an Image as described in the [kpack documentation][kpack service bindings].

**Example:**  Providing a Binding to `pack build`

Given a directory containing a build-time binding, `pack` users can provide this binding to a Paketo buildpack using the `--volume` flag.
{{< code/copyable >}}
pack build --volume <absolute-path-to-binding>:/platform/bindings/<binding-name> <image-name>
{{< /code/copyable >}}

**Example:**  Providing a Binding to `docker run`

Given a directory containing a runtime binding, `docker` users can provide the binding to the app image using the `--volume` and `--env` flags
{{< code/copyable >}}
docker run --env SERVICE_BINDING_ROOT=/bindings --volume <absolute-path-to-binding>:/bindings/<binding-name> <image-name>
{{< /code/copyable >}}

### Procfiles

Paketo users may override buildpack-provided types or augment the app-image with additional process types using a `Procfile`.  `Procfile` support is provided by the [Paketo Procfile Buildpack][bp/procfile]. The Procfile Buildpack will search for a file named `Procfile` at the root of the application. `Procfiles` should adhere to the following schema:

```plain
<type>: <command>
```

If a given [language family buildpack][language family buildpacks] does not contain the Procfile Buildpack it can be explicitly appended at runtime.

**Example**: A Hello World Procfile

The following adds a process with `type` equal to `hello` and makes it the default process.
{{< code/copyable >}}
echo "hello: echo hello world" > nodejs/yarn/Procfile
pack build samples/nodejs \
  --path nodejs/yarn \
  --buildpack paketo-buildpacks/nodejs \
  --buildpack paketo-buildpacks/procfile \
  --default-process hello
docker run samples/nodejs # should print "hello world"
{{< /code/copyable >}}

## Building Behind a Firewall

### Proxy Configuration

Paketo Buildpacks can be configured to route traffic through a proxy using the `http_proxy`, `https_proxy`, and `no_proxy` environment variables. `pack` will set these environment variables in the build container if they are set in the host environment.

### Dependency Mappings

Paketo Buildpacks may download dependencies from the internet. For example, the Java Buildpack with download the BellSoft Liberica JRE will from the Liberica [github releases][liberica releases] by default.

If a dependency URI is inaccessible from the build environment, a [binding]({{< relref "#bindings" >}}) can be used to map a new URI to a given dependency. This allows organizations to upload a copies of vetted dependencies to an accessible location and provide developers and CI/CD pipelines with configuration pointing the buildpack at the accessible dependencies.

The URI mappings can be configured with one or more bindings of `type` `dependency-mapping`. Each key value pair in the binding should map the `sha256` of a dependency to a URI. Information about the dependencies a buildpack may download (including the `sha256` and the current default `uri`) can be found in the `buildpack.toml` of each component buildpack.

**Example** Mapping the JRE to an internal URI

For example, to make the Bellsoft Liberica JRE dependency accessible available to builds in an environment where Github is inaccessible, an operator should:

1. Find the `sha256` and default `uri` for the desired dependency in [buildpack.toml][bp/bellsoft-liberica/descriptor] of the [Bellsoft Liberica buildpack][bp/bellsoft-liberica]. Example values:
    * `sha256`: `b4cb31162ff6d7926dd09e21551fa745fa3ae1758c25148b48dadcf78ab0c24c`
    * `uri`: `https://github.com/bell-sw/Liberica/releases/download/11.0.8+10/bellsoft-jre11.0.8+10-linux-amd64.tar.gz`
2. Download the dependency from the `uri` and upload it to a location on the internal network that is accessible during the build.
3. Create a binding with:
   * `type` equal to `dependency-mapping`
   * A key/value pair where the key is equal to the `sha256` of the dependency and the value is equal to the new URI.
4. Configure all builds with this binding.

## CA Certificates

Additional CA certificates may be added to the system truststore using the [Paketo CA Certificates Buildpack][bp/ca-certificates].

CA certificates can be provided at both build and runtime with a [binding]({{< relref "#bindings" >}}) of `type` `ca-certficates`. Each key value pair in the binding should map a certficate name to a single PEM encoded CA Certficates

```plain
<binding-name>
├── <cert file name>
└── type
```

If a given [language family buildpack][language family buildpacks] does not contain the Paketo CA Certificates Buildpack it can be explicitly prepended at runtime.

**Example**: Adding a CA Certificate at Runtime

The samples repository contains a simple Golang application that will make a `HEAD` request to a provided URL.

Given a file `<your-ca.pem>` containing a single PEM encoded CA certificate needed to verify a TLS connection to <!-- spellchecker-disable --> an https<!-- spellchecker-enable --> URL `<url>`, add the CA certificate to the binding.

```bash
cp <your-ca.pem> ca-certificates/binding/
```

The provided sample contains a simple Golang application that will make a `HEAD` request to a provided URL. Build the application using the CA Certificates buildpack

```bash
pack build samples/ca-certificates \
    --path ca-certificates \
    --buildpack paketo-buildpacks/ca-certificates \
    --buildpack paketo-buildpacks/go
```

Run the sample application, providing the binding, and passing the URL as a positional argument (should print `SUCCESS!`).

```bash
docker run --rm \
  --env SERVICE_BINDING_ROOT=/bindings \
  --volume "$(pwd)/ca-certficates/binding:/bindings/ca-certificates" \
  samples/ca-certificates <url>
```

**Disabling CA Certificates**

If a language family buildpack contains the Paketo CA Certifcates Buildpack,
the CA Certificates Buildpack will always pass detection so that certificates
can be provided dynamically at runtime.

To opt out of this behavior all together, the `BP_ENABLE_RUNTIME_CERT_BINDING`
environment variable can be set to `false` at build-time. This will disable the
ability to set certificates at runtime. The CA Certificates Buildpack will then
only detect if a certificate binding is provided at build-time.


## Applying Custom Labels

Paketo users may add labels to the application image using the [Image Labels Buildpack][bp/image-labels].

Environment variables prefixed with `BP_OCI_` can be used to set [OCI-specific][oci annotation keys]. For example, if `BP_OCI_AUTHORS` is set at build-time, the Image Labels Buildpack will add a label to the image with key `org.opencontainers.image.authors` and value equal to the value of `$BP_OCI_AUTHORS`.

Users may contribute arbitrary labels by providing a collection of space-delimited key-value pairs with the `BP_IMAGE_LABELS` environment variable. Values containing spaces can be quoted.

If a given [language family buildpack][language family buildpacks] does not contain the Image Labels Buildpack it can be explicitly appended at runtime.

**Example**: Adding Custom Labels
{{< code/copyable >}}
pack build samples/nodejs \
  --path nodejs/yarn \
  --buildpack paketo-buildpacks/nodejs \
  --buildpack paketo-buildpacks/image-labels \
  --env "BP_OCI_DESCRIPTION=Demo Application" \
  --env 'BP_IMAGE_LABELS=io.packeto.example="Adding Custom Labels"'
docker inspect samples/nodejs | jq '.[].Config.Labels["org.opencontainers.image.description"]' # should print "Demo Application"
docker inspect samples/nodejs | jq '.[].Config.Labels["io.packeto.example"]' # should print "Adding Custom Labels"
{{< /code/copyable >}}

## Configuring Locale

By default, an image created using Paketo buildpacks will not have a specific locale set. If you run `locale`, you'll end up with these settings:

```bash
LANG=
LANGUAGE=
LC_CTYPE="POSIX"
LC_NUMERIC="POSIX"
LC_TIME="POSIX"
LC_COLLATE="POSIX"
LC_MONETARY="POSIX"
LC_MESSAGES="POSIX"
LC_PAPER="POSIX"
LC_NAME="POSIX"
LC_ADDRESS="POSIX"
LC_TELEPHONE="POSIX"
LC_MEASUREMENT="POSIX"
LC_IDENTIFICATION="POSIX"
LC_ALL=
```

If you wish to set a locale, you may do so when you run the image by setting the corresponding environment variable. For example, with Docker one could execute `docker run -e LANG=en_US.utf8 ...` to change the locale.

This isn't always necessary but can impact output from your application. For example if you have an application that writes unicode characters to STDOUT/STDERR and you go to view those, possibly with `docker logs`, they will not display correctly unless you have a locale set that supports unicode, like UTF-8 in the example above.

<!-- buildpacks -->
[bp/ca-certificates]:https://github.com/paketo-buildpacks/ca-certificates
[bp/image-labels]:https://github.com/paketo-buildpacks/image-labels
[bp/procfile]:https://github.com/paketo-buildpacks/procfile
[bp/bellsoft-liberica]:https://github.com/paketo-buildpacks/bellsoft-liberica
[bp/bellsoft-liberica/descriptor]:https://github.com/paketo-buildpacks/bellsoft-liberica/blob/main/buildpack.toml
[bp/environment-variables]:https://github.com/paketo-buildpacks/environment-variables

[samples]:https://github.com/paketo-buildpacks/samples

<!-- paketo docs references -->
[base builder]:{{< ref "/docs/concepts/builders#base" >}}
[language family buildpacks]:{{< ref "/docs/concepts/buildpacks" >}}

<!-- cnb references -->
[pack]:https://github.com/buildpacks/pack
[platforms]:https://buildpacks.io/docs/concepts/components/platform/

<!-- other references -->
[cnb bindings deprecation]:https://github.com/buildpacks/rfcs/blob/main/text/0055-deprecate-service-bindings.md
[cnb bindings]:https://github.com/buildpacks/spec/blob/main/extensions/bindings.md
[k8s service bindings]:https://github.com/k8s-service-bindings/spec
[kpack service bindings]:https://github.com/pivotal/kpack/blob/master/docs/servicebindings.md#service-bindings
[liberica releases]:https://github.com/bell-sw/Liberica/releases
[maven settings]:https://maven.apache.org/settings.html
[maven opts]:https://maven.apache.org/configure.html#maven_opts-environment-variable
[oci annotation keys]:https://github.com/opencontainers/image-spec/blob/master/annotations.md#pre-defined-annotation-keys
[spring cloud bindings]:https://github.com/spring-cloud/spring-cloud-bindings
