# vulnerabilidades-MQTT-Broker


# Projeto MQTT — Sensor em Python com TLS e Autenticação

Este repositório apresenta um ambiente MQTT **seguro**, composto por um **broker Mosquitto**, um **sensor de temperatura em Python** (publisher) e um **subscriber com TLS**. A proposta é assegurar **autenticação, separação de portas e criptografia** para acessos externos, enquanto o sensor opera normalmente na rede interna.

---

## Estrutura do Repositório

* `docker-compose.yml`: Orquestração dos serviços do broker, do sensor e do subscriber.
* `src/temperature-sensor-1.py`: Script Python do sensor de temperatura.
* `mosquitto/config/mosquitto.conf`: Configurações do broker.
* `mosquitto/config/mosquitto.passwd`: Banco de senhas para autenticação.
* `mosquitto/config/mosquitto.acl`: Regras de ACL (controle de acesso por tópico).
* `mosquitto/certs/`: Certificados TLS (`ca.crt`, `server.crt`, `server.key`).

---

## Ajustes e Melhorias

### 1. Sensor Python

O sensor foi ajustado para operar de forma estável:

* **Protocolo:** Alterado de `MQTTv5` para `MQTTv3.1.1` (`MQTTv311`), garantindo melhor compatibilidade com o Mosquitto 2.0.20.
* **Host e Porta:** `host = mqtt-broker` e `porta = 1883` para tráfego interno via TCP.

Resultado: o sensor publica leituras continuamente de forma previsível.

### 2. Broker MQTT

O broker foi configurado com foco em segurança:

* **Porta 1883:** Uso interno para sensores (sem TLS).
* **Porta 8883:** Conexões externas com **TLS** + **autenticação obrigatória**.
* **Autorização:** `allow_anonymous false` na porta externa, `password_file` habilitado e **ACLs** para restringir acesso por tópico.
* **Criptografia:** Certificados TLS para garantir confidencialidade, integridade e autenticidade das mensagens.

---

## Autenticação — Passos Essenciais

### 1. Permissões de Arquivos

Garanta que o diretório `mosquitto` pertença ao usuário que executará os comandos:

```bash
sudo chown -R user:user /home/user/mqtt-project/mosquitto
```

### 2. Estrutura de Diretórios

Crie a hierarquia que armazenará configs, dados, logs e certificados:

```bash
mkdir -p /home/user/mqtt-project/mosquitto/{config,data,log,certs}
ls -l /home/user/mqtt-project/mosquitto
```

### 3. Configuração do Broker

No `mosquitto.conf`, desative conexões anônimas e aponte para o arquivo de senhas:

```ini
allow_anonymous false
password_file /home/user/mqtt-project/mosquitto/config/mosquitto.passwd
```

* `allow_anonymous false`: apenas clientes autenticados podem se conectar.
* `password_file`: caminho do arquivo com senhas (hash).

### 4. Criação de Usuário e Senha

Instale os pacotes e gere o arquivo de senhas com `mosquitto_passwd`:

```bash
sudo apt update && sudo apt install mosquitto mosquitto-clients -y
mosquitto_passwd -c /home/user/mqtt-project/mosquitto/config/mosquitto.passwd <USUÁRIO_CRIADO>
```

A opção `-c` cria o `mosquitto.passwd` e adiciona o usuário. Será solicitado que você defina a senha.

### 5. Verificação

Confirme se o arquivo de senhas foi criado:

```bash
ls -l /home/user/mqtt-project/mosquitto/config/mosquitto.passwd
```

---

## Como Executar

Siga os passos abaixo **na raiz do projeto** (onde está o `docker-compose.yml`).

### 1. Gerando Certificados TLS

1. **Crie as pastas necessárias:**

```bash
mkdir -p mosquitto/{config,certs}
```

2. **Gere os certificados para a porta 8883:**

```bash
# Vá para o diretório de certificados
cd mosquitto/certs

# Chave da CA
openssl genrsa -out ca.key 4096

# Certificado autoassinado da CA
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -subj "/CN=MyTestCA" -out ca.crt

# Chave do servidor e CSR
openssl genrsa -out server.key 4096
openssl req -new -key server.key -subj "/CN=mqtt-broker" -out server.csr

# Assinatura do certificado do servidor pela CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256
```

3. **Crie o arquivo de senhas (caso ainda não tenha feito):**

```bash
# No host: gerar arquivo de senha
mosquitto_passwd -b ./mosquitto.passwd <USUÁRIO_CRIADO> <SENHA>

# Mover para a pasta de configuração do broker
mv mosquitto.passwd ../config/mosquitto.passwd
```

4. **Defina as ACLs:**

Crie `mosquitto/config/mosquitto.acl` com:

```text
user <USUÁRIO_CRIADO>
topic read sensor/#
```

### 2. Subindo os Serviços

Com certificados e autenticação prontos, inicie tudo com:

```bash
docker compose up -d --build
```

Isso iniciará o **broker**, o **sensor** e o **subscriber**.

---

## Testes

### Publicação manual (rede interna)

```bash
mosquitto_pub -h localhost -p 1883 -t 'sensor/temperature' -m '27.5' -d
```

### Subscriber externo (TLS)

```bash
mosquitto_sub -h localhost -p 8883 --cafile ./mosquitto/certs/ca.crt -t 'sensor/#' -v --tls-version tlsv1.2 -u <USUÁRIO_CRIADO> -P <SENHA> -d
```

---

## Resumo Técnico

* **Autenticação** via usuário/senha habilitada e obrigatória para conexões externas.
* **Sensor Python** operando por **TCP** na rede interna.
* **Protocolo MQTT** usado pelo sensor: **MQTTv3.1.1**.
* **Broker com TLS e ACLs** para acesso controlado e criptografado.
* **Separação de portas:** `1883` (interno, sem TLS) e `8883` (externo, com TLS + auth).
* **Certificados TLS** criados para proteger o tráfego externo.

Este setup aplica boas práticas de segurança em MQTT, mantendo o ambiente operacional e garantindo a proteção e a integridade dos dados publicados e consumidos.
