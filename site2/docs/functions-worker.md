---
id: functions-worker
title: Deploy and manage functions worker
sidebar_label: "Setup: Pulsar Functions Worker"
---
Before using Pulsar Functions, you need to learn how to set up Pulsar Functions worker and how to [configure Functions runtime](functions-runtime).  

Pulsar `functions-worker` is a logic component to run Pulsar Functions in cluster mode. Two options are available, and you can select either based on your requirements. 
- [run with brokers](#run-functions-worker-with-brokers)
- [run it separately](#run-functions-worker-separately) in a different broker

:::note

The `--- Service Urls---` lines in the following diagrams represent Pulsar service URLs that Pulsar client and admin use to connect to a Pulsar cluster.

:::

## Run Functions-worker with brokers

The following diagram illustrates the deployment of functions-workers running along with brokers.

![assets/functions-worker-corun.png](/assets/functions-worker-corun.png)

To enable functions-worker running as part of a broker, you need to set `functionsWorkerEnabled` to `true` in the `broker.conf` file.

```conf

functionsWorkerEnabled=true

```

If the `functionsWorkerEnabled` is set to `true`, the functions-worker is started as part of a broker. You need to configure the `conf/functions_worker.yml` file to customize your functions_worker.

Before you run Functions-worker with broker, you have to configure Functions-worker, and then start it with brokers.

### Configure Functions-Worker to run with brokers
In this mode, most of the settings are already inherited from your broker configuration (for example, configurationStore settings, authentication settings, and so on) since `functions-worker` is running as part of the broker.

Pay attention to the following required settings when configuring functions-worker in this mode.

- `numFunctionPackageReplicas`: The number of replicas to store function packages. The default value is `1`, which is good for standalone deployment. For production deployment, to ensure high availability, set it to be larger than `2`.
- `initializedDlogMetadata`: Whether to initialize distributed log metadata in runtime. If it is set to `true`, you must ensure that it has been initialized by `bin/pulsar initialize-cluster-metadata` command.

If authentication is enabled on the BookKeeper cluster, configure the following BookKeeper authentication settings.

- `bookkeeperClientAuthenticationPlugin`: the BookKeeper client authentication plugin name.
- `bookkeeperClientAuthenticationParametersName`: the BookKeeper client authentication plugin parameters name.
- `bookkeeperClientAuthenticationParameters`: the BookKeeper client authentication plugin parameters.

### Configure Stateful-Functions to run with broker

If you want to use Stateful-Functions related functions (for example,  `putState()` and `queryState()` related interfaces), follow steps below.

1. Enable the **streamStorage** service in the BookKeeper.

   Currently, the service uses the NAR package, so you need to set the configuration in `bookkeeper.conf`.

   ```text
   
   extraServerComponents=org.apache.bookkeeper.stream.server.StreamStorageLifecycleComponent
   
   ```

   After starting bookie, use the following methods to check whether the streamStorage service is started correctly.

   Input:

   ```shell
   
   telnet localhost 4181
   
   ```

   Output:

   ```text
   
   Trying 127.0.0.1...
   Connected to localhost.
   Escape character is '^]'.
   
   ```

2. Turn on this function in `functions_worker.yml`.

   ```text
   
   stateStorageServiceUrl: bk://<bk-service-url>:4181
   
   ```

   `bk-service-url` is the service URL pointing to the BookKeeper table service.

### Start Functions-worker with broker

Once you have configured the `functions_worker.yml` file, you can start or restart your broker. 

And then you can use the following command to verify if `functions-worker` is running well.

```bash

curl <broker-ip>:8080/admin/v2/worker/cluster

```

After entering the command above, a list of active function workers in the cluster is returned. The output is similar to the following.

```json

[{"workerId":"<worker-id>","workerHostname":"<worker-hostname>","port":8080}]

```

## Run Functions-worker separately

This section illustrates how to run `functions-worker` as a separate process in separate machines.

![assets/functions-worker-separated.png](/assets/functions-worker-separated.png)

:::note

In this mode, make sure `functionsWorkerEnabled` is set to `false`, so you won't start `functions-worker` with brokers by mistake. Also, while accessing the `functions-worker` to manage any of the functions, the `pulsar-admin` CLI tool or any of the clients should use the `workerHostname` and `workerPort` that you set in [Worker parameters](#worker-parameters) to generate an `--admin-url`.

:::

### Configure Functions-worker to run separately

To run function-worker separately, you have to configure the following parameters. 

#### Worker parameters

- `workerId`: The type is string. It is unique across clusters, which is used to identify a worker machine.
- `workerHostname`: The hostname of the worker machine.
- `workerPort`: The port that the worker server listens on. Keep it as default if you don't customize it. Set it to `null` to disable the plaintext port.
- `workerPortTls`: The TLS port that the worker server listens on. Keep it as default if you don't customize it.

#### Function package parameter

- `numFunctionPackageReplicas`: The number of replicas to store function packages. The default value is `1`.

#### Function metadata parameter

- `pulsarServiceUrl`: The Pulsar service URL for your broker cluster.
- `pulsarWebServiceUrl`: The Pulsar web service URL for your broker cluster.
- `pulsarFunctionsCluster`: Set the value to your Pulsar cluster name (same as the `clusterName` setting in the broker configuration).

If authentication is enabled for your broker cluster, you *should* configure the authentication plugin and parameters for the functions worker to communicate with the brokers.

- `brokerClientAuthenticationEnabled`: Whether to enable the broker client authentication used by function workers to talk to brokers.
- `clientAuthenticationPlugin`: The authentication plugin to be used by the Pulsar client used in worker service.
- `clientAuthenticationParameters`: The authentication parameter to be used by the Pulsar client used in worker service.

#### Customize Java runtime options

If you want to pass additional arguments to the JVM command line to every process started by a function worker,
you can configure the `additionalJavaRuntimeArguments` parameter.

```

additionalJavaRuntimeArguments: ['-XX:+ExitOnOutOfMemoryError','-Dfoo=bar']

```

This is very useful in case you want to:
- add JMV flags, like `-XX:+ExitOnOutOfMemoryError`
- pass custom system properties, like `-Dlog4j2.formatMsgNoLookups`

:::note

This feature applies only to Process and Kubernetes runtimes.

:::

#### Security settings

If you want to enable security on functions workers, you *should*:
- [Enable TLS transport encryption](#enable-tls-transport-encryption)
- [Enable Authentication Provider](#enable-authentication-provider)
- [Enable Authorization Provider](#enable-authorization-provider)
- [Enable End-to-End Encryption](#enable-end-to-end-encryption)

##### Enable TLS transport encryption

To enable TLS transport encryption, configure the following settings.

```

useTLS: true
pulsarServiceUrl: pulsar+ssl://localhost:6651/
pulsarWebServiceUrl: https://localhost:8443

tlsEnabled: true
tlsCertificateFilePath: /path/to/functions-worker.cert.pem
tlsKeyFilePath:         /path/to/functions-worker.key-pk8.pem
tlsTrustCertsFilePath:  /path/to/ca.cert.pem

// The path to trusted certificates used by the Pulsar client to authenticate with Pulsar brokers
brokerClientTrustCertsFilePath: /path/to/ca.cert.pem

```

For details on TLS encryption, refer to [Transport Encryption using TLS](security-tls-transport).

##### Enable Authentication Provider

To enable authentication on Functions Worker, you need to configure the following settings.

:::note

Substitute the *providers list* with the providers you want to enable.

:::

```

authenticationEnabled: true
authenticationProviders: [ provider1, provider2 ]

```

For *TLS Authentication* provider, follow the example below to add the necessary settings.
See [TLS Authentication](security-tls-authentication) for more details.

```

brokerClientAuthenticationPlugin: org.apache.pulsar.client.impl.auth.AuthenticationTls
brokerClientAuthenticationParameters: tlsCertFile:/path/to/admin.cert.pem,tlsKeyFile:/path/to/admin.key-pk8.pem

authenticationEnabled: true
authenticationProviders: ['org.apache.pulsar.broker.authentication.AuthenticationProviderTls']

```

For *SASL Authentication* provider, add `saslJaasClientAllowedIds` and `saslJaasServerSectionName`
under `properties` if needed. 

```

properties:
  saslJaasClientAllowedIds: .*pulsar.*
  saslJaasServerSectionName: Broker

```

For *Token Authentication* provider, add necessary settings for `properties` if needed.
See [Token Authentication](security-jwt) for more details.
Note: key files must be DER-encoded

```

properties:
  tokenSecretKey:       file://my/secret.key 
  # If using public/private
  # tokenPublicKey:     file:///path/to/public.key

```

##### Enable Authorization Provider

To enable authorization on Functions Worker, you need to configure `authorizationEnabled`, `authorizationProvider` and `configurationMetadataStoreUrl`. The authentication provider connects to `configurationMetadataStoreUrl` to receive namespace policies.

```yaml

authorizationEnabled: true
authorizationProvider: org.apache.pulsar.broker.authorization.PulsarAuthorizationProvider
configurationMetadataStoreUrl: <meta-type>:<configuration-metadata-store-url>

```

You should also configure a list of superuser roles. The superuser roles are able to access any admin API. The following is a configuration example.

```yaml

superUserRoles:
  - role1
  - role2
  - role3

```

##### Enable End-to-End Encryption

You can use the public and private key pair that the application configures to perform encryption. Only the consumers with a valid key can decrypt the encrypted messages.

To enable End-to-End encryption on Functions Worker, you can set it by specifying `--producer-config` in the command line terminal, for more information, please refer to [here](security-encryption).

We include the relevant configuration information of `CryptoConfig` into `ProducerConfig`. The specific configurable field information about `CryptoConfig` is as follows:

```text

public class CryptoConfig {
    private String cryptoKeyReaderClassName;
    private Map<String, Object> cryptoKeyReaderConfig;

    private String[] encryptionKeys;
    private ProducerCryptoFailureAction producerCryptoFailureAction;

    private ConsumerCryptoFailureAction consumerCryptoFailureAction;
}

```

- `producerCryptoFailureAction`: define the action if producer fail to encrypt data one of `FAIL`, `SEND`.
- `consumerCryptoFailureAction`: define the action if consumer fail to decrypt data one of `FAIL`, `DISCARD`, `CONSUME`.

#### BookKeeper Authentication

If authentication is enabled on the BookKeeper cluster, you need configure the BookKeeper authentication settings as follows:

- `bookkeeperClientAuthenticationPlugin`: the plugin name of BookKeeper client authentication.
- `bookkeeperClientAuthenticationParametersName`: the plugin parameters name of BookKeeper client authentication.
- `bookkeeperClientAuthenticationParameters`: the plugin parameters of BookKeeper client authentication.

### Start Functions-worker

Once you have finished configuring the `functions_worker.yml` configuration file, you can start a `functions-worker` in the background by using [nohup](https://en.wikipedia.org/wiki/Nohup) with the [`pulsar-daemon`](reference-cli-tools.md#pulsar-daemon) CLI tool:

```bash

bin/pulsar-daemon start functions-worker

```

You can also start `functions-worker` in the foreground by using `pulsar` CLI tool:

```bash

bin/pulsar functions-worker

```

### Configure Proxies for Functions-workers

When you are running `functions-worker` in a separate cluster, the admin rest endpoints are split into two clusters. `functions`, `function-worker`, `source` and `sink` endpoints are now served
by the `functions-worker` cluster, while all the other remaining endpoints are served by the broker cluster.
Hence you need to configure your `pulsar-admin` to use the right service URL accordingly.

In order to address this inconvenience, you can start a proxy cluster for routing the admin rest requests accordingly. Hence you will have one central entry point for your admin service.

If you already have a proxy cluster, continue reading. If you haven't setup a proxy cluster before, you can follow the [instructions](http://pulsar.apache.org/docs/en/administration-proxy/) to
start proxies.    

![assets/functions-worker-separated.png](/assets/functions-worker-separated-proxy.png)

To enable routing functions related admin requests to `functions-worker` in a proxy, you can edit the `proxy.conf` file to modify the following settings:

```conf

functionWorkerWebServiceURL=<pulsar-functions-worker-web-service-url>
functionWorkerWebServiceURLTLS=<pulsar-functions-worker-web-service-url>

```

## Compare the Run-with-Broker and Run-separately modes

As described above, you can run Function-worker with brokers, or run it separately. And it is more convenient to run functions-workers along with brokers. However, running functions-workers in a separate cluster provides better resource isolation for running functions in `Process` or `Thread` mode.

Use which mode for your cases, refer to the following guidelines to determine.

Use the `Run-with-Broker` mode in the following cases:
- a) if resource isolation is not required when running functions in `Process` or `Thread` mode; 
- b) if you configure the functions-worker to run functions on Kubernetes (where the resource isolation problem is addressed by Kubernetes).

Use the `Run-separately` mode in the following cases:
-  a) you don't have a Kubernetes cluster; 
-  b) if you want to run functions and brokers separately.

## Troubleshooting

**Error message: Namespace missing local cluster name in clusters list**

```

Failed to get partitioned topic metadata: org.apache.pulsar.client.api.PulsarClientException$BrokerMetadataException: Namespace missing local cluster name in clusters list: local_cluster=xyz ns=public/functions clusters=[standalone]

```

The error message prompts when either of the cases occurs:
- a) a broker is started with `functionsWorkerEnabled=true`, but the `pulsarFunctionsCluster` is not set to the correct cluster in the `conf/functions_worker.yaml` file;
- b) setting up a geo-replicated Pulsar cluster with `functionsWorkerEnabled=true`, while brokers in one cluster run well, brokers in the other cluster do not work well.

**Workaround**

If any of these cases happens, follow the instructions below to fix the problem:

1. Disable Functions Worker by setting `functionsWorkerEnabled=false`, and restart brokers.

2. Get the current clusters list of `public/functions` namespace.

```bash

bin/pulsar-admin namespaces get-clusters public/functions

```

3. Check if the cluster is in the clusters list. If the cluster is not in the list, add it to the list and update the clusters list.

```bash

bin/pulsar-admin namespaces set-clusters --clusters <existing-clusters>,<new-cluster> public/functions

```

4. After setting the cluster successfully, enable functions worker by setting `functionsWorkerEnabled=true`. 

5. Set the correct cluster name in `pulsarFunctionsCluster` in the `conf/functions_worker.yml` file, and restart brokers. 
