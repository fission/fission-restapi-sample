# RESTful API in Golang

In this sample, we create a simple guestbook application to demonstrates how to create a set of CURD RESTful APIs with fission function.

## Installation

1. Add the cockroachdb helm chart

```bash
helm repo add cockroachdb https://charts.cockroachdb.com/
helm repo update
```

2. Install cockroachdb operator using helm

```bash
helm install my-release --values cockroachdb.yaml cockroachdb/cockroachdb
```

3. Zip the files in the rest-api folder

```bash
zip -j restapi-go-pkg.zip rest-api/*
```

4. Use `fission spec` to create application

```bash
fission spec apply
```

5. Check application status

After `fission spec apply`, you can check that functions, http triggers and package are successfully created.

```bash
$ fission spec list
Functions:
NAME           ENV EXECUTORTYPE MINSCALE MAXSCALE MINCPU MAXCPU MINMEMORY MAXMEMORY TARGETCPU SECRETS CONFIGMAPS
restapi-delete go  newdeploy    1        3        0      0      0         0         80                
restapi-get    go  newdeploy    1        3        0      0      0         0         80                
restapi-post   go  newdeploy    1        3        0      0      0         0         80                
restapi-update go  newdeploy    1        3        0      0      0         0         80                

Environments:
NAME IMAGE               BUILDER_IMAGE           POOLSIZE MINCPU MAXCPU MINMEMORY MAXMEMORY EXTNET GRACETIME
go   fission/go-env-1.16 fission/go-builder-1.16 3        0      0      0         0         false  0

Packages:
NAME           BUILD_STATUS ENV LASTUPDATEDAT
restapi-go-pkg succeeded    go  04 Mar 22 14:31 IST

HTTP Triggers:
NAME        METHOD   URL                             FUNCTION(s)    INGRESS HOST PATH                            TLS ANNOTATIONS
restdelete  [DELETE] /guestbook/messages/{id:[0-9]+} restapi-delete false   *    /guestbook/messages/{id:[0-9]+}     
restget     [GET]    /guestbook/messages/            restapi-get    false   *    /guestbook/messages/                
restgetpart [GET]    /guestbook/messages/{id:[0-9]+} restapi-get    false   *    /guestbook/messages/{id:[0-9]+}     
restpost    [POST]   /guestbook/messages             restapi-post   false   *    /guestbook/messages                 
restupdate  [PUT]    /guestbook/messages/{id:[0-9]+} restapi-update false   *    /guestbook/messages/{id:[0-9]+}     

```

If the build status of the package shows failed, try rebuilding it.

```bash
fission pkg rebuild --name restapi-go-pkg
```

## Usage

1. Create a post

```bash
$ curl -v -X POST \
    http://${FISSION_ROUTER}/guestbook/messages \
    -H 'Content-Type: application/json' \
    -d '{"message": "hello world!"}'
```

2. Get all posts/single post

```bash
curl -v -X GET http://${FISSION_ROUTER}/guestbook/messages/
```

You should see a list posts are returned.

```json
[
    {
        "id": 366739357484417025,
        "message": "hello world!",
        "timestamp": 1531990369
    },
    {
        "id": 366739413774237697,
        "message": "hello world!",
        "timestamp": 1531990387
    },
    {
        "id": 366739416644550657,
        "message": "hello world!",
        "timestamp": 1531990399
    }
]
```

Now, let's try to get a single post with post `id`.

```bash
curl -v -X GET http://${FISSION_ROUTER}/guestbook/messages/366456868654284801
```

Or, you can try to get the messages in a specific time range with `start` and `end`.

```bash
curl -X GET 'http://${FISSION_ROUTER}/guestbook/messages/?start=1531990369&end=1531990387'
```

3. Update post

```bash
$ curl -v -X PUT \
    http://${FISSION_ROUTER}/guestbook/messages/366456868654284801 \
    -H 'Content-Type: application/json' \
    -d '{"message": "hello world again!"}'
```

4. Delete post

```bash
$ curl -X DELETE \
    http://${FISSION_ROUTER}/guestbook/messages/366456868654284801 \
    -H 'Cache-Control: no-cache'
```

## Spec generation

```console
fission spec init
fission env create --name go --image fission/go-env-1.16 --builder fission/go-builder-1.16 --spec
fission pkg create --name restapi-go-pkg --src restapi-go-pkg.zip --env go --spec
fission fn create --name restapi-delete --executortype newdeploy --maxscale 3 --env go --pkg restapi-go-pkg --entrypoint MessageDeleteHandler --spec
fission fn create --name restapi-update --executortype newdeploy --maxscale 3 --env go --pkg restapi-go-pkg --entrypoint MessageUpdateHandler --spec
fission fn create --name restapi-post --executortype newdeploy --maxscale 3 --env go --pkg restapi-go-pkg --entrypoint MessagePostHandler --spec
fission fn create --name restapi-get --executortype newdeploy --maxscale 3 --env go --pkg restapi-go-pkg --entrypoint MessageGetHandler --spec
fission httptrigger create --url /guestbook/messages --method POST --function restapi-post --spec --name restpost
fission httptrigger create --url "/guestbook/messages/{id:[0-9]+}" --method PUT --function restapi-update --spec --name restupdate
fission httptrigger create --url "/guestbook/messages/{id:[0-9]+}" --method GET --function restapi-get --spec --name restgetpart
fission httptrigger create --url "/guestbook/messages/{id:[0-9]+}" --method DELETE --function restapi-delete --spec --name restdelete
fission httptrigger create --url "/guestbook/messages/" --method GET --function restapi-get --spec --name restget
```
