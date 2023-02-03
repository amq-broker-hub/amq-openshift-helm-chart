# Helm Automation for AMQ Broker

## Prerequisites

### Install Helm

Helm could be downloaded and install following the instructions at: https://helm.sh/docs/intro/install/

## Project Structure

```bash
.
├── broker.ks
├── Chart.yaml
├── examples
│   ├── broker
│   │   ├── cluster-broker-persistent-ssl.yaml
│   │   ├── mirror-brokers-source.yaml
│   │   ├── mirror-brokers-target.yaml
│   │   ├── simple-broker-persistent-ssl.yaml
│   │   ├── simple-broker-ssl.yaml
│   │   └── simple-broker.yaml
│   ├── queues
│   │   ├── anycast.yaml
│   │   ├── multicast.yaml
│   │   └── multiple-queues.yaml
│   ├── security
│   │   ├── guest-module.yaml
│   │   ├── keycloak-module.yaml
│   │   └── properties-module.yaml
│   └── subscription
│       ├── all-namespace-subscription.yaml
│       └── namespace-subscription.yaml
├── README.md
├── README-pt-BR.md
├── templates
│   ├── 1-operatorgroup.yaml
│   ├── 2-subscription.yaml
│   ├── 3-amq-broker.yaml
│   ├── 4-amq-address.yaml
│   └── 5-amq-security.yaml
├── values-subscription.yaml
└── values.yaml
```

### Directories

- `Chart.yaml`: Basic informations about the Chart.

- `templates/`: Folder with the Helm templates containing the Subscription and the Broker CR's for install.

- `examples/`: Several examples, divided by objects (CR's), which can be composed to build different broker architectures. 

### Parâmetros para deploy do Helm

Parâmetro                       | Descrição
------------------------------  | -------------------------
`amq.name`                      | Name of installation
`amq.operator`                  | Operator CR (subscription) properties 
`amq.broker`                    | Broker (ActiveMQArtemis CR) properties
`amq.queues`                    | Address e Queues (ActiveMQArtemisAddress CR) properties
`amq.security`                  | Security (ActiveMQArtemisSecurity CR) properties

## Get the .yamls for the examples

Instead of using `helm install <name>` or `helm upgrade <name>`, use `helm template`, eg:

```bash
## Example helm command
helm install amq --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml .

## To generate the manifests (.yaml files)
helm template --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml . 
```

## Testing helm commands

To debug or test the chart, you can use `helm install --dry-run`, `helm install --debug` or both parameters together.

```bash
## Example helm command
helm install amq --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml .

## To debug and test 
helm install amq --dry-run --debug --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml . 
```

## Updating your install

To update some value, you can use `helm upgrade` or `helm upgrade --install` if you want to upgrade with a fallback to install if the're nothing installed yet.

```bash
## Example helm command
helm install amq --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml .

## To update 
helm upgrade --install amq --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml . 
```

## Install the operator

```bash
# -----------------SUBSCRIPTION---------------
# Install in all namespaces
$ helm install amq-op --values=examples/subscription/all-namespace-subscription.yaml . 

# Install in a specific namespace
$ helm install amq-op --values=examples/subscription/namespace-subscription.yaml --set amq.operator.namespace=<namespace> .
```

PS: The default value of the subscription is with `installPlanApproval` as Manual, this model is recommended for production environments, to avoid that an operator upgrade leads to bugs and vulnerabilities, on customers installations. More info read: https://github.com/bszeti/rhamq-on-openshift/tree/main/operator-install#manual-installplan-approval


## Install the broker

### Simple Broker

#### Installation

```bash
$ helm install amq --values=examples/broker/simple-broker.yaml . 

# If you want to override some value like user and password
$ helm install amq --values=examples/broker/simple-broker.yaml \
                 --set=amq.broker.adminUser=<user> \
                 --set=amq.broker.adminPassword=<password> \
                 . 
```

#### Test

```bash
$ export USER=<user>
$ export PASS=<password>
$ export URL=$AMQ_TCP_0_SVC_PORT #check env

# Execute in the Brokers pod
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination test.queue

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Simple Broker with multiple addresses

#### Installation

```bash
helm install amq --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml .
```

#### Test

```bash
$ export USER=<user>
$ export PASS=<password>
$ export URL=$AMQ_TCP_0_SVC_PORT #check env

# Execute in the Brokers pod
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination queue://ecommerce.order

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination topic://ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Broker with SSL connection and multiple addresses

#### Installation

See: **Extra: Create keystore for SSL secret (eg: broker-tls)**

```bash
# Apply Helm
$ helm install amq --values=examples/broker/simple-broker-ssl.yaml --values=examples/queues/multiple-queues.yaml . 
```

#### Test

```bash
# variables
$ export USER=<user>
$ export PASS=<password>
$ export URL="${AMQ_SSL_0_SVC_PORT}?sslEnabled=true&trustStorePath=/etc/broker-tls-volume/broker.ks&trustStorePassword=password&verifyHost=false" #check env

# Execute in the Brokers pod
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.order::orders

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Persistent Broker with multiple addresses

#### Installation

See: **Extra: Create keystore for SSL secret (eg: broker-tls)**

```bash
# Apply Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml . 
```

#### Test

```bash
$ export USER=<user>
$ export PASS=<password>
$ export URL=$AMQ_TCP_0_SVC_PORT #check env

# Execute in the Brokers pod
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.order::orders

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Persistent Broker, with guest module and multiple addresses

#### Installation

See: **Extra: Create keystore for SSL secret (eg: broker-tls)**

```bash
# Apply Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml --values=examples/security/guest-module.yaml . 
```

#### Test

<TODO>

### Persistent Broker, with properties module and multiple addresses

#### Installation

See: **Extra: Create keystore for SSL secret (eg: broker-tls)**

```bash
# Apply Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml --values=examples/security/properties-module.yaml . 
```

#### Test

```bash
# variables
$ export USER=<user>
$ export PASS=<password>
$ export URL="${AMQ_SSL_0_SVC_PORT}?sslEnabled=true&trustStorePath=/etc/broker-tls-volume/broker.ks&trustStorePassword=password&verifyHost=false" #check env

# Execute in the Brokers pod
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.order::orders

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

If you use an user without permissions, the commands should return something similar to:
`javax.jms.JMSSecurityException: AMQ229032: User: <user> does not have permission='SEND' on address <address>`

### Persistent Broker, with keycloak module and multiple addresses

#### Installation

See: **Extra: Create keystore for SSL secret (eg: broker-tls)**

```bash
# Apply Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml --values=examples/security/keycloak-module.yaml . 
```
#### Test

<TODO>

### Persistent Broker, with LDAP module (using an init image) and multiple addresses

#### Installation

See: **Extra: Create keystore for SSL secret (eg: broker-tls)**

```bash
# Apply Helm
$ helm install amq --values=examples/broker/init-image-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml . 
```

#### Test

<TODO>

## Extra: Create keystore for SSL secret (eg: broker-tls)

```bash
$ keytool -genkey -keyalg RSA -keystore broker.ks -storepass password -dname 'CN=selfsigned.amq.openshift'

$ keytool -list -keystore broker.ks -storepass password -v

$ oc create secret generic broker-tls \
  --from-file=broker.ks=broker.ks \
  --from-file=client.ts=broker.ks \
  --from-literal=keyStorePassword=password \
  --from-literal=trustStorePassword=password
```
