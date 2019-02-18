# Network configuration: bc1


Blockchain sandbox bc1 for cooperating organizations:

  * chain-ops.net
  * civis-blockchain.org
  * thingagora.org

## Initial configuration


### Cryptography

```
cryptogen generate --config=./crypto-config.yaml
```

### Artefacts

```
export FABRIC_CFG_PATH=$PWD
```

#### Orderers

  * Orderer genesis block

```
configtxgen -profile ChainOpsBC1OrdererGenesis  -channelID bcone -outputBlock ./config/genesis.block
```

#### Channels

```
export CHANNEL=sandbox
```

  * Channel configuration transaction 

```
configtxgen -profile BlockchainCoopBC1PeerChannels -outputCreateChannelTx ./config/${CHANNEL}.tx -channelID $CHANNEL
```

  * Anchor peer transaction

```
configtxgen -profile BlockchainCoopBC1PeerChannels -outputAnchorPeersUpdate ./config/BlockchainCoopBC1PeerMSPanchors.tx -channelID ${CHANNEL} -asOrg ChainOpsBC1Peer
```

## Initial deployment

### Docker containers environment

```
export MSP=ChainOpsBC1PeerMSP
export ORGA=pr-bc1.chain-ops.net

echo COMPOSE_PROJECT_NAME="bc1" > .env
echo ca__CA_KEYFILE=$(basename $(ls crypto-config/peerOrganizations/${ORGA}/ca/*_sk)) >> .env
echo ca__TLS_KEYFILE=$(basename $(ls crypto-config/peerOrganizations/${ORGA}/tlsca/*_sk)) >> .env
echo ca__ADMIN=$(cat /dev/urandom | xxd | head -n 1 | cut -b 10-49 | sed "s/ //g") >> .env
echo ca__PASSWD=$(cat /dev/urandom | xxd | head -n 1 | cut -b 10-49 | sed "s/ //g") >> .env

echo cli_MSP=${MSP} >> .env
echo cli_ORGA=${ORGA} >> .env
echo cli_USER=Admin >> .env
```

Note that the `.env` file contains the CA admin and password, remove it from remote hosts and keep the credentials safe.

### Remote hosts setup

Run the following commands as root to create the blockchain user.

```
apt-get install docker.io docker-compose
adduser blockchain
adduser blockchain docker
```

### Create archives for remote hosts

```
orderer-archive.sh orderer0 or-bc1.chain-ops.net
scp orderer_orderer0_or-bc1.chain-ops.net.tgz blockchain@orderer0.or-bc1.chain-ops.net:
```

Peer organisations: pr-bc1.chain-ops.net , pr-bc1.civis-blockchain.org , pr-bc1.thingagora.org

```
export ORGA=pr-bc1.chain-ops.net
```

```
echo ORGA=${ORGA} > env
ca-archive.sh ${ORGA}
scp ca_${ORGA}.tgz blockchain@ca.${ORGA}:
scp env blockchain@ca.${ORGA}:
```

Peer indexes starting from 0

```
idx=0
```

```
echo ORGA=${ORGA} > env
echo idx=${idx} >> env
peer-archive.sh peer${idx} ${ORGA}
scp peer_peer${idx}_${ORGA}.tgz blockchain@peer${idx}.${ORGA}:
scp env blockchain@peer${idx}.${ORGA}:
```

### Start containers on remote hosts

```
ssh blockchain@orderer0.or-bc1.chain-ops.net
tar xvzf orderer_orderer0_or-bc1.chain-ops.net.tgz
docker-compose -f docker-compose.yaml up -d orderer0.or-bc1.chain-ops.net
```

```
ssh blockchain@ca.${ORGA}
source env
tar xvzf ca_${ORGA}.tgz
docker-compose -f docker-compose.yaml up -d ca.${ORGA}
rm env .env docker-compose.yaml ca_${ORGA}.tgz
```

```
ssh blockchain@peer${idx}.${ORGA}
source env
tar xvzf peer_peer${idx}_${ORGA}.tgz
docker-compose -f docker-compose.yaml up -d peer${idx}.${ORGA}
rm env .env docker-compose.yaml peer_peer${idx}_${ORGA}.tgz
```

### Start CLI container locally

```
docker-compose -f docker-compose.yaml up -d cli.or-bc1.chain-ops.net
```

## Network setup on a local machine

### Start network locally with only one peer 

```
docker-compose -f docker-compose.yaml up -d ca.${ORGA} orderer0.or-bc1.chain-ops.net peer${idx}.${ORGA} cli.or-bc1.chain-ops.net
```

### Use the CLI container environment


  * Runtime CLI configuration

Peer MSP ids: ChainOpsBC1Peer , CivisBlockchainBC1Peer , ThingagoraBC1Peer

```
cli_MSP=${MSP}
cli_ORGA=${ORGA}
idx=0
echo cli_ORGA=${cli_ORGA} > ../util/env

# orderer address and certificate
echo ORDERER_ADDR="orderer0.or-bc1.chain-ops.net:7050" >> ../util/env
echo ORDERER_CERT="/etc/hyperledger/orderer/tlsca.or-bc1.chain-ops.net-cert.pem" >> ../util/env

# current session chaincode
echo CHANNEL="sandbox" >> ../util/env
echo CHAINCODE="ex02" >> ../util/env

# override target peer if needed
idx=1
echo CORE_PEER_ADDRESS="peer${idx}.${cli_ORGA}:7051" >> ../util/env
echo CORE_PEER_LOCALMSPID=${cli_MSP} >> ../util/env

# Enter CLI environment
docker exec -it cli-ChainOpsBC1Orderer /bin/bash
```

```
# Channel creation in CLI environment
source /opt/blockchain-coop/env
peer channel create -o ${ORDERER_ADDR} -c ${CHANNEL} -f /etc/hyperledger/config/${CHANNEL}.tx --tls --cafile ${ORDERER_CERT}
peer channel join -b ${CHANNEL}.block
```

```
# Chaincode intantiation in CLI environment
source /opt/blockchain-coop/env
peer chaincode install -n ${CHAINCODE} -v 1.0 -p blockchain-coop/go/chaincode_example02/
peer chaincode instantiate -o ${ORDERER_ADDR} --tls --cafile ${ORDERER_CERT} -C ${CHANNEL} -n ${CHAINCODE} -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('ChainOpsBC1PeerMSP.member', 'CivisBlockchainBC1PeerMSP.member', 'ThingagoraBC1PeerMSP.member')"
```

```
# Chaincode usage in CLI environment
source /opt/blockchain-coop/env
peer chaincode query -C ${CHANNEL} -n ${CHAINCODE} -c '{"Args":["query","a"]}'
peer chaincode invoke -o ${ORDERER_ADDR} -C ${CHANNEL} -n ${CHAINCODE} --tls --cafile ${ORDERER_CERT} -c '{"Args":["invoke","a","b","10"]}'
peer chaincode query -C ${CHANNEL} -n ${CHAINCODE} -c '{"Args":["query","b"]}'
```

```
exit
```

### Stop network and cleanup

```
docker-compose -f docker-compose.yaml down
```

```
git clean -fdx
```

