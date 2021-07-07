# Configure, Deploy and Call OpenQ Airnode

You'll need a `MASTER_KEY_MNEMONIC` which will be used to pay for:
- the creation of your provider record
- updating authorizers
- create requesters
- endorsing client contracts

```bash
export MASTER_KEY_MNEMONIC="my mnemonic here"
```

## example.env

### Rename to .env and add your AWS creds and MASTER_KEY_MNEMONIC

Just make sure the AWS creds you use can create Lambdas.

To make it easy just make it an admin if you're unsure how to do so.

## Calculate EndpointId

Start a Node REPL with the below to make sure you can load modules:

```bash
NODE_PATH=$(npm root -g) node
```

```bash
const ethers = require('ethers')
const oisTitle = ""
const endpointName = ""
endpointId = ethers.utils.keccak256(ethers.utils.defaultAbiCoder.encode(['string'], [`${oisTitle}/${endpointName}`]));
```

Exit the REPL and save it as a shell variable for later

```bash
export ENDPOINT_ID={output from above}
```

## config.json
Learn about the Oracle Integration Specification [here](https://docs.api3.org/pre-alpha/airnode/specifications/config-json.html).

Save the `ENDPOINT_ID` and `PROVIDER_URL` you set in `config.json` into shell variables.

```bash
export ENDPOINT_ID=0x43ec2e57fb7e9d4e286a32a73a8bdb8c92e9bfd3885a4e0c94276eca0b322809
export PROVIDER_URL=https://rinkeby.infura.io/v3/54e58d0c329e42cfac03f0fd769abc2a
```

## Deploy Airnode Lambdas

```bash
docker run -it --rm \
  --env-file .env \
  --env COMMAND=deploy-first-time \
  -v $(pwd):/airnode/out \
  api3/airnode-deployer:pre-alpha
```

### Save `PROVIDER_ID` to shell variable

```bash
export PROVIDER_ID=0xb95594ffd2d343e999afc4e249fe77f1b563250597715ca0a06a220a1ae95040
```

## Set authorizers to everyone

```bash
npx @api3/airnode-admin update-authorizers \
  --providerUrl $PROVIDER_URL \
  --mnemonic "$MASTER_KEY_MNEMONIC" \
  --providerId $PROVIDER_ID \
  --endpointId $ENDPOINT_ID \
  --authorizersFilePath ./authorizers.json
```

## Create a requester
Choose a requester admin. This is the wallet holder on whomst'ds behalf your client contract's will make oracle calls.

```bash
export REQUESTER_ADMIN=0x385E436446E3bfA0e91Cb6956beb821177d490b3
```

```bash
npx @api3/airnode-admin create-requester \
  --providerUrl $PROVIDER_URL \
  --mnemonic "$MASTER_KEY_MNEMONIC" \
  --requesterAdmin $REQUESTER_ADMIN
```

```bash
export REQUESTER_INDEX=8
```

## Derive a designated wallet

```bash
npx @api3/airnode-admin derive-designated-wallet \
  --providerUrl $PROVIDER_URL \
  --providerId $PROVIDER_ID \
  --requesterIndex $REQUESTER_INDEX
```

### Save designated wallet's address

```bash
export DESIGNATED_WALLET=0x26F8be4204910e4Df4a8017A7400E32ab19cbc24
```

### Fund your designated wallet
Fund your designated wallet. This is what will pay for gas each time your oracle writes the results of an API call or computation to your `ExampleClient.sol` contract's `fulfill` method.

Note that you are not the custodian of this wallet. It's derived from the provider Airnode's private key.

## Deploy ExampleClient.sol to the same testnet as your Airnode

### Get the right Airnode contract address from [here](https://github.com/api3dao/airnode/tree/pre-alpha/packages/protocol/deployments) or from your `config.json` if you already have it
Pass it as the constructor parameter for launching the ExampleClient.sol contract in Remix

### Save the client address in a shell variable for later

```bash
export CLIENT_ADDRESS=0xD4f6206d651f43ce303044505E30d710Bba3f24F
```

## Endorse ExampleClient.sol
```bash
npx @api3/airnode-admin endorse-client \
  --providerUrl $PROVIDER_URL \
  --mnemonic "$MASTER_KEY_MNEMONIC" \
  --requesterIndex $REQUESTER_INDEX \
  --clientAddress $CLIENT_ADDRESS
```

## Call `makeRequest` on `ExampleClient.sol`

You'll need to encode your parameters before you make the call.

### Encode your parameters in a Node REPL
In a command line Node REPL:

```javascript
const airnodeAbi = require('@api3/airnode-abi');
airnodeAbi.encode([{ name: 'org', type: 'bytes32', value: 'openqdev' }, { name: 'repo', type: 'bytes32', value: 'app' },{ name: 'issueNumber', type: 'bytes32', value: '88' }])
```

Copy ONLY the inner content of the bytes output, not including the single-quotes.

Pass these byte-encoded params, along with the other required params of course, to the `makeRequest` function.

You can check the logs of your lambdas to see when they run.

### Get the `requestId` from the logs of the `makeRequest` transaction
This will be printed in the Remix console

### See the data returned by passing that `requestId` to the `fulfilledData` map
