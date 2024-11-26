# Prisma

## Descripción general

{% hint style="info" %}
[Prysm](https://github.com/prysmaticlabs/prysm) is a Go implementation of Ethereum protocol with a focus on usability, security, and reliability. Prysm is developed by [Prysmatic Labs](https://prysmaticlabs.com), a company with the sole focus on the development of their client. Prysm is written in Go and released under a GPL-3.0 license.
{% endhint %}

#### Enlaces oficiales

| Asunto        | Enlaces                                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------- |
| Lanzamientos  | [https://github.com/prysmaticlabs/prysm/releases](https://github.com/prysmaticlabs/prysm/releases) |
| Documentación | [https://docs.prylabs.network](https://docs.prylabs.network/)                                      |
| Sitio web     | [https://prysmaticlabs.com](https://prysmaticlabs.com/)                                            |

### 1. Configuración inicial 

Cree un usuario de servicio para el servicio de consenso, cree un directorio de datos y asigne
la propiedad.

```bash
sudo adduser --system --no-create-home --group consensus
sudo mkdir -p /var/lib/prysm/beacon
sudo chown -R consensus:consensus /var/lib/prysm/beacon
```

Instalar dependencias.

```bash
sudo apt install curl jq git ccze -y
```

### 2. Instalar Binarios

* Descargar archivos binarios suele ser más rápido y cómodo.
* Construir a partir del código fuente puede ofrecer una mejor compatibilidad y está más alineado con el espíritu de FOSS (software libre de código abierto).

<details>

<summary>Option 1 - Download binaries</summary>

Ejecute lo siguiente para descargar automáticamente los archivos binarios más recientes.

```bash
cd $HOME
prysm_version=$(curl -f -s https://prysmaticlabs.com/releases/latest)
file_beacon=beacon-chain-${prysm_version}-linux-amd64
file_validator=validator-${prysm_version}-linux-amd64
curl -f -L "https://prysmaticlabs.com/releases/${file_beacon}" -o beacon-chain
curl -f -L "https://prysmaticlabs.com/releases/${file_validator}" -o validator
chmod +x beacon-chain validator
```

Instalar los binarios.

<pre class="language-bash"><code class="lang-bash"><strong>sudo mv beacon-chain validator /usr/local/bin
</strong></code></pre>

</details>

<details>

<summary>Opción 2 - Construir desde el código fuente</summary>

Instalar dependencias de Go.

```bash
wget -O go.tar.gz https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go.tar.gz
echo export PATH=$PATH:/usr/local/go/bin >> $HOME/.bashrc
source $HOME/.bashrc
```

Verifique que Go esté instalado correctamente verificando la versión y los archivos de limpieza.

```bash
go version
rm go.tar.gz
```

Instalar dependencias de compilación.

```bash
sudo apt-get update
sudo apt install build-essential git
```

Construye los binarios.

```bash
mkdir -p ~/git
cd ~/git
git clone https://github.com/prysmaticlabs/prysm.git
cd prysm
git fetch --tags
RELEASETAG=$(curl -s https://api.github.com/repos/prysmaticlabs/prysm/releases/latest | jq -r .tag_name)
git checkout tags/$RELEASETAG
go build -o=./build/beacon-chain ./cmd/beacon-chain
go build -o=./build/validator ./cmd/validator
```

Instalar los binarios.

```shell
sudo cp $HOME/git/prysm/build/beacon-chain /usr/local/bin
sudo cp $HOME/git/prysm/build/validator /usr/local/bin
```

</details>

### **3. Instalar y configurar systemd**

Cree un **archivo de unidad systemd** para definir su  `consensus.service` configuración.

```bash
sudo nano /etc/systemd/system/consensus.service
```

Pegue la siguiente configuración en el archivo.

<pre class="language-bash"><code class="lang-bash"><strong>[Unit]
</strong>Description=Prysm Consensus Layer Client service for Mainnet
Wants=network-online.target
After=network-online.target
Documentation=https://www.coincashew.com

[Service]
Type=simple
User=consensus
Group=consensus
Restart=on-failure
RestartSec=3
KillSignal=SIGINT
TimeoutStopSec=900
ExecStart=/usr/local/bin/beacon-chain \
  --mainnet \
  --datadir=/var/lib/prysm/beacon \
  --grpc-gateway-port 5052 \
  --p2p-tcp-port 13000 \
  --p2p-udp-port 12000 \
  --p2p-max-peers 80 \
  --monitoring-port 8008 \
  --checkpoint-sync-url=https://beaconstate.info \
  --execution-endpoint=http://localhost:8551 \
  --jwt-secret=/secrets/jwtsecret \
  --accept-terms-of-use=true \
  --suggested-fee-recipient=&#x3C;0x_CHANGE_THIS_TO_MY_ETH_FEE_RECIPIENT_ADDRESS>

[Install]
WantedBy=multi-user.target
</code></pre>

* Replace**`<0x_CHANGE_THIS_TO_MY_ETH_FEE_RECIPIENT_ADDRESS>`** with your own Ethereum address that you control. Tips are sent to this address and are immediately spendable.
* **Not staking?** If you only want a full node, delete the whole lines beginning with

```
--suggested-fee-recipient
```

Para salir y guardar, presione `Ctrl` + `X`, luego `Y`, luego `Enter`.

Ejecute lo siguiente para habilitar el inicio automático en el momento del arranque.

```bash
sudo systemctl daemon-reload
sudo systemctl enable consensus
```

Finalmente, inicie su cliente de capa de consenso y verifique su estado.

```bash
sudo systemctl start consensus
sudo systemctl status consensus
```

Presione `Ctrl` + `C para salir del estado.

Verifique sus registros para confirmar que los clientes de consenso estén activos y sincronizados.

```bash
sudo journalctl -fu consensus | ccze
```

**Ejemplo de registros de cliente de consenso sincronizados**

```bash
"Peer summary" activePeers=69 inbound=0 outbound=69 prefix=p2p
"Synced new block" block=0xb5ccb2f85... epoch=1837 finalizedEpoch=1838 finalizedRoot=0x1dce0... prefix=blockchain slot=21338 "Finished applying state transition" attestations=128 payloadHash=0x000000000000 prefix=blockchain slot=2138 syncBitsCount=213 txCount=0"terminal difficulty has not been reached yet" latestDifficulty=10000000 prefix=powchain terminalDifficulty=10000000
```

### 4. Comandos útiles del cliente de consenso

{% tabs %}
{% tab title="View Logs" %}
```bash
sudo journalctl -fu consensus | ccze
```

**Ejemplo de registros de cliente de consenso de Prysm sincronizados**

```bash
time="2023-02-02 11:21:00" level=info msg="Peer summary" activePeers=35 inbound=10 outbound=25 prefix=p2p
time="2023-02-02 11:21:00" level=info msg="Synced new block" block=0xd9ddeza1289... epoch=11795 finalizedEpoch=111794 finalizedRoot=0x462e3275... prefix=blockchain slot=31205
time="2023-02-02 11:21:00" level=info msg="Finished applying state transition" attestations=64 payloadHash=0x000000000000 prefix=blockchain slot=31205 syncBitsCount=209 txCount=0
```
{% endtab %}

{% tab title="Stop" %}
```bash
sudo systemctl stop consensus
```
{% endtab %}

{% tab title="Start" %}
```bash
sudo systemctl start consensus
```
{% endtab %}

{% tab title="View Status" %}
```bash
sudo systemctl status consensus
```
{% endtab %}

{% tab title="Reset Database" %}
Las razones comunes para restablecer la base de datos pueden incluir:

* Para reducir el uso de espacio en disco
* Para recuperarse de una base de datos dañada debido a un corte de energía o una falla de hardware
* Para actualizar a un nuevo formato de almacenamiento

```bash
sudo systemctl stop consensus
sudo rm -rf /var/lib/prysm/beacon/beaconchaindata
sudo systemctl restart consensus
```

Con la sincronización de puntos de control habilitada, el tiempo para volver a sincronizar el cliente de consenso debería tomar solo uno o dos minutos.
{% endtab %}
{% endtabs %}

Ahora que su cliente de consenso está configurado e iniciado, tiene un nodo completo.

Continúe con el siguiente paso para configurar su cliente validador, que convierte un nodo completo en un nodo de participación.

{% hint style="info" %}
If you wanted to setup a full node, not a staking node, stop here! Congrats on running your own full node! :tada:
{% endhint %}
