# Automação Helm para o AMQ Broker

## Pré-requisitos

### Instalar o Helm

O Helm pode ser baixado e instalado seguindo as intruções em: https://helm.sh/docs/intro/install/

## Estrutura do projeto

```bash
.
├── Chart.yaml
├── examples
│   ├── broker
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
├── templates
│   ├── 1-operatorgroup.yaml
│   ├── 2-subscription.yaml
│   ├── 3-amq-broker.yaml
│   ├── 4-amq-address.yaml
│   └── 5-amq-security.yaml
├── values-subscription.yaml
└── values.yaml
```

### Diretórios

- `Chart.yaml`: Informações básicas do Chart.

- `templates/`: Pasta com os templates do Helm contendo os arquivos de subscrição e CR's para instalação.

- `examples/`: Diversos exemplos, divididos por objetos (CR's), que podem ser compostos para construir diferentes arquiteturas. 

### Parâmetros para deploy do Helm

Parâmetro                       | Descrição
------------------------------  | -------------------------
`amq.name`                      | Nome da instalação
`amq.operator`                  | Informações do CR do Operator (subscription)
`amq.broker`                    | Informações do Broker (ActiveMQArtemis CR) 
`amq.queues`                    | Informações de Address e Queues (ActiveMQArtemisAddress CR) 
`amq.security`                  | Informações de Security (ActiveMQArtemisSecurity CR) 


## Instalação do operator

```bash
# -----------------SUBSCRIPTION---------------
# Instalação em todos os namespaces
$ helm install amq-op --values=examples/subscription/all-namespace-subscription.yaml . 

# Instalação em um namespace específico
$ helm install amq-op --values=examples/subscription/namespace-subscription.yaml --set amq.operator.namespace=<namespace> .
```

PS: O valor default da subscription é com `installPlanApproval` como Manual, esse modelo é indicado para produção, para evitar que o operator atualize a versão e insira vulnerabilidades e bugs, de novas versões, nas instalações dos clientes. Para mais informações leia: https://github.com/bszeti/rhamq-on-openshift/tree/main/operator-install#manual-installplan-approval


## Instalação do broker

### Broker simples

#### Instalação

```bash
$ helm install amq --values=examples/broker/simple-broker.yaml . 

#Se quiser sobrescrever algum valor como usuário e senha
$ helm install amq --values=examples/broker/simple-broker.yaml \
                 --set=amq.broker.adminUser=<user> \
                 --set=amq.broker.adminPassword=<password> \
                 . 
```

#### Teste

```bash
$ export USER=<user>
$ export PASS=<password>
$ export URL=$AMQ_TCP_0_SVC_PORT #check env

# Executar no pod do AMQ
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination test.queue

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Broker simples com múltiplos Addresses

#### Instalação

```bash
helm install amq --values=examples/broker/simple-broker.yaml --values=examples/queues/multiple-queues.yaml .
```

#### Teste

```bash
$ export USER=<user>
$ export PASS=<password>
$ export URL=$AMQ_TCP_0_SVC_PORT #check env

# Executar no pod do Broker
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination queue://ecommerce.order

./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination topic://ecommerce.delivery

./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Broker com conexão SSL e múltiplos Addresses

#### Instalação

Ver: Extra: Criar keytore para o secret de ssl (ex: broker-tls)

```bash
# Aplicar o Helm
$ helm install amq --values=examples/broker/simple-broker-ssl.yaml --values=examples/queues/multiple-queues.yaml . 
```

#### Teste

```bash
# variables
$ export USER=<user>
$ export PASS=<password>
$ export URL="${AMQ_SSL_0_SVC_PORT}?sslEnabled=true&trustStorePath=/etc/broker-tls-volume/broker.ks&trustStorePassword=password&verifyHost=false" #check env

# Executar no pod do Broker
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.order::orders

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Broker persistente com múltiplos Addresses

#### Instalação

Ver: Extra: Criar keytore para o secret de ssl (ex: broker-tls)

```bash
# Aplicar o Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml . 
```

#### Teste

```bash
$ export USER=<user>
$ export PASS=<password>
$ export URL=$AMQ_TCP_0_SVC_PORT #check env

# Executar no pod do Broker
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.order::orders

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

### Broker persistente, com guest module e múltiplos Addresses

#### Instalação

Ver: Extra: Criar keytore para o secret de ssl (ex: broker-tls)

```bash
# Aplicar o Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml --values=examples/security/guest-module.yaml . 
```

#### Teste

<TODO>

### Broker persistente, com properties module e múltiplos Addresses

#### Instalação

Ver: Extra: Criar keytore para o secret de ssl (ex: broker-tls)

```bash
# Aplicar o Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml --values=examples/security/properties-module.yaml . 
```

#### Teste

```bash
# variables
$ export USER=<user>
$ export PASS=<password>
$ export URL="${AMQ_SSL_0_SVC_PORT}?sslEnabled=true&trustStorePath=/etc/broker-tls-volume/broker.ks&trustStorePassword=password&verifyHost=false" #check env

# Executar no pod do Broker
# send message to queue
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.order::orders

# send message to topic
./amq-broker/bin/artemis producer --url $URL --user $USER --password $PASS --message-count 1 --destination ecommerce.delivery

# retrieve status
./amq-broker/bin/artemis queue stat --url $URL --user $USER --password $PASS
```

Se for utilizado um usuário sem permissão, os comandos devem retornar algo similar a: 
`javax.jms.JMSSecurityException: AMQ229032: User: <user> does not have permission='SEND' on address <address>`

### Broker persistente, com keycloak module e múltiplos Addresses

#### Instalação

Ver: Extra: Criar keytore para o secret de ssl (ex: broker-tls)

```bash
# Aplicar o Helm
$ helm install amq --values=examples/broker/simple-broker-persistent-ssl.yaml --values=examples/queues/multiple-queues.yaml --values=examples/security/keycloak-module.yaml . 
```
#### Teste

<TODO>

## Extra: Criar keytore para o secret de ssl (ex: broker-tls)

```bash
$ keytool -genkey -keyalg RSA -keystore broker.ks -storepass password -dname 'CN=selfsigned.amq.openshift'

$ keytool -list -keystore broker.ks -storepass password -v

$ oc create secret generic broker-tls \
  --from-file=broker.ks=broker.ks \
  --from-file=client.ts=broker.ks \
  --from-literal=keyStorePassword=password \
  --from-literal=trustStorePassword=password
```