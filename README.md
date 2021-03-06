# x-gae-dep

This project illustrates usage of `https://github.com/golang/dep` tool (and
vendoring) in Google App Engine project.

It is created to try approach described at
https://stackoverflow.com/a/40118834/2063744.

Tested in the environment as follows:

```console
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.3 LTS
Release:	16.04
Codename:	xenial
$ go version
go version go1.9.2 linux/amd64
$ dep version
dep:
 version     : devel
 build date  : 
 git hash    : 
 go version  : go1.9.2
 go compiler : gc
 platform    : linux/amd64
$ goapp version
go version 1.8.3 (appengine-1.9.63) linux/amd64
$ gcloud version
Google Cloud SDK 182.0.0
app-engine-go 
app-engine-python 1.9.63
bq 2.0.27
core 2017.12.01
gsutil 4.28
```

To use `dep` and `goapp` you'll need to setup worksapce (GOPATH, GOROOT, PATH
environment variables for current project). Easy way to do it is via `source`
shell command. Fix paths, according to your local directory layout in the
`./appengine-env` file and use it to setup environment in a new shell.

```console
$ source ./appengine-env
## dep init/ensure/status:
## - default service
$ cd src/def
$ dep ensure
## - service A
$ cd ../a
$ dep ensure
## - service B
$ cd ../b
$ dep ensure
## dep status:
$ dep status
$ dep status -dot | dot -T png | display
## Run app locally:
$ cd ../..
$ goapp version
## Build service (check that it has no compilation errors):
$ goapp build gaedep/def gaedep/a gaedep/b
## Serve all modules:
$ goapp serve ./appengine/def/app.yaml ./appengine/a/a.yaml ./appengine/b/b.yaml
## Deploy app. Note: gcloud tool is currently broken (attempts to deploy vendor dir), goapp works.
## gcloud app deploy --project xgaedep ./appengine/def/app.yaml ./appengine/a/a.yaml ./appengine/b/b.yaml
$ goapp deploy -application xgaedep -version v1 ./appengine/def/app.yaml ./appengine/a/a.yaml ./appengine/b/b.yaml
```

**NOTE:** I used `goapp` tool in the snippet above. Google depricated it, but it still can be found within SDK.
In this project's `./appengine-env` the path to the SDK platform tools is hardcoded, but if you want to make
more portable script, you may try a trick I used in another my project:

```console
# This snippet is from Mac OS X, `greadlink` is a GNU `readlink`,
# installed with `brew install coreutils`.
# On Linux it is just `readlink`. You may create an alias or bash function to unify them.

# find full path to `gcloud` tool, it's used to calculate path to platfrom tools
path_gcloud=$(greadlink -f $(command -v gcloud))

# replace `bin/gcloud` substring with `platform/google_appengine/goroot-1.8`
export GOROOT=${path_gcloud/%bin\/gcloud/platform\/google_appengine\/goroot-1.8}

# add to `$PATH`
export PATH=$GOROOT/bin:$PATH

# use project's root dir as `$GOPATH`
export GOPATH=$(greadlink -f .)
```

**NOTE:** `gcloud app deploy` is currently broken (in regard to vendoring),
see [the issue](https://issuetracker.google.com/issues/38449183).
Workaround, suggested in the discussion of the issue, is a bit ugly, but the only option worked for me:

- move `vendor` dir(s) from the `$GOPATH/src` directory (subdirectories in case of multiple modules
  like in this sample) into temporary dir;
- setup a `trap` to move them back after deployment;
- setup `$GOPATH` for `gcloud app deploy` so that it contain temporary dir(s) with vendored dependencies;
- execute `gcloud app deploy`, directories should be restored after script finished.

Here is a snippet to illustrate the idea, but for single module (adopted from another my project):

```console
TMP_VENDOR=$(mktemp -d "${TMPDIR:-/tmp}"/deploy-gopath.XXXX)
mv ./src/gaedep/a/vendor "$TMP_VENDOR"/src
export GOPATH=$GOPATH:$TMP_VENDOR
function move_vendor_back() {
    mv "$TMP_VENDOR"/src ./src/gaedep/a/vendor
    rm -rf "$TMP_VENDOR"
}
trap move_vendor_back EXIT
gcloud app deploy ...
```

# Links to the test app

- https://xgaedep.appspot.com/
- https://a-dot-xgaedep.appspot.com/
- https://b-dot-xgaedep.appspot.com/

# Useful links

- https://github.com/golang/dep
- https://stackoverflow.com/a/40118834/2063744
- https://cloud.google.com/appengine/docs/flexible/go/configuration-files
- https://cloud.google.com/appengine/docs/standard/python/designing-microservice-api
- https://issuetracker.google.com/issues/38449183
