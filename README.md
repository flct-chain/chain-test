This guide outlines how to deploy the explorer using a local lh-geth testnet. Utilized postgres, redis and little_bigtable as data storage

# Install docker
If you never worked with Docker, [this short video](https://www.youtube.com/watch?v=rOTqprHv1YE) gives an overview to understand roughly what we will do with it.

Now, let us install it:
```
sudo apt update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

# Install kurtosis-cli
Kurtosis is a software which will launch the different parts of a test network and the beaconcha.in explorer, all running locally, using Docker. You will not have to deal with it (nor with Docker), because automating the launch of interdependent modules with Docker and configuring them is the point of Kurtosis. [This short video](https://www.loom.com/share/4256e2b84e5840d3a0a941a80037aebe) gives an overview if it is your first time.

Now, let us install it:
```
echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
sudo apt update
sudo apt install kurtosis-cli
```

# Install golang
You will find the last version of Go [on this page](https://go.dev/doc/install). The commands that you will type to install it will look like this:

```
wget https://go.dev/dl/go1.21.4.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.4.linux-amd64.tar.gz
```
Add the golang binaries to the path by adding the following lines to your _~/.profile_ file and then logout & login again.
```
export PATH=$PATH:/usr/local/go/bin
export PATH=$PATH:$HOME/go/bin
```
The second line is not mentionned in the installation instructions of Go's website but will be necessary for our system.
Before continuing, restarting your computer now might save you from unexplained errors during the next steps.

# Clone the explorer repository
```
cd ~
git clone https://github.com/gobitfly/eth2-beaconchain-explorer.git
cd eth2-beaconchain-explorer
```

1. run ethereum-package
    kurtosis clean -a
    kurtosis run --enclave my-testnet github.com/ethpandaops/ethereum-package --args-file network_params.yaml

    ```

    participants:
    - el_type: reth
        el_image: ghcr.io/paradigmxyz/reth
        el_volume_size: 200000 # 300GB: 300000, 200GB: 200000, 100GB: 100000
        el_max_cpu: 2000 # 4cores: 4000, 2cores: 2000, 1cores: 1000
        el_max_mem: 8192 # 16GB: 16384, 8GB: 8192, 4GB: 4096
        el_extra_params:
        - --full
        - --gpo.maxprice=5000000 # default: 500000000000
        - --rpc.gascap=50000 # default: 50000000
        cl_type: lighthouse
        cl_image: sigp/lighthouse:latest-unstable
        cl_volume_size: 200000 # 300GB: 300000, 200GB: 200000, 100GB: 100000
        cl_max_cpu: 2000 # 4cores: 4000, 2cores: 2000, 1cores: 1000
        cl_max_mem: 8192 # 16GB: 16384, 8GB: 8192, 4GB: 4096
        cl_extra_params:
        - --reconstruct-historic-states
        - --target-peers=70
        - --http-allow-origin=*
        - --gui
        count: 1
    network_params:
    network_id: "315801020"
    genesis_delay: 20
    seconds_per_slot: 12
    global_log_level: debug
    keymanager_enabled: true
    port_publisher:
    nat_exit_ip: 43.133.56.163
    el:
        enabled: true
        public_port_start: 32000
    cl:
        enabled: true
        public_port_start: 33000
    vc:
        enabled: true
        public_port_start: 34000
    additional_services:
        enabled: true
        public_port_start: 35000


    ```

2. run eth2-beaconchain-explorer
    - move to local-deployment directory
    - replace this code in main.star file
  
        ```

        POSTGRES_PORT_ID = "postgres"
        POSTGRES_DB = "db"
        POSTGRES_USER = "postgres"
        POSTGRES_PASSWORD = "pass"

        REDIS_PORT_ID = "redis"

        LITTLE_BIGTABLE_PORT_ID = "littlebigtable"


        def run(plan):

            db_services = plan.add_services(
                configs={
                    # Add a Postgres server
                    "postgres": ServiceConfig(
                        image="postgres:15.2-alpine",
                        ports={
                            POSTGRES_PORT_ID: PortSpec(5432, application_protocol="postgresql"),
                        },
                        env_vars={
                            "POSTGRES_DB": POSTGRES_DB,
                            "POSTGRES_USER": POSTGRES_USER,
                            "POSTGRES_PASSWORD": POSTGRES_PASSWORD,
                        },
                    ),
                    # Add a Redis server
                    "redis": ServiceConfig(
                        image="redis:7",
                        ports={
                            REDIS_PORT_ID: PortSpec(6379, application_protocol="tcp"),
                        },
                    ),
                    # Add a Bigtable Emulator server
                    "littlebigtable": ServiceConfig(
                        image="gobitfly/little_bigtable:latest",
                        ports={
                            LITTLE_BIGTABLE_PORT_ID: PortSpec(9000, application_protocol="tcp"),
                        },
                    ),
                }
            )

        ```

    - replace this code in provision-explorer-config.sh file

    ``` 

        #! /bin/bash

        CL_PORT=$(kurtosis enclave inspect my-testnet | grep 4000/tcp | tr -s ' ' | cut -d " " -f 6 | sed -e 's/http\:\/\/127.0.0.1\://' | head -n 1)
        echo "CL Node port is $CL_PORT"

        EL_PORT=$(kurtosis enclave inspect my-testnet | grep 8545/tcp | tr -s ' ' | cut -d " " -f 5 | sed -e 's/127.0.0.1\://' | head -n 1)
        echo "EL Node port is $EL_PORT"

        REDIS_PORT=$(kurtosis enclave inspect database-testnet | grep 6379/tcp | tr -s ' ' | cut -d " " -f 6 | sed -e 's/tcp\:\/\/127.0.0.1\://' | head -n 1)
        echo "Redis port is $REDIS_PORT"

        REDIS_SESSIONS_PORT=$(comm -23 <(seq 49152 65535 | sort) <(ss -Htan | awk '{print $4}' | cut -d':' -f2 | sort -u) | shuf | head -n 1)
        echo "Redis sessions port is $REDIS_SESSIONS_PORT"

        POSTGRES_PORT=$(kurtosis enclave inspect database-testnet | grep 5432/tcp | tr -s ' ' | cut -d " " -f 6 | sed -e 's/postgresql\:\/\/127.0.0.1\://' | head -n 1)
        echo "Postgres port is $POSTGRES_PORT"

        LBT_PORT=$(kurtosis enclave inspect database-testnet | grep 9000/tcp | tr -s ' ' | cut -d " " -f 6 | sed -e 's/tcp\:\/\/127.0.0.1\://' | tail -n 1)
        echo "Little bigtable port is $LBT_PORT"

        cat <<EOF > .env
        CL_PORT=$CL_PORT
        EL_PORT=$EL_PORT
        REDIS_PORT=$REDIS_PORT
        REDIS_SESSIONS_PORT=$REDIS_SESSIONS_PORT
        POSTGRES_PORT=$POSTGRES_PORT
        LBT_PORT=$LBT_PORT
        EOF

        touch elconfig.json
        cat >elconfig.json <<EOL
        {
            "byzantiumBlock": 0,
            "constantinopleBlock": 0
        }
        EOL

        touch config.yml

        cat >config.yml <<EOL
        chain:
        clConfigPath: 'node'
        elConfigPath: 'local-deployment/elconfig.json'
        readerDatabase:
        name: db
        host: 127.0.0.1
        port: "$POSTGRES_PORT"
        user: postgres
        password: "pass"
        writerDatabase:
        name: db
        host: 127.0.0.1
        port: "$POSTGRES_PORT"
        user: postgres
        password: "pass"
        bigtable:
        project: explorer
        instance: explorer
        emulator: true
        emulatorPort: $LBT_PORT
        eth1ErigonEndpoint: 'http://127.0.0.1:$EL_PORT'
        eth1GethEndpoint: 'http://127.0.0.1:$EL_PORT'
        redisCacheEndpoint: '127.0.0.1:$REDIS_PORT'
        redisSessionStoreEndpoint: '127.0.0.1:$REDIS_SESSIONS_PORT'
        tieredCacheProvider: 'redis'
        frontend:
        siteDomain: "localhost:8080"
        siteName: 'Open Source Ethereum (ETH) Testnet Explorer' # Name of the site, displayed in the title tag
        siteSubtitle: "Showing a local testnet."
        clCurrency: "PNC"
        elCurrency: "PNC"
        mainCurrency: "PNC"
        clCurrencyDecimals: 18
        clCurrencyDivisor: 1e9
        elCurrencyDecimals: 18
        elCurrencyDivisor: 1e18
        server:
            host: '0.0.0.0' # Address to listen on
            port: '8080' # Port to listen on
        readerDatabase:
            name: db
            host: 127.0.0.1
            port: "$POSTGRES_PORT"
            user: postgres
            password: "pass"
        writerDatabase:
            name: db
            host: 127.0.0.1
            port: "$POSTGRES_PORT"
            user: postgres
            password: "pass"
        recaptchaSiteKey: "6LefJAAqAAAAAExh8LsnwJgA70pyKsADZvpfVWNo"
        sessionSecret: "11111111111111111111111111111111"
        jwtSigningSecret: "1111111111111111111111111111111111111111111111111111111111111111"
        jwtIssuer: "localhost"
        jwtValidityInMinutes: 30
        maxMailsPerEmailPerDay: 10
        mail:
            mailgun:
            sender: no-reply@localhost
            domain: mg.localhost
            privateKey: "key-11111111111111111111111111111111"
        csrfAuthKey: '1111111111111111111111111111111111111111111111111111111111111111'
        legal:
            termsOfServiceUrl: "tos.pdf"
            privacyPolicyUrl: "privacy.pdf"
            imprintTemplate: '{{ define "js" }}{{ end }}{{ define "css" }}{{ end }}{{ define "content" }}Imprint{{ end }}'
        stripe:
            sapphire: price_sapphire
            emerald: price_emerald
            diamond: price_diamond
        ratelimitUpdateInterval: 1s

        indexer:
        # fullIndexOnStartup: false # Perform a one time full db index on startup
        # indexMissingEpochsOnStartup: true # Check for missing epochs and export them after startup
        node:
            host: 127.0.0.1
            port: '$CL_PORT'
            type: lighthouse
        eth1DepositContractFirstBlock: 0
        EOL

        echo "generated config written to config.yml"

        echo "initializing bigtable schema"
        PROJECT="explorer"
        INSTANCE="explorer"
        HOST="127.0.0.1:$LBT_PORT"
        cd ..
        go run ./cmd/misc/main.go -config local-deployment/config.yml -command initBigtableSchema

        echo "bigtable schema initialization completed"

        echo "provisioning postgres db schema"
        go run ./cmd/misc/main.go -config local-deployment/config.yml -command applyDbSchema
        echo "postgres db schema initialization completed"

    ```

    - replace this code in run.sh file

    ```

    #!/bin/bash
    set -e
    DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null && pwd )"
    cd $DIR
    touch .env
    . .env

    var_help="./run.sh <cmd> <options>

    run.sh start  # start local chain and explorer (will run stop first to make sure everything is clean)
    run.sh stop   # stop everything and clean up
    run.sh sql    # connect to database (only works if running)
    "

    fn_main() {
        if test $# -eq 0; then
            echo "$var_help"
            return
        fi
        while test $# -ne 0; do
            case $1 in
                start) shift; fn_start "$@"; exit;;
                stop) shift; fn_stop "$@"; exit;;
                sql) shift; fn_sql "$@"; exit;;
                redis) shift; fn_redis "$@"; exit;;
                misc) shift; fn_misc "$@"; exit;;
                *) echo "$var_help"
            esac
            shift
        done
    }

    fn_misc() {
        docker compose exec misc go run ./cmd/misc -config /app/local-deployment/config.yml $@
    }

    fn_sql() {
        if [ -z "${1}" ]; then
            PGPASSWORD=pass psql -h localhost -p$POSTGRES_PORT -U postgres -d db
        else
            PGPASSWORD=pass psql -h localhost -p$POSTGRES_PORT -U postgres -d db -c "$@" --csv --pset=pager=off
        fi
    }

    fn_redis() {
        if [ -z "${1}" ]; then
            docker compose exec redis-sessions redis-cli
        else
            docker compose exec redis-sessions redis-cli "$@"
        fi
        #redis-cli -h localhost -p $REDIS_PORT
    }

    fn_start() {
        fn_stop
        # build once before starting all services to prevent multiple parallel builds
        docker compose --profile=build-once run -T build-once &
        kurtosis run --enclave database-testnet main.star &
        wait
        bash provision-explorer-config.sh
        docker compose up -d
        echo "Waiting for explorer to start, then browse http://localhost:8080"
    }

    fn_stop() {
        docker compose down -v
        kurtosis enclave rm -f database-testnet
    }

    fn_main "$@"

    ```
