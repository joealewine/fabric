# Create a channel without a system channel

Fabric v2.3 introduces the ability to create channels between network members without requiring an ordering service system channel. This feature greatly simplifies the channel creation process and provides a mechanism for Raft consenters to join a channel with zero downtime. Use this tutorial to learn how to create new channels without a system channel by using [configtxgen](../commands/configtxgen.html) tool along with the [osnadmin CLI](../commands/osnadminchannel.html).

**How this process differs from the legacy process:**

 * **System channel no longer required** - Previously you had to define a system channel using `configtxgen` on which all application channels were based.
 * **Ordering service consortium no longer required** - You no longer need to define the set of organizations, also referred to as the "consortium", to transact on an application channel.
 * **Channel simplicity** - There is no longer the concept of a system channel or an application channel. There is simply a "channel" that allows two or more organizations to transact on the blockchain network. Each channel is administered separately, and each ordering service node can join or leave as needed.

**Benefits of the new process:**

 * **Increased Privacy** - A single orderer does not know about all the channels on the ordering service.
 * **Scalability** - When there is a large number of ordering service nodes (orderers) on the system channel, starting the orderer can take a long time as it has to wait for each orderer to replicate the channel ledger.
 * **Operational benefits** - Updating an application channel no longer requires write permissions from a separate ordering organization. Also, you now have a way to list all the channels an orderer is a member of. Lastly, you have the ability to easily remove a channel and its respective resources from an orderer.

Whereas in the past an orderer could only be part of a single ordering service, now orderers can now join (or leave) any number of channels as needed, similar to how peers can participate in multiple channels.

**Note:** This process only works with Raft ordering nodes, it cannot be used with a SOLO or Kafka ordering service.

In the process of creating the channel, this tutorial will take you through the following steps and concepts:

- [Prerequisites](#prerequisites)
- [Step one: Generate the genesis block of the channel](#step-one-generate-the-genesis-block-of-the-channel)
- [Step two: Use the osnadmin CLI to create the channel](#step-two-use-the-osnadmin-cli-to-create-the-channel)
- [Step three: Join additional ordering nodes](#step-three-join-additional-ordering-nodes)
- [Next steps](#next-steps)

## Folder structure

Although not mandatory, the steps throughout this tutorial use the following folder structure for the generated orderer organization MSP and orderer certificates, and is useful when referring to the certificates referenced by the commands.

```
├── organizations
│       ├── ordererOrganizations
│       │   └── ordererOrg1.example.com
│       │       ├── msp
│       │       │   ├── cacerts
│       │       │   | └── ca-cert.pem
│       │       │   ├── config.yaml
│       │       │   ├── tlscacerts
│       │       │   | └── tls-ca-cert.pem
│       │       └── ordering-service-nodes
│       │           ├── osn1.ordererOrg1.example.com
│       │           │   ├── msp
│       │           │   │   ├── IssuerPublicKey
│       │           │   │   ├── IssuerRevocationPublicKey
│       │           │   │   ├── cacerts
│       │           │   │   │   └── HOST1-7055.pem
│       │           │   │   ├── config.yaml
│       │           │   │   ├── keystore
│       │           │   │   │   └── key.pem
│       │           │   │   ├── signcerts
│       │           │   │   │   └── cert.pem
│       │           │   │   └── user
│       │           │   └── tls
│       │           │       ├── IssuerPublicKey
│       │           │       ├── IssuerRevocationPublicKey
│       │           │       ├── cacerts
│       │           │       │   └── tls-ca-cert.pem
│       │           │       ├── keystore
│       │           │       │   └── tls-key.pem
│       │           │       ├── signcerts
│       │           │       │   └── cert.pem
│       │           │       └── user

```

There are three sections in the folder structure above to consider:
- **Orderer organization MSP:** `organizations/ordererOrganizations/ordererOrg1.example.com/msp` folder contains the orderer organization MSP that includes the `cacerts` and  `tlscacerts` folders that you need to create and then copy in the root certificates (`ca-cert.pem`) for the organization CA and TLS CA respectively. If you are using an intermediate CA, you also need to include the corresponding `intermediatecerts` and `tlsintermediatecerts` folders.
- **Orderer local MSP:**`organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/msp` folder, also known as the orderer local MSP, contains the enrollment certificate and private key for the ordering service `osn1` node. This folder is automatically generated when you enroll the orderer identity with the organization CA.
- **TLS certificates:** `organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls` folder contains the TLS certificate and private key for the orderer `osn1` node as well as the TLS CA root cert `tls-ca-cert.pem`.  

The `config.yaml` file in the orderer organization MSP and orderer local MSP enables Node OU support for the channel, an important feature that allows the channel to recognize certificates that contain the "admin" OU as an admin identity of the channel. Learn more in the [Fabric CA](https://hyperledger-fabric-ca.readthedocs.io/en/release-1.4/deployguide/use_CA.html#nodeous) documentation.

## Prerequisites

Because `osnadmin` commands are performed from the perspective of an ordering service node, the ordering service needs to exist before you can create a channel. You can attempt this tutorial with your existing ordering service or deploy a new ordering service.  If you decide to use `osnadmin` command  with an existing ordering service, the system channel must first be removed from each ordering node before you can create any new channels. Choose whether you want to use your existing ordering service or deploy a new one:

- [Use existing ordering service](#use-existing-ordering-service)
- [Deploy a new ordering service](#deploy-a-new-ordering-service)

### Use existing ordering service

Before you can take advantage of this feature on a deployed ordering service, you need to remove the system channel from each ordering node in the application channel consenter set. A "mixed mode" of orderers on a channel, where some nodes are part of a system channel and others are not, is not supported. The `osadmin` CLI includes both a `channel list` and a `system-channel remove` command to facilitate the process of removing the system channel. If you prefer to deploy a new ordering service instead, skip ahead to [Deploy a new ordering service](#deploy-a-new-ordering-service).

#### Remove the system channel

- Before attempting these steps ensure that you have [upgraded](../upgrade.html) your Fabric network to v2.3 or higher.
- Modify the `orderer.yaml` for each ordering node to support this feature and restart the node. See the orderer [sampleconfig](https://github.com/hyperledger/fabric/blob/{BRANCH}/sampleconfig/orderer.yaml) for these new parameters.
    - `General.BootstrapMethod` - Set this value to `none`. Because the system channel is no longer required, the orderer.yaml file on each orderer needs to be configured with `BootstrapMethod: none` which means that no genesis block is required or used to start up the ordering service.
    - `Admin.ListenAddress` - The orderer admin server address (host and port) that can be used by the `osnadmin` command to configure channels on the ordering service. This value should be a unique host:port combination to avoid conflicts.
    - `Admin.TLS.PrivateKey:` - The path to and file name of the orderer private key issued by the TLS CA.
    - `Admin.TLS.Certificate:` - The path to and file name of the orderer signed certificate issued by the TLS CA.
    - `Admin.TLS.RootCAs:`  - The path to and file name of the TLS CA Root cert `tls-ca-cert.pem`.  
    - `Admin.TLS.ClientRootCAs:` - The path to and file name of the TLS CA Root cert `tls-ca-cert.pem`.  
    - `ChannelParticipation:Enabled` - Set this value to `true` to enable this feature on the orderer.
- Put the [system channel into maintenance mode](../kafka_raft_migration.html) using the same process for the Kafka to Raft migration.  
- Remove the system channel from the set of orderers, one by one. If an application channel can tolerate one server offline, you should still be able to submit transactions to the channel, via the other orderers that are not undergoing the removal at that time.  Use the `osnadmin channel list` command to view the channels that this orderer is a member of:
    ```
    osnadmin channel list –-orderer [ORDERER_ADMIN_LISTENADDRESS] --ca-file [TLS_CA_ROOT_CERT] --client-cert [OSN_TLS_SIGN_CERT] --client-key [OSN_TLS_PRIVATE_KEY]
    ```

    where:

    - `ORDERER_ADMIN_LISTENADDRESS` corresponds to the `Orderer.Admin.ListenAddress` defined in the `orderer.yaml` for this orderer.
    - `TLS_CA_ROOT_CERT` is the fully qualified path and file name of the orderer organization TLS CA root certificate and intermediate certificate if using an intermediate TLS CA.
    - `OSN_TLS_SIGN_CERT` is the orderer signed certificate from the TLS CA.
    - `OSN_TLS_PRIVATE_KEY` is the orderer private key from the TLS CA.

    **Note:** The connection between the `osnadmin` CLI and the orderer requires mutual TLS, although it is not required for the network itself. This means you need to pass the `--client-cert` and `--client-key` parameters on each `osdadmin` command. The `--client-cert` parameter points to the orderer certificate issued by the TLS CA, and `--client-key` refers to the orderer private key issued by the TLS CA.  

    For example:
    ```
    osnadmin channel list –-orderer HOST1:7081 --ca-file organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/cacerts/tls-ca-cert.pem --client-cert organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/signcerts/cert.pem --client-key organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/keystore/tls-key.pem.pem
    ```

    The output of this command looks similar to:

    ```
    {
      "systemChannel": {
          "name": "systemchannel",
          "url": "/participation/v1/channels/systemchannel"
      },
      "channels":[
          {
            "name": "channel1",
            "url": "/participation/v1/channels/channel1"
          }
      ]
    }
    ```
    Run `osnadmin system-channel remove` to remove the system channel from the node configuration:

    ```
    osnadmin system-channel remove --orderer [ORDERER_ADMIN_LISTENADDRESS] --channelID systemchannel --ca-file [TLS_CA_ROOT_CERT] --client-cert [OSN_TLS_SIGN_CERT] --client-key [OSN_TLS_PRIVATE_KEY]
    ```

    For example:
    ```
    osnadmin system-channel remove --orderer HOST1:7081 --channelID systemchannel --ca-file organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/cacerts/tls-ca-cert.pem --client-cert organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/signcerts/cert.pem --client-key organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/keystore/tls-key.pem.pem
    ```

    When you rerun the `osnadmin channel list` command, you can verify that the system channel was removed:
    ```
    osnadmin channel list –-orderer [ORDERER_ADMIN_LISTENADDRESS] --ca-file [TLS_CA_ROOT_CERT] --client-cert [OSN_TLS_SIGN_CERT] --client-key [OSN_TLS_PRIVATE_KEY]
    ```

    Examine the output of the command to verify that the system channel is no longer listed:
    ```
    {
      "channels":[
          {
            "name": "channel1",
            "url": "/participation/v1/channels/channel1"
          }
      ]
    }
    ```

    Repeat these commands for each ordering node.


### Deploy a new ordering service

Use these steps if you prefer to deploy a new ordering service to try out this feature. Creating the ordering service is a two-step process:

- [Create the ordering organization MSP and generate ordering node certificates](#create-the-ordering-organization-msp-and-generate-ordering-node-certificates)
- [Configure the orderer.yaml file for each orderer](#configure-the-orderer-yaml-file-for-each-orderer)

#### Create the ordering organization MSP and generate ordering node certificates

Before you can create an ordering service, you need to define the ordering organization MSP definition and generate the TLS and enrollment certificates for each Raft ordering node. To learn how to use a CA to create these identities, check out [Registering and enrolling identities with a CA](https://hyperledger-fabric-ca.readthedocs.io/en/release-1.4/deployguide/use_CA.html). After completing that process, you should have the enrollment and TLS certificates for each node as well as the orderer organization MSP definition. To keep track of the generated certificates and MSP you can use the [folder structure](#folder-structure) defined in this topic, although it is not mandatory.  

If you are using a containerized solution for running your network (which for obvious reasons is a popular choice), **it is a best practice to mount these folders (volumes) external to the container where the node itself is running. This will allow the certificates to be used to create a new node should the node container go down, become corrupted, or is restarted.**

**Repeat this process for the each ordering node**  

Because this tutorial demonstrates the process for creating a channel with **three orderers** deployed for a single organization, you need to generate enrollment and TLS certificates for each node. Why three orderers? This configuration allows for a policy of `MAJORITY` quorum on the Raft cluster. Namely, when there are three orderers, one at a time can go down for maintenance, while a `MAJORITY` (2 of 3) is maintained. But for simplicity and learning purposes, you could also just deploy a single node ordering service.

#### Configure the orderer.yaml file for each orderer

Follow the instructions in the [ordering service deployment guide](LINK) to build an ordering service with three ordering nodes. Because the system channel is no longer required when you start an ordering service, you can skip the entire process of [generating the genesis block](LINK) in those instructions.  However, when you configure the `orderer.yaml` file for each orderer, there are a few additional modifications you need to make to leverage this feature. You can refer to the orderer [sampleconfig](https://github.com/hyperledger/fabric/blob/{BRANCH}/sampleconfig/orderer.yaml) for more information about these new parameters.

- `General.BootstrapMethod` - Set this value to `none`. Because the system channel is no longer required, the orderer.yaml file on each orderer needs to be configured with `BootstrapMethod: none` which means that no genesis block is required or used to start up the ordering service.
- `Admin.ListenAddress` - The orderer admin server address (host and port) that can be used by the `osnadmin` command to configure channels on the ordering service. This value should be a unique host:port combination to avoid conflicts.
- `Admin.TLS.PrivateKey:` - The path to and file name of the orderer private key issued by the TLS CA.
- `Admin.TLS.Certificate:` - The path to and file name of the orderer signed certificate issued by the TLS CA.
- `Admin.TLS.RootCAs:`  - The path to and file name of the TLS CA Root cert `tls-ca-cert.pem`.  
- `Admin.TLS.ClientRootCAs:` - The path to and file name of the TLS CA Root cert `tls-ca-cert.pem`.  
- `ChannelParticipation:Enabled` - Set this value to `true` to enable this feature on the orderer.

**Start each orderer**  

1. If you have not already, set the path to the location of the Fabric binaries on your system:
    ```
    export PATH=<path to download location>/bin:$PATH
    ```
2. In your terminal window set `FABRIC_CFG_PATH` to point to the location of the `orderer.yaml` file, relative to where you are running the fabric commands from. For example, if you download the binaries, and run the commands from the `/bin` directory and the `orderer.yaml` is under `/config`, the path would be:
    ```
    export FABRIC_CFG_PATH=../config
    ```
3. You can now start the ordering service by running the following command on each ordering node:
    ```
    orderer start
    ```

When the ordering service starts successfully and a Raft leader is elected, you should see something similar to the following output:
```
INFO 01a Beginning to serve requests
INFO 01b Applied config change to add node 1, current nodes in channel: [1] channel=syschannel node=1
INFO 01c Applied config change to add node 2, current nodes in channel: [1 2] channel=syschannel node=1
INFO 01d Applied config change to add node 3, current nodes in channel: [1 2 3] channel=syschannel node=1
INFO 01e raft.node: 1 elected leader 2 at term 11 channel=syschannel node=1
INFO 01f Raft leader changed: 0 -> 2 channel=syschannel node=1
```

While the ordering service is started, there are no channels on the ordering service yet, we create a channel in the subsequent steps.

### Define your peer organizations

Because the channel you are creating is meant to be used by two or more peer organizations to transact privately on the network, you need to have at least one peer organization defined to act as the channel administrator who can add other organizations. Technically, the peer nodes themselves do not yet have to be deployed, but you do need to create one or more peer organization MSP definitions and at least one peer organization needs to be provided in the `configtx.yaml` in the next step. Before proceeding to the next section, follow the steps in the [Fabric CA](https://hyperledger-fabric-ca.readthedocs.io/en/release-1.4/deployguide/use_CA.html#create-the-org-msp-needed-to-add-an-org-to-a-channel) documentation to build your peer organization MSP definition.  

## Step one: Generate the genesis block of the channel

Channels are created by building a channel creation transaction, also referred to as the "genesis block", and submitting the transaction to the ordering service. The channel creation transaction specifies the initial configuration of the channel and is used by the ordering service to write the channel genesis block. While only one member needs to create the genesis block, it can be shared out of band with the other members on the channel who can inspect it to ensure they agree to the channel configuration and then used by each orderer.

### Set up the configtxgen tool

While it is possible to build the channel creation transaction file manually, it is easier to use the [configtxgen](../commands/configtxgen.html) tool. The tool works by reading a `configtx.yaml` file that defines the configuration of your channel, and then writing the relevant information into the channel creation transaction.

The  `configtxgen` tool is located in the `bin` folder of downloaded Fabric binaries.

Before using `configtxgen`, confirm you have to set the `FABRIC_CFG_PATH` environment variable to the path of the directory that contains your local copy of the `configtx.yaml` file, as described in the Prerequisites section of this topic. You can check that you can are able to use the tool by printing the `configtxgen` help text:

```
configtxgen --help
```

### The configtx.yaml file

The `configtx.yaml` file specifies the **channel configuration** of new channels. The information that is required to build the channel configuration is specified in a readable and editable form in the `configtx.yaml` file. The `configtxgen` tool uses the channel profiles that are defined in the `configtx.yaml` file to create the channel configuration and write it to the [protobuf format](https://developers.google.com/protocol-buffers) that can be read by Fabric.

The `configtx.yaml` file is located in the `config` folder of the binary alongside images that you downloaded. The file contains the following configuration sections that we need to create our new channel:

- **Organizations:** The organizations that can become members of your channel. Each organization has a reference to the cryptographic material that is used to build the [channel MSP](../membership/membership.html). Exactly which members are part of a channel configuration is defined in the **Profiles** section below.
- **Orderer:** Contains the list of ordering nodes will form the Raft consenter set of the channel.
- **Policies** Different sections of the file work together to define the channel policies that will govern how organizations interact with the channel and which organizations need to approve channel updates. For the purposes of this tutorial, we will use the default that are used by Fabric.
- **Profiles** Each channel profile references information from other sections of the `configtx.yaml` file to build a channel configuration. The profiles are used to create the genesis block of the channel.  

The `configtxgen` tool uses `configtx.yaml` file to create the genesis block for the channel.

You should refer to the [Using configtx.yaml to build a channel configuration](create_channel_config.html) tutorial to learn more about this file. However, the following three sections require specific configuration in order to create a channel without a system channel.

**Note:** Peer organizations are not required when you initially create the channel, but if you know them it is recommended to add them now to the Profiles `Organizations:` and `Applications` section to avoid having to update the channel configuration later.

#### Organizations:

Provide the Orderer organization MSP and Peer organization MSP(s) if they are known. Also provide the endpoint addresses of each ordering node.

- **Organizations.OrdererEndpoints:**

For example:
```
OrdererEndpoints:
            - "Host1:7080"
            - "Host2:7080"
            - "Host3:7080"
```

#### Orderer:

- **Orderer.OrdererType** Set this value to `etcdraft`. As mentioned before, this process does not work with SOLO or Kafka ordering nodes.
- **Orderer.EtcdRaft.Consenters** Provide the list of ordering node addresses, in the form of `host:port`, that are considered active members of the Raft consenter set. All orderers that are listed in this section will become active "consenters" on the channel when they join the channel.

For example:
```
EtcdRaft:
    Consenters:
        - Host: Host1
          Port: 7090
          ClientTLSCert: ../config/organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/signcerts/cert.pem
          ServerTLSCert: ../config/organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn1.ordererOrg1.example.com/tls/signcerts/cert.pem
        - Host: Host2
          Port: 7091
          ClientTLSCert: ../config/organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn2.ordererOrg1.example.com/tls/signcerts/cert.pem
          ServerTLSCert: ../config/organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn2.ordererOrg1.example.com/tls/signcerts/cert.pem
        - Host: Host3
          Port: 7092
          ClientTLSCert: ../config/organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn3.ordererOrg1.example.com/tls/signcerts/cert.pem
          ServerTLSCert: ../config/organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn3.ordererOrg1.example.com/tls/signcerts/cert.pem
```

When the channel configuration block is created, the configtxgen tool reads the paths to the TLS certificates, and replaces the paths with the corresponding bytes of the certificates.

#### Profiles:

If you are familiar with the legacy process for creating channels, you will recall that the `Profiles:` section of the `configtx.yaml` contained a consortium section under the `Orderer:` group. The consortium definition is no longer required. Rather, under the `Orderer:` section you simply include MSP ID of the ordering organization or organizations in the case of a multi-organization Raft ordering service. And in the `Application:` section, you include the MSP ID of the peer organizations that will be recognized as members of the channel. At least one orderer organization and one peer organization must be provided.

The following snippet is an example of a channel profile that contains an orderer configuration based on the default channel, orderer, organization, and policy configurations. The Application section, where the peer organizations are listed, includes the default Application settings as well as at least one peer organization (SampleOrg) and the corresponding policies for the channel.

```
Profiles:
    # SampleAppGenesisEtcdRaft defines a channel configuration with only the
    # sample org as a member. It uses the etcd/raft-based orderer.
    SampleAppGenesisEtcdRaft:
        <<: *ChannelDefaults
        Orderer:
            <<: *OrdererDefaults
            OrdererType: etcdraft
            Organizations:
                - <<: *OrdererOrg
                  Policies:
                      <<: *OrdererOrgPolicies
                      Admins:
                          Type: Signature
                          Rule: "OR('OrdererOrg.member')"
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - <<: *SampleOrg
                  Policies:
                      <<: *SampleOrgPolicies
                      Admins:
                          Type: Signature
                          Rule: "OR('SampleOrg.member')"

```

You can create multiple channel profiles according to your privacy needs by including different combinations of peer and orderer organizations. In the next step, you specify which profile to use when you generate the channel genesis block.

### Generate the genesis block

After you have completed editing the `configtx.yaml`, you can start to create a new channel for the peer organizations.  Every channel configuration starts with a genesis block.  Because we previously set the environment variables for the `configtxgen` tool, you can run the following command to build the genesis block for `channel1` using the `SampleAppGenesisEtcdRaft` profile:

```
configtxgen -profile SampleAppGenesisEtcdRaft -outputCreateChannelTx ./channel-artifacts/channel1.tx -channelID channel1
```

Where:
- `-profile` -  Set this value to the name of the profile to use from `configtx.yaml`.
- `-outputCreateChannelTx` - Provide the location of where to store the generated configuration block file.
- `-channelID` - Specify the name of the channel you want to create. Channel names must be all lower case, less than 250 characters long and match the regular expression ``[a-z][a-z0-9.-]*``. The command uses the `-profile` flag to reference the `SampleAppGenesisEtcdRaft:` profile from `configtx.yaml`.


When the command successful, you will see logs of `configtxgen` loading the `configtx.yaml` file and printing a channel creation transaction:
```
2020-03-11 16:37:12.695 EDT [common.tools.configtxgen] main -> INFO 001 Loading configuration
2020-03-11 16:37:12.738 EDT [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /Usrs/fabric-samples/test-network/configtx/configtx.yaml
2020-03-11 16:37:12.740 EDT [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 003 Generating new channel configtx
2020-03-11 16:37:12.789 EDT [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 004 Writing new channel tx
```

## Step two: Use the osnadmin CLI to create the channel

Each orderer is now able to easily control which channels it joins or leaves. Therefore, the first orderer that runs the `osnadmin channel join` command effectively creates the channel by providing the genesis block, although it is not operational until a consenter quorum is established. In the case of this tutorial, that means running the `osnadmin channel join` command on at least two of the three orderers.

**Note:** If you try to run the `osnadmin` commands (aside from the `channel list` command) against an ordering node that is a member of a system channel, you get an error which indicates that the system channel still exists. The system channel needs to be [removed](#remove-the-system-channel) before the `osnadmin` commands can be used to create or join channels.

Each orderer needs to run the following command:

```
osnadmin channel join –-channelID [CHANNEL_NAME]  --configBlock [CHANNEL_CONFIG_BLOCK] –-orderer [ORDERER_ADMIN_LISTENADDRESS] --ca-file [CA_ROOT_CERT]
```

Replace:
- `CHANNEL_NAME` with the name you want to call this channel.
- `CHANNEL_CONFIG_BLOCK` with the path and file name of the genesis block you created in Step one if you are creating the channel. Each subsequent ordering node can join the configuration starting with the genesis block, or they can join by providing the latest config block instead.  
- `ORDERER_ADMIN_LISTENADDRESS` with `Orderer.Admin.ListenAddress` defined in the `orderer.yaml` for this orderer.
- `CA_ROOT_CERT` with the path to and file name of the TLS CA Root cert `ca-cert.pem`.  

For example:
```
osnadmin channel join –-channelID channel1 -configBlock genesis_block.pb –-orderer OSN1.example.com:7050 --ca-file path
```

<!--**Note:** Because this tutorial assumes you have enabled _server-side_ TLS communications on your network, which is recommended for a Production environment, you need to provide the TLS CA root certificate. If _mutual TLS_ is enabled on your network, you also need to pass the `--client-cert`, and `--clientKey` files on the commands which are omitted from the sample commands throughout this tutorial. -->

The output of this command looks similar to:
```
{
  "name": "channel1",
  "url": "participation/v1/channels/channel1",
  "status": "active",
  "clusterRelation": "consenter",
  "height": 1
}
```

When successful, the channel is created with a ledger height `1`. Because this orderer is joining from the genesis block, the `status` is "active". And because the orderer is part of the channel consenter, its `clusterRelation` is "consenter". We delve more into the `clusterRelation` value in the next section.

You can repeat the same command on the other two orderers. Because these two orderers join the channel from the genesis block and are part of the consenter set, their status transitions almost immediately from  "onboarding" to "active" and the "clusterRelation" status is "consenter". At this point, the channel becomes operational because the default policy of a MAJORITY of consenters (as defined in the channel configuration) are "active".

Note that after the channel is created by the first orderer, subsequent nodes can join from either the genesis block or from the latest config block. When an orderer joins from a config block, its status is always "onboarding" while its ledger catches up to the config block that was specified on the command, after which the status is automatically updated to "active".

Use the `osnadmin channel list` command to view the `status` and `clusterRelation` of an ordering node:

```
osnadmin channel list –-channelID [CHANNEL_NAME] –-orderer [ORDERER_ADMIN_LISTENADDRESS] --ca-file [CA_ROOT_CERT]
```

For example:
```
osnadmin channel list –-channelID channel1 –-orderer HOST2:7081 --ca-file organizations/ordererOrganizations/ordererOrg1.example.com/ordering-service-nodes/osn2.ordererOrg1.example.com/tls
```

Replace:

- `CHANNEL_NAME` with the name of the channel.
- `ORDERER_ADMIN_LISTENADDRESS` with Orderer.Admin.ListenAddress defined in the orderer.yaml for this orderer.
- `CA_ROOT_CERT` with the fully qualified path to the folder that contains the orderer TLS certificates.

The output of this command looks similar to:
```
{
  "name": "channel1",
  "url": "participation/v1/channels/channel1",
  "status": "active",
  "clusterRelation": "consenter",
  "height": 0
}
```

Assuming you have successfully run the `osnadmin channel join` on all three ordering nodes, you now have an active channel and the ordering service is ready to order blocks. Peers can join the channel and begin to transact.  But later, you might want to join additional ordering nodes. The next section describes that process.

The following diagram summarizes the steps you have completed:

![create-channel.osnadmin1-3](./osnadmin1-3.png)
_You used the configtxgen command to create the channel genesis block and provided that file when you ran the osnadmin channel join command for each orderer by targeting the admin endpoint on each node._

## Step three: Join additional ordering nodes

Over time it may be necessary to add additional orderers, for example to allow multiple orderers to go down for maintenance at the same time, or other organizations may want to contribute their own orderers to the cluster. Any time a new orderer joins the cluster, you need to ensure that you do not lose quorum while the ledger on the new orderer catches up. Adding a fourth orderer to an existing three node cluster changes the majority from two to three. And the ordering service will stop and wait for the ledger on the fourth orderer to catch up. To account for this situation and avoid any downtime, the command introduces the "clusterRelation" status of an orderer, which can be either "consenter" or "follower". When a node joins as a follower, the channel ledger is replicated on the orderer, but as it is not an active member, channel operations are not impacted. Because replicating a long chain of blocks could take a long time, joining as a follower is useful for channels with large ledger heights. To join an orderer to a channel as a follower, **do not include the node in the channel configuration consenter set**.

To simplify the tutorial, we assume this additional orderer is part of the same organization as the previous three orderers and the orderer organization is already part of the channel configuration. If the orderer organization is not part of the channel configuration, a channel configuration update must be submitted to add the organization with a minimum of "read" privileges so its orderers can begin pulling blocks.  Run the following command to join the new orderer to the channel:

```
osnadmin channel join –-channelID [CHANNEL_NAME]  -configBlock [CHANNEL_CONFIG_BLOCK] –-orderer [ORDERER_ADMIN_LISTENADDRESS] --ca-file [CA_ROOT_CERT]
```

The orderer can join the channel by providing the genesis block, or the latest config block. But the value of `clusterRelation` is always  "follower" until this orderer is added to the channel consenter set by submitting an update to the channel configuration.

If joining from the genesis block, the output of this command looks similar to:
```
{
  "name": "channel1",
  "url": "participation/v1/channels/channel1",
  "status": "active",
  "clusterRelation": "follower",
  "height": 1
}
```

Otherwise, if joining from the latest config block, the `status` is "onboarding". In that case, after the channel ledger has caught up to the specified config block, you can issue the same `osnadmin channel list` command to confirm the status changes to **active** and the ledger height has caught up. For example, if the total number of blocks on the channel ledger at the specified config block was 1151, the output would look similar to:

```
{
  "name": "channel1",
  "url": "participation/v1/channels/channel1",
  "status": "active",
  "clusterRelation": "follower",
  "height": 1151
}
```

Note that to pull the blocks to the channel ledger, even as a follower, this new orderer must belong to an organization that is part of the current channel configuration, otherwise it is unable to replicate any blocks to its ledger.

Notice the `clusterRelation` status is still **follower** because the node is not part of the channel consenter set.

The diagram shows the new ordering node that has been added as a follower:

![create-channel.osnadmin4](./osnadmin4.png)

While the orderer status is now **active**, it is still not able to participate in the ordering service because its `clusterRelation` status is still a **follower**, it will continue to pull blocks and stay current with the ledger. Because the status is now active, when you are ready for the orderer to transition from a follower to a consenter, you need to add the orderer to the consenter set on the channel by submitting a [channel configuration update transaction](../config_update.html), or by using the [fabric-config](https://github.com/hyperledger/fabric-config) library.  As a reminder,  you should never attempt any configuration changes to the Raft consenters, such as adding a consenter, unless all consenters are online and healthy as you risk losing quorum and your ordering service will not be able to process transactions.

Ensure that the orderer organization has write privileges in the channel configuration so the orderer can become a consenter. When the channel update transaction is complete, you can use the `osnadmin channel list` command again to confirm that the `clusterRelation` status for the orderer automatically changes to **consenter**.

```
{
  "name": "channel1",
  "url": "participation/v1/channels/channel1",
  "status": "active",
  "clusterRelation": "consenter",
  "height": 1151
}
```

## Next steps

### Join peers to the channel

After the channel has been created, you can follow the normal process to join peers to the channel and configure anchor peers.  If the peer organization was not originally included in the channel configuration, you need to submit a [channel configuration update transaction](../config_update.html) to add the organization. If it did include the peer organization, then all peer organizations that are members of the channel can fetch the channel genesis block from the ordering service using the [peer channel fetch](../commands/peerchannel.html#peer-channel-fetch) command. The organization can then use the genesis block to join the peer to the channel using the [peer channel join](../commands/peerchannel.html#peer-channel-join) command. Once the peer is joined to the channel, the peer will build the blockchain ledger by retrieving the other blocks on the channel from the ordering service.

### Add or remove orderers from existing channels

You can continue to use the `osnadmin channel join` and `osnadmin channel remove` commands to add an remove orderers on each channel according to your business needs.

<!--- Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/ -->
