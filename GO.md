# Versioning

The Go version to use for each module is specified with the [`go` directive of
the `go.mod` file](https://go.dev/doc/modules/gomod-ref#go).

For example:

```bash
go 1.24.4
```

You can programmatically retrieve that version number with `goenv`:

```bash
GOENV_GOMOD_VERSION_ENABLE=1 goenv local
```

## Implementation details

If you are interested in the details of how the CI handles this, please refer
to [`golang_install.sh`](https://github.com/swissgeo/infra-terraform/blob/main/scripts/codebuild/golang_install.sh)
and the [`goenv`](https://github.com/go-nv/goenv) documentation.

It should be possible to override the Go version specified in `go.md` with a
`.go-version` file. This is discouraged as different tools may then have
different ideas about which version of Go to use.

While the `go.mod` documentation claims the `go` directive is meant to be the
minimum version and it discourages from specifying the patch level (only
`minor.major` version), in practice some tools treat it as the actual version
and others are confused about the lack of patch level. Thus
[we made the decision](https://swissgeoplatform.atlassian.net/wiki/x/BACDBw) to
treat it as the actual version and to specify the patch level.
