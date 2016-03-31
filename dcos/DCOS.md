# Gob's Web Service on DC/OS

This documents the steps required to build and deploy Gob on DC/OS, integrated with the linkerd DC/OS package.

All commands are run from the root of this repo.

## Build Images

```bash
docker build -t buoyantio/gobs-project:gensvc -f gob/Dockerfile-gensvc .
docker build -t buoyantio/gobs-project:websvc -f gob/Dockerfile-websvc .
docker build -t buoyantio/gobs-project:wordsvc -f gob/Dockerfile-wordsvc .
```

## Push Images

```bash
docker push buoyantio/gobs-project:gensvc
docker push buoyantio/gobs-project:websvc
docker push buoyantio/gobs-project:wordsvc
```

## Linkerd/Namerd Package Install

### DC/OS Package Config

#### linkerd

`linkerd-package-options.json` is the linkerd DC/OS package install configuration. Package options of note:

- `instances` - size of the DC/OS cluster, to ensure linkerd is installed on each node
- `config-filename` - linkerd runtime configuration filename, appends to `config-uri`
- `config-uri` - linkerd runtime configuration location, prepends to `config-filename`
- `routing-port` - port to route rpc calls. must match the `routers/servers/port` specified in `linkerd-dcos-gob.yaml`

#### namerd

`namerd-package-options.json` is the namerd DC/OS package install configuration. Package options of note:

- `config-filename` - namerd runtime configuration filename, appends to `config-uri`
- `config-uri` - namerd runtime configuration location, prepends to `namerd-filename`
- `http-port` - http control port, must match the `interfaces/httpController` port specified in `namerd-dcos-gob.yaml`
- `thrift-port` - thrift interface port, must match:
  - `interfaces/thriftNameInterpreter` port specified in `namerd-dcos-gob.yaml`
  - `router/interpreter/dst` port specified in `linkerd-dcos-gob.yaml`
- `resource-role` - the role to run namerd instances on.


### linkerd Config

`https://s3.amazonaws.com/buoyant-dcos/linkerd-dcos-gob.yaml` is the linkerd runtime configuration, customized for this Gob example:

- linkerd routes all traffic via port `4140`. Services perform all RPC calls via localhost:4140.
- The `baseDtab` section routes on host headers. Calling the word service looks like this:

    ```bash
    curl -H "Host: word" localhost:4140
    ```

- The `baseDtab` section defaults routing to the `web` service. An internet-facing load balancer should route traffic to localhost:4140 on the DC/OS public node, from there linkerd will route to `web`.

### linkerd/namerd Package Install

For the sake of this demonstration, assume $PUBLIC_URL is set to a public-facing mesos node.

```bash
export PUBLIC_URL=http://example.com
```

Upload you DC/OS config to a publicly accessible URL.

```bash
aws s3 cp dcos/linkerd-dcos-gob.yaml s3://buoyant-dcos --grants read=uri=http://acs.amazonaws.com/groups/global/AllUsers
aws s3 cp dcos/namerd-dcos-gob.yaml s3://buoyant-dcos --grants read=uri=http://acs.amazonaws.com/groups/global/AllUsers
```

Add the linkerd universe repo to DC/OS

```bash
dcos package repo add linkerd https://github.com/buoyantio/universe/archive/siggy/linkerd.zip
```

Initialize Zookeeper path

`namerd-dcos-gob.yaml` specifies a `storage/pathPrefix` for zk storage. Create this path in preparation for namerd installation.

```bash
ssh core@$PUBLIC_URL -L 2181:$PUBLIC_URL:2181 -N
zkCli -server localhost:2181 create /dtabs null
```

Install the packages, via command line, or the DC/OS UI

```bash
dcos package install linkerd --options=dcos/linkerd-package-options.json
dcos package install namerd --options=dcos/namerd-package-options.json
```

## Gob Deploy

```bash
dcos marathon app add dcos/marathon/gensvc.json
dcos marathon app add dcos/marathon/wordsvc.json
dcos marathon app add dcos/marathon/websvc.json
```

## Confirm it all works

### Initialize namerd dtabs

```bash
curl -vX POST -d @dcos/namerd.dtab --header "Content-Type: application/dtab" $PUBLIC_URL:4180/api/1/dtabs/default
open $PUBLIC_URL:4180/api/1/dtabs/default
```

### Update namerd dtabs

```bash
ETAG=`curl -i $PUBLIC_URL:4180/api/1/dtabs/default|grep -Fi ETag: | awk {'print $2'} | sed -e 's/[[:cntrl:]]//'`
curl -vX PUT -d @dcos/namerd.dtab --header "Content-Type: application/dtab" --header "If-Match: $ETAG" $PUBLIC_URL:4180/api/1/dtabs/default
open $PUBLIC_URL:4180/api/1/dtabs/default
```

### Test Gob app

```bash
curl "$PUBLIC_URL:4140/gob?text=foo&limit=2"
```