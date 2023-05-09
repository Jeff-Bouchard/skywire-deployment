# Skywire Deployment

This repository contains submodules of all the repositories used for skywire deployment

* [skywire](https://github.com/skycoin/skywire)
* [dmsg](https://github.com/skycoin/dmsg)
* [skywire-utilities](https://github.com/skycoin/skywire-utilities)
* [skywire-services](https://github.com/skycoin/skywire-services)
* [skycoin-service-discovery](https://github.com/skycoin/skycoin-service-discovery)

## Table of Contents

* [Skywire Deployment](#skywire-deployment)
   * [Table of Contents](#table-of-contents)
   * [Initialize the repo](#initialize-the-repo)
   * [Code Checks &amp; Tests](#code-checks--tests)
   * [Building](#building)
   * [Runtime Dependencies by Service](#runtime-dependencies-by-service)
      * [Redis](#redis)
         * [Redis setup](#redis-setup)
      * [Postgres](#postgres)
         * [Postgres DB Setup](#postgres-db-setup)
   * [Required Services](#required-services)
   * [Key generation for services](#key-generation-for-services)
      * [Visor Config Bootstrap Endpoint](#visor-config-bootstrap-endpoint)
   * [SKYCOIN-SERVICE-DISCOVERY Setup](#skycoin-service-discovery-setup)
   * [service-discovery](#service-discovery)
   * [DMSG Setup](#dmsg-setup)
      * [dmsg-discovery](#dmsg-discovery)
      * [dmsg-server](#dmsg-server)
         * [Configure dmsg-server](#configure-dmsg-server)
         * [Run dmsg-server](#run-dmsg-server)
   * [SKYWIRE-SERVICES Setup](#skywire-services-setup)
      * [address-resolver](#address-resolver)
      * [route-finder](#route-finder)
      * [transport-discovery](#transport-discovery)
      * [network-monitor](#network-monitor)
   * [SKYWIRE Setup](#skywire-setup)
      * [Route setup-node](#route-setup-node)
      * [skywire-visor](#skywire-visor)
   * [Using Dmsg to connect to the deployment](#using-dmsg-to-connect-to-the-deployment)
   * [HTTP Service Configuration](#http-service-configuration)

## Initialize the repo

This repo contains all necessary repos to deploy Skywire as submodules. To set the source code up locally, 
make sure you have `make` and `git` installed on your machine. 

Clone this repo and all its submodules with:

```
git clone --recurse-submodules https://github.com/skycoin/skywire-deployment.git
```

## Code Checks & Tests

Dependencies:
* [go](https://go.dev/)
* [goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports)
* [goimports-reviser](https://github.com/incu6us/goimports-reviser)

For any given submodule repository, included is a Makefile with directives such as `format` and `check`.
`make format check` is currently used by the CI to check pull requests.

`make format check` executes a variation of the following commands:
```
go mod tidy -v
[[ -d pkg ]] && goimports -w -local github.com/skycoin/$(basename $(pwd)) ./pkg
[[ -d cmd ]] && goimports -w -local github.com/skycoin/$(basename $(pwd)) ./cmd
[[ -d internal ]] && goimports -w -local github.com/skycoin/$(basename $(pwd)) ./internal
find . -type f -name '*.go' -not -path "./.git/*" -not -path "./vendor/*"  -exec goimports-reviser -project-name github.com/skycoin/$(basename $(pwd)) {} \;
go clean -testcache &>/dev/null
[[ -d internal ]] && go test  -cover -timeout=5m -mod=vendor ./internal/...
[[ -d pkg ]] && go test -cover -timeout=5m -mod=vendor ./pkg/...
```

The flags used for `go test` may vary from repo to repo

## Building

Dependencies:
* [go](https://go.dev/)

For any given submodule repository (except skywire-utilities) included is a Makefile with a `build` directive.

`make build` executes `go build` for all the binaries that can be built from .go files in `./cmd` subdirectories, for example

```
cd skywire
go build ./cmd/skywire-visor
cd ..
```

OR, using `go run` instead

```
cd skywire
go run ./cmd/skywire-visor --help
go run ./cmd/skywire-cli --help
```

Note that some binaries may have gcc / libc6 dependency unless statically compiled with musl.

## Runtime Dependencies by Service

### Redis
* address-resolver
* transport-discovery
* network-monitor
* dmsg-discovery
* service-discovery

#### Redis setup

Redis setup simply entails installing redis and starting the service.

### Postgres
* transport-discovery
* route-finder
* service-discovery

#### Postgres DB Setup

Some services use postgresql. The database setup is included in this document for each service which requires it.

As a brief overview of the process, one must run postgresql, and make a database with UTF-8 character-set. The name of the database is inconsequential. The database name as well as the postgres credentials are specified via environmental variables

__Pass the database, user, and password as envs__
```
export PG_USER=username
export PG_PASSWORD=pass
export PG_DATABASE=sampledb
```

__Specify the postgres host and port via flags__
```
route-finder --pg-host localhost --pg-port 5432
```

__All tables are created automatically.__


## Required Services

* address-resolver
* network-monitor
* dmsg-discovery
* dmsg-server
* transport-discovery
* route-finder
* service-discovery
* setup-node

A list of endpoints corresponding to some of these services in the current deployment is provided here for reference:

* sd.skycoin.com/api/services?type=proxy
* ar.skywire.skycoin.com/transports
* tpd.skywire.skycoin.com/all-transports
* dmsgd.skywire.skycoin.com/dmsg-discovery/entries
* rf.skywire.skycoin.com/


## Key generation for services

Public/secret keypairs for all services have identical format can can be used interchangeably. There are multiple ways of potentially generating these, for example with skywire-cli:

```
skywire-cli config gen -n | head -n4 | tail -n2
```

A utility called `keys-gen` is included with skywire-services and can be used, which prints the public and secret key.

The keypair can be written to a file and used by a service which accepts it in the following way with keys-gen

```
keys-gen | tee dmsgd-config.json
dmsg-discovery --sk $(tail -n1 dmsgd-config.json)
```

### Visor Config Bootstrap Endpoint

The following endpoint is queried by the visor on config gen:
https://conf.skywire.skycoin.com/

This endpoint contains json which will become part of the visor's config.
The following file is created manually to reflect your deployment:
```
{
  "dmsg_discovery": "http://dmsgd.skywire.skycoin.com",
  "transport_discovery": "http://tpd.skywire.skycoin.com",
  "address_resolver": "http://ar.skywire.skycoin.com",
  "route_finder": "http://rf.skywire.skycoin.com",
  "setup_nodes": [
    "0324579f003e6b4048bae2def4365e634d8e0e3054a20fc7af49daf2a179658557"
  ],
  "uptime_tracker": "http://ut.skywire.skycoin.com",
  "service_discovery": "http://sd.skycoin.com",
  "stun_servers": [
    "139.162.12.30:3478",
    "170.187.228.181:3478",
    "172.104.161.184:3478",
    "170.187.231.137:3478",
    "143.42.74.91:3478",
    "170.187.225.78:3478",
    "143.42.78.123:3478",
    "139.162.12.244:3478"
  ],
  "dns_server": "1.1.1.1"
}
```

This endpoint is used to avoid relying on hardcoded defaults for the production deployment endpoints in the visor's config.
Without it, any changes to the deployment would require an updated version of skywire to manifest correct config default values or significant user intervention.

A deployment of _all services as a set_ will use this for visors to query the endpoint during `config gen` to get the services for the custom deployment, otherwise it is not strictly required.

## SKYCOIN-SERVICE-DISCOVERY Setup

This section deals with services which can be built from the [skycoin/skycoin-service-discovery](https://github.com/skycoin/skycoin-service-discovery) repo

To build all the binaries

```
[[ ! -d skycoin-service-discovery/.git ]] && rm -rf skycoin-service-discovery || true
[[ ! -d skycoin-service-discovery ]] git clone https://github.com/skycoin/skycoin-service-discovery
cd skycoin-service-discovery
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go build ./cmd/service-discovery/service-discovery.go
```

## `service-discovery`
[service-discovery](https://github.com/skycoin/skycoin-service-discovery/tree/develop/cmd/service-discovery)

_Note: this service requires redis_

_Note: this service requires postgresql & initial DB setup_
```
sudo -iu postgres createdb sd
```
Run the Service Discovery server
```
keys-gen | tee sd-config.json
PG_USER="postgres" PG_DATABASE="sd" PG_PASSWORD="" service-discovery  --addr ":9098" --redis "redis://localhost:6379" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 sd-config.json)
```

Example `go run` the Service Discovery server from source in a bash shell.
__Note: the keys-gen command is expected to be present in the executable PATH of this environment!__
```
rm -rf skycoin-service-discovery
git clone https://github.com/skycoin/skycoin-service-discovery
cd skycoin-service-discovery
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
keys-gen | tee sd-config.json
go run cmd/service-discovery/service-discovery.go  --addr ":9098" --redis "redis://localhost:6379" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 sd-config.json)
```

## DMSG Setup

This section deals with services which can be built from the [skycoin/dmsg](https://github.com/skycoin/dmsg) repo

To build all the binaries

```
[[ ! -d dmsg/.git ]] && rm -rf dmsg || true
[[ ! -d dmsg ]] git clone https://github.com/skycoin/dmsg
cd dmsg
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go build cmd/dmsg-discovery/dmsg-discovery.go & \
go build cmd/dmsgget/dmsgget.go & \
go build cmd/dmsgpty-cli/dmsgpty-cli.go & \
go build cmd/dmsgpty-host/dmsgpty-host.go & \
go build cmd/dmsgpty-ui/dmsgpty-ui.go & \
go build cmd/dmsg-server/dmsg-server.go
wait
```

optionally, any of those can be `go run` without explicitly compiling

### `dmsg-discovery`
[dmsg-discovery](https://github.com/skycoin/dmsg/tree/develop/cmd/dmsg-discovery)

_Note: this service requires redis_

Run the Dmsg Discovery server
```
keys-gen | tee dmsgd-config.json
dmsg-discovery --addr ":9090" --redis "redis://localhost:6379" --sk $(tail -n1 dmsgd-config.json)
```

Example `go run` the Dmsg Discovery server from source  in a bash shell

```
rm -rf dmsg
git clone https://github.com/skycoin/dmsg
cd dmsg
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go run cmd/dmsg-server/dmsg-server.go config gen -o dmsgd-conf.json ; head -n3 dmsgd-conf.json | tail -n2 | cut -d '"' -f4 | tee dmsgd-config.json ; rm dmsgd-conf.json
go run cmd/dmsg-discovery/dmsg-discovery.go --addr ":9090" --redis "redis://localhost:6379" --sk $(tail -n1 dmsgd-config.json)
```

### `dmsg-server`
[dmsg-server](https://github.com/skycoin/dmsg/tree/develop/cmd/dmsg-server)

#### Configure `dmsg-server`
Generate a config for the dmsg server
```
dmsg-server config gen -o dmsg-config.json
```
output
```
{
	"public_key": "02a4072022716ee7fb8205edd4286dc73abe514ff8a7a74c6a27131c6efbe71876",
	"secret_key": "62bcd18c5693ad1d2a24239f7303a84d6afb6966340fc70afe93579987cceb90",
	"discovery": "http://dmsgd.skywire.skycoin.com",
	"public_address": "127.0.0.1:8081",
	"local_address": ":8081",
	"health_endpoint_address": ":8082",
	"log_level": "info",
	"update_interval": 0,
	"max_sessions": 2048
}
```
__The IP address must be a public ip address.__

__The port on which the dmsg server is running must be forwarded or otherwise accessible for public servers.__

The above "discovery" endpoint should be changed to match the endpoint of the dmsg discovery server referenced in the previous section.
The default ports are shown.

#### Run `dmsg-server`

Run the dmsg server
```
dmsg-server start dmsg-config.json
```

Example `go run` the Dmsg server from source  in a bash shell
Please note that `git`, `jq`, `curl`, `sed` and `cut` are used in the following example.

```
rm -rf dmsg
git clone https://github.com/skycoin/dmsg
cd dmsg
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go run cmd/dmsg-server/dmsg-server.go config gen -o dmsg-config.json ; sed -i -e 's/dmsgd.skywire.skycoin.com/127.0.0.1:9090/g' -e "s/127.0.0.1:8081/$(curl -L https://ip.skycoin.com | jq '.ip_address' | cut -d '"' -f2):8081/g" dmsg-config.json
go run cmd/dmsg-server/dmsg-server.go start dmsg-config.json
```

## SKYWIRE-SERVICES Setup

This section deals with services which can be built from the [skycoin/skywire-services](https://github.com/skycoin/skywire-services) repo

To build all the binaries

```
[[ ! -d skywire-services/.git ]] && rm -rf skywire-services || true
[[ ! -d skywire-services ]] git clone https://github.com/skycoin/skywire-services
cd skywire-services
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go build ./cmd/address-resolver/address-resolver.go & \
go build ./cmd/config-bootstrapper/config.go & \
go build ./cmd/dmsg-monitor/dmsg-monitor.go & \
go build ./cmd/keys-gen/keys-gen.go & \
go build ./cmd/liveness-checker/liveness-checker.go & \
go build ./cmd/network-monitor/network-monitor.go & \
go build ./cmd/node-visualizer/node-visualizer.go & \
go build ./cmd/public-visor-monitor/public-visor-monitor.go & \
go build ./cmd/route-finder/route-finder.go & \
go build ./cmd/setup-node/setup-node.go & \
go build ./cmd/sw-env/sw-env.go & \
go build ./cmd/tpd-monitor/tpd-monitor.go & \
go build ./cmd/transport-discovery/transport-discovery.go & \
go build ./cmd/transport-setup/transport-setup.go & \
go build ./cmd/vpn-lite-client/vpn-lite-client.go & \
go build ./cmd/vpn-monitor/vpn-monitor.go
wait
```

optionally, any of those can be `go run` without explicitly compiling

### `address-resolver`
[address-resolver](https://github.com/skycoin/skywire-services/tree/develop/cmd/address-resolver)

_Note: this service requires redis_

Run the address resolver
```
keys-gen | tee ar-config.json
address-resolver --addr ":9093" --redis "redis://localhost:6379" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 ar-config.json)
```

Example `go run` the Address Resolver server from source in a bash shell.

```
rm -rf skywire-services
git clone https://github.com/skycoin/skywire-services
cd skywire-services
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go run cmd/keys-gen/keys-gen.go | tee ar-config.json
go run cmd/address-resolver/address-resolver.go --addr ":9093" --redis "redis://localhost:6379" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 ar-config.json)
```


### `route-finder`
[route-finder](https://github.com/skycoin/skywire-services/tree/develop/cmd/route-finder)

_Note: this service requires postgresql & initial DB setup_
```
sudo -iu postgres createdb rf
```
Run the Route Finder
```
keys-gen | tee rf-config.json
PG_USER="postgres" PG_DATABASE="rf" PG_PASSWORD="" route-finder  --addr ":9092" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 rf-config.json)
```

Example `go run` the Route Finder server from source in a bash shell.

```
rm -rf skywire-services
git clone https://github.com/skycoin/skywire-services
cd skywire-services
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go run cmd/keys-gen/keys-gen.go | tee rf-config.json
PG_USER="postgres" PG_DATABASE="rf" PG_PASSWORD="" go run cmd/route-finder/route-finder.go  --addr ":9092" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 rf-config.json)
```

### `transport-discovery`
[transport-discovery](https://github.com/skycoin/skywire-services/tree/develop/cmd/transport-discovery)

_Note: this service requires redis_

_Note: this service requires postgresql & initial DB setup_
```
sudo -iu postgres createdb tpd
```
Run the Transport Discovery server
```
keys-gen | tee tpd-config.json
PG_USER="postgres" PG_DATABASE="tpd" PG_PASSWORD="" transport-discovery  --addr ":9091" --redis "redis://localhost:6379" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 tpd-config.json)
```

Example `go run` the Transport Discovery server from source in a bash shell.

```
rm -rf skywire-services
git clone https://github.com/skycoin/skywire-services
cd skywire-services
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go run cmd/keys-gen/keys-gen.go | tee tpd-config.json
go run cmd/transport-discovery/transport-discovery.go  --addr ":9091" --redis "redis://localhost:6379" --dmsg-disc "http://127.0.0.1:9090" --sk $(tail -n1 tpd-config.json)
```

### `network-monitor`
[network-monitor](https://github.com/skycoin/skywire-services/tree/develop/cmd/network-monitor)

__Note: this service depends on the uptime tracker which is not yet open source.__

__Note: this service depends on skywire-cli for config generation.__

Network Monitor takes a regular skywire config.
To make network monitor use the services which have been set up, use the `-a` flag for `skywire-cli config gen`
```
skywire-cli config gen -a conf.magnetosphere.net -o nm-config.json
```

The network-monitor enforces the existence of the path `./local/transport_logs` and ignores any changes to that path in the config currently.

```
mkdir -p local/transport_logs
```

Run the network monitor
```
network-monitor -c nm-config.json -a ":9080"
```

Example `go run` the Network Monitor node from source in a bash shell.

__Note: the skywire-cli command is expected to be present in the executable PATH of this environment!__
```
rm -rf skywire-services
git clone https://github.com/skycoin/skywire-services
cd skywire-services
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
skywire-cli config gen -a conf.magnetosphere.net -o nm-config.json
go run cmd/network-monitor/network-monitor.go -c nm-config.json -a ":9080"
```

_Note: the network-monitor IS a skywire visor by another name ; the ports will conflict with the ports used by skywire-visor, please refer to the following section_

## SKYWIRE Setup

This section deals with services which can be built from the [skycoin/skywire](https://github.com/skycoin/skywire) repo

To build all the binaries:

_Please note: use make build to generate correctly versioned binaries_
```
[[ ! -d skywire/.git ]] && rm -rf skywire || true
[[ ! -d skywire ]] git clone https://github.com/skycoin/skywire
cd skywire
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go build ./cmd/skywire-visor/skywire-visor.go & \
go build ./cmd/skywire-cli/skywire-cli.go & \
go build ./cmd/setup-node/setup-node.go & \
go build ./cmd/apps/vpn-server/vpn-server.go & \
go build ./cmd/apps/vpn-client/vpn-client.go & \
go build ./cmd/apps/skysocks/skysocks.go & \
go build ./cmd/apps/skysocks-client/skysocks-client.go & \
go build ./cmd/apps/skychat/skychat.go
wait
```

optionally, `go run` without explicitly compiling


### Route `setup-node`
[setup-node](https://github.com/skycoin/skywire/tree/develop/cmd/setup-node)

Running route setup-node requires a config as follows:
```
{
	"public_key": "02a4072022716ee7fb8205edd4286dc73abe514ff8a7a74c6a27131c6efbe71876",
	"secret_key": "62bcd18c5693ad1d2a24239f7303a84d6afb6966340fc70afe93579987cceb90",
	"dmsg": {
		"discovery": "http://dmsgd.skywire.skycoin.com",
		"sessions_count": 1,
		"servers": []
	},
	"transport_discovery": "http://tpd.skywire.skycoin.com",
	"log_level": "debug"
}
```
This configuration can be parsed from the output of `skywire-cli config gen -n` and should be repopulated with the endpoints for your deployment.

Run the setup-node
```
setup-node setup-node-config.json
```

Example `go run` the Route Setup node from source  in a bash shell

```
rm -rf skywire
git clone https://github.com/skycoin/skywire
cd skywire
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
go run cmd/skywire-cli/skywire-cli.go config gen -n | head -n4 | sed -e '2d' -e 's/sk/secret_key/g' -e 's/pk/public_key/g' | tee setup-node-config.json
echo "	\"dmsg\": {
		\"discovery\": \"http://127.0.0.1:9090\",
		\"sessions_count\": 1,
		\"servers\": []
	},
	\"transport_discovery\": \"http://127.0.0.1:9091\",
	\"log_level\": \"debug\"
}" | tee -a setup-node-config.json
go run cmd/setup-node/setup-node.go setup-node-config.json
```



### `skywire-visor`
[skywire-visor](https://github.com/skycoin/skywire/tree/develop/cmd/skywire-visor)
[skywire-cli](https://github.com/skycoin/skywire/tree/develop/cmd/skywire-cli)

A brief overview of the visor's use with a new deployment is as follows

Generate a config with defaults for your deployment:

_Note the `-p` flag is used for the linux package installation path_
```
skywire-cli config gen -bpirxa conf.magnetosphere.net -o skywire-config.json
```
example output
```
{
	"version": "v1.3.7",
	"sk": "b1dcb86168ae1282ad05f2fa9667106a9bc1373e80ab1c43dcb6952511ad62d8",
	"pk": "02512556f8dbb9e29c42b5236c8d8c848f2a97e2fcf089db07deda0316bf1e341c",
  "dmsg": {
  	"discovery": "http://dmsgd.magnetosphere.net",
  	"sessions_count": 1,
  	"servers": []
  },
  "dmsgpty": {
  	"dmsg_port": 22,
  	"cli_network": "unix",
  	"cli_address": "/tmp/dmsgpty.sock"
  },
  "skywire-tcp": {
  	"pk_table": null,
  	"listening_address": ":7777"
  },
  "transport": {
  	"discovery": "http://tpd.magnetosphere.net",
  	"address_resolver": "http://ar.magnetosphere.net",
  	"public_autoconnect": true,
  	"transport_setup_nodes": null,
  	"log_store": {
  		"type": "file",
  		"location": "/opt/skywire/local/transport_logs",
  		"rotation_interval": "168h0m0s"
  	}
  },
  "routing": {
  	"setup_nodes": [
  		"024fbd3997d4260f731b01abcfce60b8967a6d4c6a11d1008812810ea1437ce438"
  	],
  	"route_finder": "http://rf.magnetosphere.net",
  	"route_finder_timeout": "10s",
  	"min_hops": 0
  },
  "uptime_tracker": {
  	"addr": ""
  },
  "launcher": {
  	"service_discovery": "http://sd.magnetosphere.net",
  	"apps": [
  		{
  			"name": "vpn-client",
  			"binary": "vpn-client",
  			"args": [
  				"-dns",
  				"1.1.1.1"
  			],
  			"auto_start": false,
  			"port": 43
  		},
  		{
  			"name": "skychat",
  			"binary": "skychat",
  			"args": [
  				"-addr",
  				":8001"
  			],
  			"auto_start": true,
  			"port": 1
  			},
  		{
  			"name": "skysocks",
  			"binary": "skysocks",
  			"auto_start": true,
  			"port": 3
  		},
  		{
  			"name": "skysocks-client",
  			"binary": "skysocks-client",
  			"auto_start": false,
  			"port": 13
  		},
  		{
  			"name": "vpn-server",
  			"binary": "vpn-server",
  			"auto_start": false,
  			"port": 44
  		}
  	],
  	"server_addr": "localhost:5505",
  	"bin_path": "/opt/skywire/apps",
  	"display_node_ip": false
  },
  "hypervisors": [],
  "cli_addr": "localhost:3435",
  "log_level": "info",
  "local_path": "/opt/skywire/local",
  "dmsghttp_server_path": "/opt/skywire/local/custom",
  "stun_servers": [
  	"139.162.12.30:3478",
  	"170.187.228.181:3478",
  	"172.104.161.184:3478",
  	"170.187.231.137:3478",
  	"143.42.74.91:3478",
  	"170.187.225.78:3478",
  	"143.42.78.123:3478",
  	"139.162.12.244:3478"
  ],
  "shutdown_timeout": "10s",
  "restart_check_delay": "1s",
  "is_public": false,
  "persistent_transports": null,
  "hypervisor": {
  	"db_path": "/opt/skywire/users.db",
  	"enable_auth": true,
  	"cookies": {
  		"hash_key": 7993b3add6e2f10626568dd8d634e9fcccbdf1b7de767bb2547fabb3be87ab2225438ad37139778169477f6e8ca9308607a5b5f8ebc6ad767e8d69e036b20b79",
  		"block_key": "44143e7231bc37138854dcec93ae25f54ad7fbb400782b3e5e70f0cadc593175",
  		"expires_duration": 43200000000000,
  		"path": "/",
  		"domain": ""
  	},
  	"dmsg_port": 46,
  	"http_addr": ":8000",
  	"enable_tls": false,
  	"tls_cert_file": "./ssl/cert.pem",
  	"tls_key_file": "./ssl/key.pem"
  }
}
```

__Note the default ports:__
```
:7777
:3435
:5505
:8000
:8001
```
__Note: tcp, udp, and http ports in the config will always have a colon and will never be low ports!__


The other ports are `virtual` dmsg ports

Run skywire-visor
```
skywire-visor -c skywire-config.json
```

Example `go run` the Skywire Visor (node) from source  in a bash shell
_Note: substitute your own service conf URL in place of conf.magnetosphere.net for `skywire-cli config gen`_
_Note: the vpn client requires root; the visor is typically run as root._
_Note: running the visor as root or with sudo permissions in the user space will create files and folders as root._
```
rm -rf skywire
git clone https://github.com/skycoin/skywire
cd skywire
#checkout a commit or a branch
git checkout develop
#sync the dependencies just in case
go mod tidy ; go mod vendor
[[ -d apps ]] && rm -r apps || true
mkdir -p apps
echo -e '#!/bin/bash \n go run ../../cmd/apps/skychat/skychat.go' | tee apps/skychat
echo -e '#!/bin/bash \n go run ../../cmd/apps/skysocks/skysocks.go' | tee apps/skysocks
echo -e '#!/bin/bash \n go run ../../cmd/apps/skysocks-client/skysocks-client.go' | tee apps/skysocks-client
echo -e '#!/bin/bash \n go run ../../cmd/apps/vpn-client/vpn-client.go' | tee apps/vpn-client
echo -e '#!/bin/bash \n go run ../../cmd/apps/vpn-server/vpn-server.go' | tee apps/vpn-server
chmod +x ./apps/*
go run cmd/skywire-cli/skywire-cli.go config gen -bixra conf.magnetosphere.net -o skywire-config.json
sudo go run ./cmd/skywire-visor/skywire-visor.go -c skywire-config.json || true
```


## Using Dmsg to connect to the deployment

A JSON config file called `dmsghttp-config.json` can be created containing the dmsg addresses of the services for a deployment. This configuration is used automatically based on region (iran, china) with `skywire-cli config gen -b` or used by default with `skywire-cli config gen -d` and must be present at the expected path - either the current directory or the installation directory - for that configuration to be generated. This file is included in the skywire github repository as well as with our releases.

the following file is created manually to reflect the deployment
```
{
  "test": {
    "dmsg_servers": [
      {
        "static":"024716428e6315d954356e9ad72bea32bb2b41aab5a54a9b5cb4313964016e64d8",
        "server":{
          "address":"172.104.188.39:30080"
        }
      },
      {
        "static":"03f6b0a20be8fe0fd2fd0bd850507cfb10d0eaa37dce5c174654d07db5749a41bf",
        "server":{
          "address":"192.53.115.181:8083"
        }
      },
      {
        "static":"02ae49c901480b49f9c40f6f90fa8e775cf9b00b61ded673a9b0776dfb7cccd374",
        "server":{
          "address":"139.162.45.141:8083"
        }
      }
    ],
    "dmsg_discovery": "dmsg://03cd2336e5de74bdab2bbdb44b06b0c8c713a5ee9029615f5526f8c99a6afe87b8:80",
    "transport_discovery": "dmsg://02703cf828ea11d25b2c8eb0796132ecc7e53b22325b20ce3674ce5cd8693e4fb6:80",
    "address_resolver": "dmsg://030eb7d8cf6eac40c19bbc433de6d6b9bb7a47f2e1d7095c6a01aa676471670ad2:80",
    "route_finder": "dmsg://02ece5b69eaee13ef967b7eb67ca93f1dfddad3a51c9cb1808c4bd0d8d8aa32053:80",
    "uptime_tracker": "dmsg://022c788cca11f208cdfd83ed0c2a8c7b661221736c461adc7c6738a2c1b041c7f8:80",
    "service_discovery": "dmsg://038f751df4af75fb3d51f6693602bfe8289145e633ffdd1e67d686bea595f84d55:80"
  },
  "prod": {
    "dmsg_servers": [
      {
        "static":"02a2d4c346dabd165fd555dfdba4a7f4d18786fe7e055e562397cd5102bdd7f8dd",
        "server":{
          "address":"dmsg.server02a2d4c3.skywire.skycoin.com:30081"
        }
      },
      {
        "static":"03717576ada5b1744e395c66c2bb11cea73b0e23d0dcd54422139b1a7f12e962c4",
        "server":{
          "address":"dmsg.server03717576.skywire.skycoin.com:30082"
        }
      },
      {
        "static":"0228af3fd99c8d86a882495c8e0202bdd4da78c69e013065d8634286dd4a0ac098",
        "server":{
          "address":"45.118.133.242:30084"
        }
      },
      {
        "static":"03d5b55d1133b26485c664cf8b95cff6746d1e321c34e48c9fed293eff0d6d49e5",
        "server":{
          "address":"dmsg.server03d5b55d.skywire.skycoin.com:30083"
        }
      },
      {
        "static":"0281a102c82820e811368c8d028cf11b1a985043b726b1bcdb8fce89b27384b2cb",
        "server":{
          "address":"192.53.114.142:30085"
        }
      },
      {
        "static":"02a49bc0aa1b5b78f638e9189be4ed095bac5d6839c828465a8350f80ac07629c0",
        "server":{
          "address":"dmsg.server02a4.skywire.skycoin.com:30089"
        }
      }
    ],
    "dmsg_discovery": "dmsg://022e607e0914d6e7ccda7587f95790c09e126bbd506cc476a1eda852325aadd1aa:80",
    "transport_discovery": "dmsg://02b307aee5c8ce1666c63891f8af25ad2f0a47a243914c963942b3ba35b9d095ae:80",
    "address_resolver": "dmsg://03234b2ee4128d1f78c180d06911102906c80795dfe41bd6253f2619c8b6252a02:80",
    "route_finder": "dmsg://039d89c5eedfda4a28b0c58b0b643eff949f08e4f68c8357278081d26f5a592d74:80",
    "uptime_tracker": "dmsg://022c424caa6239ba7d1d9d8f7dab56cd5ec6ae2ea9ad97bb94ad4b48f62a540d3f:80",
    "service_discovery": "dmsg://0204890f9def4f9a5448c2e824c6a4afc85fd1f877322320898fafdf407cc6fef7:80"
  }
}
```

## HTTP Service Configuration

The following [caddy-server](https://caddyserver.com/) [`Caddyfile`](https://caddyserver.com/docs/caddyfile/concepts#caddyfile-concepts) configuration reflects the default ports of the http services.

```
dmsgd.magnetosphere.net {
reverse_proxy http://127.0.0.1:9090
import common
}
ar.magnetosphere.net {
reverse_proxy http://127.0.0.1:9093
import common
}
rf.magnetosphere.net {
reverse_proxy http://127.0.0.1:9092
import common
}
tpd.magnetosphere.net {
reverse_proxy http://127.0.0.1:9091
import common
}
sd.magnetosphere.net {
reverse_proxy http://127.0.0.1:9098
import common
}
conf.magnetosphere.net {
header Content-Type	application/json
respond {"dmsg_discovery":"http://dmsgd.magnetosphere.net","transport_discovery":"http://tpd.magnetosphere.net","address_resolver":"http://ar.magnetosphere.net","route_finder":"http://rf.magnetosphere.net","setup_nodes":["024fbd3997d4260f731b01abcfce60b8967a6d4c6a11d1008812810ea1437ce438"],"uptime_tracker":"http://ut.skywire.skycoin.com","service_discovery":"http://sd.magnetosphere.net","stun_servers":["139.162.12.30:3478","170.187.228.181:3478","172.104.161.184:3478","170.187.231.137:3478","143.42.74.91:3478","170.187.225.78:3478","143.42.78.123:3478","139.162.12.244:3478"],"dns_server":"1.1.1.1"}
import common
}
```
