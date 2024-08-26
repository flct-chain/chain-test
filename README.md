# Install kurtosis-cli
Kurtosis is a software which will launch the different parts of a test network and the beaconcha.in explorer, all running locally, using Docker. You will not have to deal with it (nor with Docker), because automating the launch of interdependent modules with Docker and configuring them is the point of Kurtosis. [This short video](https://www.loom.com/share/4256e2b84e5840d3a0a941a80037aebe) gives an overview if it is your first time.

Now, let us install it:
```
echo "deb [trusted=yes] https://apt.fury.io/kurtosis-tech/ /" | sudo tee /etc/apt/sources.list.d/kurtosis.list
sudo apt update
sudo apt install kurtosis-cli
```

# run local testnet
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
