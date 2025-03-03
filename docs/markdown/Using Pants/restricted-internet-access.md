---
title: "Restricted Internet access"
slug: "restricted-internet-access"
excerpt: "How to use Pants when you have restricted access to the Internet"
hidden: false
createdAt: "2020-10-23T19:49:45.143Z"
---
Some organizations place restrictions on their users' Internet access, for security or compliance reasons.  Such restrictions may prevent Pants from downloading various underlying tools it uses, and it may interfere with bootstrapping Pants itself. 

In such cases, users are typically still able to access internal proxies and servers. This page shows how to configure Pants to work smoothly in these circumstances.

Installing Pants
----------------

The `pants` script from [Installing Pants](doc:installation) uses PyPI to download and install the wheel `pantsbuild.pants` and all of Pants's dependencies. 

If you cannot access PyPI directly, you may have an internal mirror or custom Python package repository. If so, you can ensure that `pantsbuild.pants` and all of its dependencies are available in that repository, and modify your `pants` script to bootstrap from it.

Otherwise, you may instead download Pants as a PEX binary from <https://github.com/pantsbuild/pants/releases>. After downloading the PEX artifact, you can rename the file to `pants`, run `chmod +x pants`, then run `pants --version` like you normally would. 

You may want to check the binary into version control so that everyone in your organization can use it. To upgrade to a new Pants release, update the `pants_version` option in `pants.toml` and download the newest release from <https://github.com/pantsbuild/pants/releases>.

Setting up a Certificate Authority
----------------------------------

By default, Pants will respect and pass through the `SSL_CERT_DIR` and `SSL_CERT_FILE` environment variables.

If you need to override those values, you can configure Pants to use a custom Certificate Authority (CA) bundle:

```toml pants.toml
[GLOBAL]
ca_certs_path = "/path/to/certs/file"
```

Setting `HTTP_PROXY` and `HTTPS_PROXY`
--------------------------------------

You may need to set standard proxy-related environment variables, such as `http_proxy`, `https_proxy` and `all_proxy`, in executed subprocesses:

```toml pants.toml
[subprocess-environment]
env_vars.add = ["http_proxy=http://myproxy", "https_proxy"]
```

You may need to use lowercase or uppercase env var names, or both.

Note that if you leave of the env var's value, as for `https_proxy` above, Pants will use the value of the same variable in the environment in which it is invoked.

Customizing tool download locations
-----------------------------------

There are three types of tools that Pants may need to download and invoke:

- **Python tools**: these are resolved from a package repository (PyPI by default) via requirement strings such as `mypy==0.910`.
- **JVM tools**: these are resolved from a package repository (Maven Central by default) via coordinates such as `org.scalatest:scalatest_2.13:3.2.10`.
- **Standalone binaries**: these are downloaded from a configured URL and verified against a SHA256 hash.

If you cannot access these resources from their default locations, you can customize those locations.

You can get a list of the tools Pants uses, in all three categories, with `pants help tools`. 

### Python tools

Pants downloads the various Python-related tools it uses from [PyPI](https://pypi.org/), just as it does for your Python code's dependencies. 

If you use Python but cannot access PyPI directly, then you probably have an internal mirror or a custom Python package repository.  So all you have to do is configure Pants to access this custom repository, and ensure that the tools it needs are available there.

See [Python third-party dependencies](doc:python-third-party-dependencies#custom-repositories) for instructions on how to set up Pants to access a custom Python package repository. 

### JVM tools

Pants downloads the various JVM-related tools it uses from [Maven Central](<>), just as it does for your JVM code's dependencies.

If you use JVM code but cannot access Maven Central directly, then you probably have an internal mirror or a custom JVM package repository. So all you have to do is configure Pants to access this custom repository, and ensure that the tools it needs are available there. 

To do so, set the [`repos`](doc:reference-coursier#section-repos) option on the `[coursier]` scope. E.g., 

```text pants.toml
[coursier]
repos = ["https://my.custom.repo/maven2"]
```

### Binary tools

Pants downloads various binary tools from preset locations, and verifies them against a SHA. If you are not able to allowlist these locations, you can host the binaries yourself and instruct Pants to use the custom locations. 

You set these custom locations by setting the `url_template` option for the tool. In this URL template, Pants will replace `{version}` with the requested version of the tool and `{platform}`, with the platform name (e.g., `linux.x86_64`). 

The platform name used to replace the `{platform}` placeholder can be modified using the `url_platform_mapping` option for the tool. This option maps a canonical platform name (`linux_arm64`, `linux_x86_64`, `macos_arm64`, `macos_x86_64`) to the name that should be substituted into the template. 

This is best understood by looking at an example:

`pants help-advanced protoc` (or its [online equivalent](doc:reference-protoc#advanced-options)) shows that the default URL template is `https://github.com/protocolbuffers/protobuf/releases/download/v{version}/protoc-{version}-{platform}.zip`. 

- We see the `version` option is set to `3.11.4`. 
- We are running on macOS ARM, so look up `macos_arm64` in the `url_platform_mapping` option and find the string `osx-x86_64`. 

Thus, the final URL is:  
`https://github.com/protocolbuffers/protobuf/releases/download/v3.11.4/protoc-3.11.4-osx-x86_64.zip`.

It should be clear from this example how to modify the URL template to point to your own hosted binaries:

```python pants.toml
[protoc]
url_template = "https://my.custom.host/bin/protoc/{version}/{platform}/protoc.zip"
```

For simplicity, we used the original value for `url_platform_mapping`, meaning that we set up our hosted URL to store the macOS x86 binary at `.../osx-x86_64/protoc.zip`, for example. You can override the option `url_platform_mapping` if you want to use different values.

Occasionally, new Pants releases will upgrade to new versions of these binaries, which will be mentioned in the "User API Changes" part of the changelog <https://github.com/pantsbuild/pants/tree/master/src/python/pants/notes>. When upgrading to these new Pants releases, you should download the new artifact and upload a copy to your host.

> 📘 Asking for help
> 
> It's possible that Pants does not yet have all the mechanisms it'll need to work with your organization's specific networking setup, which we'd love to fix.
> 
> Please reach out on [Slack](doc:the-pants-community) or open a [GitHub issue](https://github.com/pantsbuild/pants/issues) for any help.
