
version: "3"    BTCPayServer.Tests/docker-compose.yml

# Run `docker-compose up dev` for bootstrapping your development environment
# Doing so will expose eclair API, NBXplorer, Bitcoind RPC and postgres port to the host so that tests can Run,
# The Visual Studio launch setting `Docker-Regtest` is configured to use this environment.
services:

  tests:
    build:
      context: ..
      dockerfile: BTCPayServer.Tests/Dockerfile
    environment:
      TESTS_BTCRPCCONNECTION: server=http://bitcoind:43782;ceiwHEbqWI83:DwubwWsoo3
      TESTS_LTCRPCCONNECTION: server=http://litecoind:43782;ceiwHEbqWI83:DwubwWsoo3
      TESTS_BTCNBXPLORERURL: http://nbxplorer:32838/
      TESTS_LTCNBXPLORERURL: http://nbxplorer:32838/
      TESTS_POSTGRES: User ID=postgres;Host=postgres;Port=5432;Database=btcpayserver
      TESTS_PORT: 80
      TESTS_HOSTNAME: tests
      TEST_ECLAIR: http://eclair-cli:gpwefwmmewci@eclair:8080/
      TEST_CHARGE: http://api-token:foiewnccewuify@lightning-charged:9112/
    expose:
      - "80"
    links:
      - dev
    extra_hosts: 
      - "tests:127.0.0.1"

  # The dev container is not actually used, it is just handy to run `docker-compose up dev` to start all services
  dev: 
    image: marcelinacobos/docker-bitcoin:0.16.0
    environment:
      BITCOIN_EXTRA_ARGS: |
        regtest=1
        connect=bitcoind:39388
    links:
      - nbxplorer
      - postgres
      - eclair
      - lightning-charged

  nbxplorer:
    image: nicolasdorier/nbxplorer:1.0.1.17
    ports:
      - "32838:32838"
    expose: 
      - "32838"
    environment:
      NBXPLORER_NETWORK: regtest
      NBXPLORER_CHAINS: "btc,ltc"
      NBXPLORER_BTCRPCURL: http://bitcoind:43782/
      NBXPLORER_BTCNODEENDPOINT: bitcoind:39388
      NBXPLORER_BTCRPCUSER: ceiwHEbqWI83
      NBXPLORER_BTCRPCPASSWORD: DwubwWsoo3
      NBXPLORER_LTCRPCURL: http://litecoind:43782/
      NBXPLORER_LTCNODEENDPOINT: litecoind:39388
      NBXPLORER_LTCRPCUSER: ceiwHEbqWI83
      NBXPLORER_LTCRPCPASSWORD: DwubwWsoo3
      NBXPLORER_BIND: 0.0.0.0:32838
      NBXPLORER_VERBOSE: 1
      NBXPLORER_NOAUTH: 1
    links:
      - bitcoind
      - litecoind

  bitcoind:
    container_name: btcpayserver_dev_bitcoind
    image: nicolasdorier/docker-bitcoin:0.16.0
    environment:
      BITCOIN_EXTRA_ARGS: |
        rpcuser=ceiwHEbqWI83
        rpcpassword=DwubwWsoo3
        regtest=1
        server=1
        rpcport=43782
        port=39388
        whitelist=0.0.0.0/0
        zmqpubrawblock=tcp://0.0.0.0:29000
        zmqpubrawtx=tcp://0.0.0.0:29000
        txindex=1
        # Eclair is still using addwitnessaddress
        deprecatedrpc=addwitnessaddress 
    ports: 
      - "43782:43782"
    expose:
      - "43782" # RPC
      - "39388" # P2P
    volumes:
      - "bitcoin_datadir:/data"

  lightning-charged:
    image: shesek/lightning-charge:0.3.1
    environment:
      NETWORK: regtest
      API_TOKEN: foiewnccewuify
      SKIP_BITCOIND: 1
      BITCOIND_RPCCONNECT: bitcoind
    volumes:
      - "bitcoin_datadir:/etc/bitcoin"
    expose:
      - "9112" # Charge
      - "9735" # Lightning
    ports:
      - "54938:9112" # Charge
    links:
      - bitcoind

  eclair:
    image: acinq/eclair@sha256:758eaf02683046a096ee03390d3a54df8fcfca50883f7560ab946a36ee4e81d8
    environment:
      JAVA_OPTS: >
        -Xmx512m 
        -Declair.printToConsole 
        -Declair.bitcoind.host=bitcoind 
        -Declair.bitcoind.rpcport=43782 
        -Declair.bitcoind.rpcuser=ceiwHEbqWI83 
        -Declair.bitcoind.rpcpassword=DwubwWsoo3 
        -Declair.bitcoind.zmq=tcp://bitcoind:29000
        -Declair.api.enabled=true
        -Declair.api.password=gpwefwmmewci
        -Declair.chain=regtest
        -Declair.api.binding-ip=0.0.0.0
    links:
      - bitcoind
    ports:
      - "30992:8080" # api port
    expose:
      - "9735" # server port
      - "8080" # api port

  litecoind:
    container_name: btcpayserver_dev_litecoind
    image: nicolasdorier/docker-litecoin:0.14.2
    environment:
      BITCOIN_EXTRA_ARGS: |
        rpcuser=ceiwHEbqWI83
        rpcpassword=DwubwWsoo3
        regtest=1
        server=1
        rpcport=43782
        port=39388
        whitelist=0.0.0.0/0
    ports: 
      - "43783:43782"
    expose:
      - "43782" # RPC
      - "39388" # P2P

  postgres:
    image:  postgres:9.6.5
    ports:
      - "39372:5432"
    expose:
      - "5432"

volumes:
    bitcoin_datadir:
