# Skywire Deployment

This repository contains submodules of all the repositories used for skywire deployment

* [skywire](https://github.com/skycoin/skywire)
* [dmsg](https://github.com/skycoin/dmsg)
* [skywire-utilities](https://github.com/skycoin/skywire-utilities)
* [skywire-services](https://github.com/skycoin/skywire-services)
* [skycoin-service-discovery](https://github.com/skycoin/skycoin-service-discovery)


## Code Checks & Tests

Dependencies: [go](https://go.dev/) [goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports) & [goimports-reviser](https://github.com/incu6us/goimports-reviser)

For any given submodule repository, included is a Makefile with directives such as `format` and `check`.
`make format check` is currently used by the CI to check pull requests.

`make format check` executes a variation of the following commands
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

Dependencies: [go](https://go.dev/)

For any given submodule repository (except skywire-utilities) included is a Makefile with a `build` directive

`make build` executes `go build` for all the binaries that can be built from .go files in `./cmd` subdirectories, for example

```
	go build ./cmd/skywire-visor
```

Note that some binaries may have gcc / libc6 dependency unless statically compiled with musl.

## Runtime Dependencies by Service

### Redis
address-resolver
network-monitor
dmsg-discovery

### Postgres
transport-discovery
route-finder
service-discovery

#### DB Setup

Some services use postgresql.

Run postgresql, make a database with UTF-8 character-set

Pass the database, user, and password as envs

```
export PG_USER=username
export PG_PASSWORD=pass
export PG_DATABASE=sampledb
```

Specify the postgres host and port via flags

```
route-finder --pg-host localhost --pg-port 5432
```

All tables created automatically.


## Required Services

* address-resolver
* network-monitor
* dmsg-discovery
* dmsg-server
* transport-discovery
* route-finder
* service-discovery
* setup-node

A list of endpoints corresponding to some of these services in the current deployment

sd.skycoin.com/api/services?type=proxy
ar.skywire.skycoin.com/transports
tpd.skywire.skycoin.com/all-transports
dmsgd.skywire.skycoin.com/dmsg-discovery/entries
rf.skywire.skycoin.com/


## Key generation

Public/secret keypairs for all services have identical format can can be used interchangeably. There are multiple ways of potentially generating these, but for the sake of brevity the following command is suggested

```
skywire-cli config gen -n | head -n4 | tail -n2 | tee dmsgd-config.json
grep -m1 "sk" dmsgd-config.json | cut -d '"' -f4 | tee -a dmsgd-config.json
```

those services which accept a secret key can then be run as follows
```
dmsg-discovery --sk $(tail -n1 dmsgd-config.json)
```

### Visor Config Bootstrap Endpoint

The following endpoint is queried by the visor on config gen:
https://conf.skywire.skycoin.com/

This endpoint contains json which will become part of the visor's config
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

This endpoint is used to avoid relying on hardcoded defaults for the production deployment endpoints in the visor. Without it, any changes here would require an updated version of skywire to manifest correct config default values

A deployment of _all services as a set_ will require this for visors to query this endpoint during `config gen`

### `dmsg-discovery`
Run the Dmsg Discovery server
```
skywire-cli config gen -n | head -n4 | tail -n2 | tee dmsgd-config.json
grep -m1 "sk" dmsgd-config.json | cut -d '"' -f4 | tee -a dmsgd-config.json
dmsg-discovery --addr ":9090" --redis "redis://localhost:6379" --sk $(tail -n1 dmsgd-config.json)
```

### `dmsg-server`

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

The above "discovery" endpoint should be changed to match the endpoint of the dmsg discovery server referenced in the previous section.
The default ports are shown.

### Run `dmsg-server`

Run the dmsg server (read config from file not working)
```
cat dmsg-config.json | dmsg-server start -c STDIN
```

__The port which the dmsg server is running on must be forwarded or otherwise accessible for public servers.__

## Route `setup-node`

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

## `address-resolver`
Note: this service requires redis

Run the address resolver
```
skywire-cli config gen -n | head -n4 | tail -n2 | tee ar-config.json
grep -m1 "sk" ar-config.json | cut -d '"' -f4 | tee -a ar-config.json
address-resolver --addr ":9093" --redis "redis://localhost:6379" --dmsg-disc "http://dmsgd.skywire.skycoin.com" --sk $(tail -n1 ar-config.json)
```

## `route-finder`

Note: this service requires postgresql & initial DB setup
```
sudo -iu postgres
createdb rf
exit
```
Run the route finder
```
skywire-cli config gen -n | head -n4 | tail -n2 | tee rf-config.json
grep -m1 "sk" rf-config.json | cut -d '"' -f4 | tee -a rf-config.json
PG_USER="postgres" PG_DATABASE="route-finder" PG_PASSWORD="" route-finder  --addr ":9092" --dmsg-disc "http://dmsgd.skywire.skycoin.com" --sk $(tail -n1 rf-config.json)
```

## `transport-discovery`

Note: this service requires postgresql & initial DB setup
```
sudo -iu postgres
createdb tpd
exit
```
Run the transport discovery
```
skywire-cli config gen -n | head -n4 | tail -n2 | tee tpd-config.json
grep -m1 "sk" tpd-config.json | cut -d '"' -f4 | tee -a tpd-config.json
PG_USER="postgres" PG_DATABASE="tpd" PG_PASSWORD="" transport-discovery  --addr ":9091" --dmsg-disc "http://dmsgd.skywire.skycoin.com" --sk $(tail -n1 tpd-config.json)
```

## `service-discovery`

Note: this service requires postgresql & initial DB setup
```
sudo -iu postgres
createdb sd
exit
```
Run the service discovery
```
skywire-cli config gen -n | head -n4 | tail -n2 | tee sd-config.json
grep -m1 "sk" sd-config.json | cut -d '"' -f4 | tee -a sd-config.json
PG_USER="postgres" PG_DATABASE="sd" PG_PASSWORD="" service-discovery  --addr ":9098" --dmsg-disc "http://dmsgd.skywire.skycoin.com" --sk $(tail -n1 sd-config.json)

```

## `network-monitor`

__Note: this service depends on the uptime tracker which is not yet open source.__

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

_Note: the network-monitor is a skywire visor by another name, the ports will conflict with the ports used by skywire-visor_

## `skywire-visor`

The use of the visor is well documented, a brief overview of it's use with a new deployment is as follows

Generate a config with defaults for your deployment:
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
__Note: tcp, udp, and http ports in the config will always have a colon and will never be low ports__

the other ports are 'virtual' dmsg ports

Run skywire-visor
```
skywire-visor -c skywire-config.json
```

## Using Dmsg to connect to the deployment

a config file called `dmsghttp-config.json` can be created containing the dmsg addresses of the services, this configuration is used automatically based on region (iran, china) with `skywire-cli config gen -b` or used by default with `skywire-cli config gen -d`

the following file is created manually to reflect your deployment
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
