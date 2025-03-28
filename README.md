# slugcmplr

`slugcmplr` allows you to compile Heroku compatible slugs using official and
custom buildpacks, in the same manner as the [Heroku Slug Compiler](https://devcenter.heroku.com/articles/slug-compiler)

This enables more control over the build process and allows for
detaching building and releasing your Heroku applications.

## Install

With the `go` toolchain installed:
```bash
# In order to use this as a library
go get github.com/cga1123/slugcmplr

# In order to install as a CLI
go install github.com/cga1123/slugcmplr@latest
```

There are also precompiled binaries and their associated checksums are
available attached to tagged [releases].

## Info

`slugcmplr` has 3 steps/sub-commands:

#### `prepare [APPLICATION] --build-dir [BUILD-DIR] --source-dir [SOURCE-DIR]`

In the prepare step, `slugcmplr` will fetch the metadata required to compile
your application. It will copy your project `SOURCE-DIR` into `BUILD-DIR/app`.

It will fetch the buildpacks as defined by your Heroku application, download, and
decompress them into `BUILD-DIR/buildpacks`. If using official buildpacks (e.g.
`heroku/go`, `heroku/ruby`) `slugcmplr` will use the same buildpack as
currently deployed to Heroku's production environment.

It will fetch the config vars as defined by your Heroku application and put
them into the `BUILD-DIR/environment` directory.

Finally, `prepare` writes metadata (such as the application name, stack,
buildpack order, source version) to the `BUILD-DIR/compile.json` file to allow
the `compile` step to bootstrap itself.

#### `compile --build-dir [BUILD-DIR] --cache-dir [CACHE-DIR]`

In the compile step, `slugcmplr` executes your buildpacks in the specified
order, outputs your Heroku slug, and uploads it to Heroku for future release.

`compile` will output the slug for your application to `BUILD-DIR/app.tgz`

`compile` will output metadata about the compilation to
`BUILD-DIR/release.tgz`, this contains information such as the slug ID as
uploaded to Heroku.

The `CACHE-DIR` will be used by the buildpacks as their cache argument to speed
up builds in the future, as per the [Buildpack API](https://devcenter.heroku.com/articles/buildpack-api)

To guarantee full compatibility, it is recommended to run this step using
Heroku's build containers. e.g. `heroku/heroku:24-build`.

#### `release --build-dir [BUILD-DIR]`

In the release step, `slugcmplr` triggers a release of your previously compiled
slug.

It uses the `BUILD-DIR/release.json` file in order to fetch metadata in order
to create this release.

You can optionally pass `--app [APPLICATION]` to target an application that is
different from the one you built from. This will work as long as the
applications are in the same Heroku team. (This is becuase the slug must be
accessible to the application).

You can optionally pass `--commit [COMMIT]` to associate this release with a
separate commit from the one used to build this slug initially.

## Authentication

The `slugcmplr` CLI looks for credentials to `api.heroku.com` in your `.netrc`
file, this is the same technique used by [`heroku/cli`] and so if you are
currently making use of the `heroku` command during CI, you should already be
logging in somehow and have no issues.

Otherwise populating your `.netrc` should be a case of adding something
equivalent to the following script (assuming the `HEROKU_EMAIL` and
`HEROKU_API_KEY` are correctly populated environment variables):

```bash
cat << EOF >> ${HOME}/.netrc
machine api.heroku.com
  login ${HEROKU_EMAIL}
  password ${HEROKU_API_KEY}
EOF
```

By default, `slugcmplr` will look in `${HOME}/.netrc` for the credentials,
however it will respect the `${NETRC}` environment variable if set and
non-empty.

## Testing

The majority of tests for this project are acceptance tests that will create
and release live Heroku applications. These require the correct credentials to
be set in the environment as well as setting a sentinel value to execute the
acceptance tests:

```bash
SLUGCMPLR_ACC=true \
  SLUGCMPLR_ACC_HEROKU_PASS=<HEROKU-API-KEY> \
  SLUGCMPLR_ACC_HEROKU_EMAIL=<HEROKU-EMAIL> \
  go test -v
```

Fixture application are hosted in separate repositories which will be cloned
and created by using the `withHarness` function. Fixture repositories are
expected to contain a `app.json` file which describes the Heroku applications.

See [app.json Schema](https://devcenter.heroku.com/articles/app-json-schema)

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/CGA1123/slugcmplr

See [issues](https://github.com/CGA1123/slugcmplr/issues) if you want any
inspiration as to what to help with or of any pending discussion/work.

This project is Codespaces compatible! If you want to get started quickly spin one up.

The `script` directory should help you get going with building and testing.

Some buildpacks are not compatible with `arm`/`arm64` architectures, you may experience
issues testing `slugcmplr` on an M1 Chip, even if using docker containers.

---

For more background on this you might find [this] Medium article helpful.

[this]: https://medium.com/carwow-product-engineering/speeding-up-our-heroku-deploys-by-35-percent-f9fa6f6cf404
[`heroku/cli`]: https://github.com/heroku/cli
[releases]: https://github.com/cga1123/slugcmplr/releases
