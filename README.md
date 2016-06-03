Sometimes you maybe want to run your own little ethereum network. This is how you do it:

# PREREQUISITES
- install docker
- install docker-compose (`pip install -r requirements.txt`)
- prepare the client container by running
    
        make setup

## TODOS
- determine working docker versions
- determine working docker-compose version
- fix upstream `dockerhub/ethereum/client-python`

# WHAT'S IN THE BOX?
This repository contains two examples:
- `simple`, which suffices to spin up a local private network and should make it easier to understand what is going on
- `with-netstats`, which is pretty much the same, from the ethereum side of things, but adds a local instance of the
  wonderful https://github.com/cubedro/eth-netstats to the package, that lets you monitor your network.

## SIMPLE
I will explain the simple version first and assume, you navigated into the `simple` directory.

`docker-compose.yml` defines four services, which can start 4 slightly different configurations of pyethapp.

1.  **bootstrap**:
A bootstrap node for your network, which also acts as a network bridge between the docker network and the host machine.
*!You can always only run one instance of this!*
2.  **eth**:
Simple "member" nodes, that will connect and relay blocks.
3.  **miner**:
Same as above, but will also mine new blocks.
4.  **debug**:
Non-mining member node, that has extensive logging activated. Start it (see RUN IT below), if you want to trace what is going on in the network.
Then follow its log:

    `docker logs --follow debug  # instance name is 'debug'`

## RUN IT
Navigate to the downloaded docker-compose.yml and start your network with

    docker-compose scale bootstrap=1 miner=2 eth=3
  
If you call `docker-compose ps` afterwards, you should see something like this:

    docker-pyeth-cluster/ % docker-compose ps
    Name                   Command               State                           Ports                          
    ----------------------------------------------------------------------------------------------------------------
    bootstrap        /usr/local/bin/pyethapp -c ...   Up      127.0.0.1:30304->30303/tcp, 127.0.0.1:30304->30303/udp 
    simple_eth_1     /usr/local/bin/pyethapp -c ...   Up                                                             
    simple_eth_2     /usr/local/bin/pyethapp -c ...   Up                                                             
    simple_eth_3     /usr/local/bin/pyethapp -c ...   Up                                                             
    simple_miner_1   /usr/local/bin/pyethapp -c ...   Up                                                             
    simple_miner_2   /usr/local/bin/pyethapp -c ...   Up                                                             

In order to make sure, the network is indeed running and mining, call `docker-compose logs`. If you see things like the following in the log

    miner_1     | INFO:pow.subprocess   nonce found 
    miner_1     | INFO:pow.subprocess   sending nonce   
    miner_1     | INFO:pow  nonce found mining_hash='ffcb7228f9fd1756f9a5fb37e9119574486c9f71f84a3bada9a387ec6d0b933d'
    miner_1     | New head: b88e43795484c92b5881b5e0cf9b41649199bd90ae3756fbbe1ff63a5e92770e 26
    eth_1       | New head: b88e43795484c92b5881b5e0cf9b41649199bd90ae3756fbbe1ff63a5e92770e 26
    bootstrap   | New head: b88e43795484c92b5881b5e0cf9b41649199bd90ae3756fbbe1ff63a5e92770e 26
    eth_2       | New head: b88e43795484c92b5881b5e0cf9b41649199bd90ae3756fbbe1ff63a5e92770e 26
    eth_1       | INFO:eth.chainservice added   block=<Block(#26 b88e4379)> gas_used=0 txs=0
    eth_1       | INFO:eth.chainservice processing time avg=0.04951763153076172 last=0.03158283233642578 max=0.09055614471435547 median=0.048686981201171875 min=0.02843189239501953
    bootstrap   | INFO:eth.chainservice added   block=<Block(#26 b88e4379)> gas_used=0 txs=0
    bootstrap   | INFO:eth.chainservice processing time avg=0.04922186851501465 last=0.03145098686218262 max=0.06899094581604004 median=0.050950050354003906 min=0.021473169326782227
    miner_2     | New head: b88e43795484c92b5881b5e0cf9b41649199bd90ae3756fbbe1ff63a5e92770e 26
    eth_2       | INFO:eth.chainservice added   block=<Block(#26 b88e4379)> gas_used=0 txs=0
    eth_2       | INFO:eth.chainservice processing time avg=0.05634074409802755 last=0.047019004821777344 max=0.13787102699279785 median=0.049088478088378906 min=0.03183603286743164
    miner_2     | INFO:eth.chainservice added   block=<Block(#26 b88e4379)> gas_used=0 txs=0
    miner_2     | INFO:eth.chainservice processing time avg=0.04983563423156738 last=0.04456615447998047 max=0.07030606269836426 median=0.05105710029602051 min=0.027590036392211914


then you know, your network is healthy.

## PLAY WITH IT

### interactive container
Start yourself an interactive (`-it`) instance, with a persistent data volume and create a new account like this:

```
docker run -it --rm --link bootstrap:bootstrap -v /tmp/pyethapp:/root/.config localethereum/client-python -c eth.network_id=1337 -b 'enode://288b97262895b1c7ec61cf314c2e2004407d0a5dc77566877aad1f2a36659c8b698f4b56fd06c4a0c0bf007b4cfb3e7122d907da3b005fa90e724441902eb19e@bootstrap:30303' -m 50 -c eth.genesis_hash=283bd9430c5f3114872f93beefe99d6626980b3a4a18a44ddd27749cd89688f2 account new
    
    WARNING:eth.pow using C++ implementation    
    INFO:config setup default config    path='/root/.config/pyethapp'
    INFO:config writing config  path='/root/.config/pyethapp/config.yaml'
    INFO:app    using data in   path='/root/.config/pyethapp'
    INFO:config loading config  path='/root/.config/pyethapp'
    WARNING:accounts    keystore directory does not exist   directory='/root/.config/pyethapp/keystore'
    WARNING:accounts    no accounts found   
    INFO:app    registering service service='accounts'
    Password to encrypt private key: 
    Repeat for confirmation: 
    INFO:accounts   adding account  account=<Account(address=b0ccb152cb6737b4f5bfac3496595802230d77c8, id=None)>
    Account creation successful
      Address: b0ccb152cb6737b4f5bfac3496595802230d77c8
           Id: None
```

Now run an instance with your account and mine yourself some ether:

```
docker run -it --rm --link bootstrap:bootstrap -v /tmp/pyethapp:/root/.config localethereum/client-python -c eth.network_id=1337 -b 'enode://288b97262895b1c7ec61cf314c2e2004407d0a5dc77566877aad1f2a36659c8b698f4b56fd06c4a0c0bf007b4cfb3e7122d907da3b005fa90e724441902eb19e@bootstrap:30303' -m 50 -c eth.genesis_hash=283bd9430c5f3114872f93beefe99d6626980b3a4a18a44ddd27749cd89688f2 run --fake
```

### native on the host
Since the `bootstrap` node publishes the ports to your local network, you should be able to connect to it from a client on the host system. Try this:

    pyethapp -c eth.network_id=1337 -b 'enode://288b97262895b1c7ec61cf314c2e2004407d0a5dc77566877aad1f2a36659c8b698f4b56fd06c4a0c0bf007b4cfb3e7122d907da3b005fa90e724441902eb19e@localhost:30304' -c eth.genesis_hash=283bd9430c5f3114872f93beefe99d6626980b3a4a18a44ddd27749cd89688f2 run --fake

# WITH NETSTATS
As said in the introduction, the network side of things is pretty much the same. Adding the monitoring daemons to the clients made the containers a little messier though. So, in order to see the whole thing in all its glory, navigate into the `with-netstats`folder.
Since there is an outstandig [bug with docker-compose and networks](https://github.com/docker/compose/issues/2908), you
need to create a network first:

    docker network create withnetstats_ethereum

This needs to be done whenever you start from scratch, i.e. called `docker-compose down` in the `with-netstats` folder.
Now you can spin up the cluster with:
    
    docker-compose scale bootstrap=1 miner=2 eth=10 statsmon=1

(please make sure, you `stop`ped and `rm`ed your simple network beforehand). If all works out, point your browser at http://localhost:3000 and you should see something similar to this:

![alt text](https://github.com/konradkonrad/docker-pyeth-cluster/raw/master/screenshot.png "Private Eth with netstats
and ...")

Oh...and send your ETH to 00000a129284a66728bab6693bfe6cbcaed72bf8 if you want to get rid of them.
