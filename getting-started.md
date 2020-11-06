# Getting Started with Edgeless DB

In a nutshell, Edgeless DB is a SQL database architected for the SGX environment. Like a normal SQL database, Edgeless DB writes a transaction log to disk and also keeps cold data on disk. With Edgeless DB, all data on disk is strongly encrypted. Data is only ever decrypted inside SGX enclaves. 

Inherent to Edgeless DB is the concept of a *manifest*. Before an instance of Edgeless DB becomes operational, it needs to be initialized with a manifest. The manifest is a simple JSON file that defines how the data stored in Edgeless DB can be accessed by different parties. Clients can (and should) verify that a given instance of Edgeless DB adheres to a certain manifest before they connect via TLS.

Clients talk to Edgeless DB via TLS. On the side of Edgeless DB, all TLS connections terminate inside secure enclaves. Edgeless DB has two interfaces: a standard SQL interface and a REST-like HTTPS interface. The SQL interface typically requires certificate-based client authentication, whereas the HTTPS interface accepts anonymous client connections.

## Docker images

Docker is the simplest way to install and run Edgeless DB. The following Docker images are hosted in private repositories on [hub.docker.com](hub.docker.com).

* `edgelesssystems/edb` runs the Edgeless DB server. The host running this container needs to be an SGX-enabled Intel Xeon processor with Data Center Attestation Primitives (DCAP) support. DCAP is the only form of remote attestation currently supported by Edgeless DB. Please make sure to install the Intel SGX DCAP driver on the host, which is available [here](https://download.01.org/intel-sgx/latest/dcap-latest/linux/distro/). On success, the `/dev/sgx` device should appear on the host.

* `edgelesssystems/pccs` runs a pre-configured Intel Provisioning Certificate Caching Service (PCCS). The PCCS is required for using DCAP. The PCCS contacts remote Intel servers once for each new SGX machine to cache certificates. It thus needs access to the Internet.

* `edgelesssystems/edb-tools` contains tools and example files used in the [walkthrough](#walkthrough) below.

You can run the server containers on the same host directly from the command line as follows.

```bash
docker network create edb-net
docker run --network edb-net --network-alias edgeless.db --restart unless-stopped --detach --volume edb:/edb/data --device /dev/sgx edgelesssystems/edb
docker run --network edb-net --network-alias pccs --restart unless-stopped --detach edgelesssystems/pccs
```

This connects the containers via the Docker network called `edb-net`. Further, it maps the `data` folder in which Edgeless DB stores its files to a folder on the host for persistence. `docker inspect <name of edb container>` gives, among others, the corresponding path on the host. 

Alternatively, you can use the provided `docker-compose.yml` to create the network and start the server containers in a single line using [Docker Compose](https://docs.docker.com/compose/install/). 

```bash
docker-compose --file docker-compose.yml up --detach 
```

## Walkthrough: setting up and using Edgeless DB

This walkthrough describes how to create a simple manifest and initialize and verify Edgeless DB. In this example we consider three parties.

* **Readers** can read data from a set of tables.
* **Writers** can write to a set of tables (but not read).
* The **recoverer** gets access to the recovery key and may decrypt the database for disaster recovery. 

The `edgelesssystems/edb-tools` container contains all tools and files used in this walkthrough. Use the following command to run it with an interactive shell, connect it to the `edb-net` network, and map the `/tools` folder to the host.

```bash
docker run --network edb-net --volume edb-tools:/tools --tty --interactive edgelesssystems/edb-tools
```

You should get a prompt similar to the following.

```bash
root@3b1a17b06c66:/tools#
```

### Step #1: Generate cryptographic keys and certificates

Edgeless DB identifies clients based on their X.509 certificates. The same kind of certificate is used for HTTPS on the web for instance.
Before you can define a manifest, you need to know the X.509 certificates of the relevant parties. The following will create (self-signed) certificates and corresponding private keys in the folders *reader*, *writer* and *recovery*.

```bash
./genkeys.sh
```
### Step #2: Create the manifest

The script from Step #1 also creates an example manifest in `manifest.json`, which should look similar to the following.

```jsons
{
  "sql": [
      "CREATE USER reader REQUIRE ISSUER '/CN=Reader'",
      "CREATE USER writer REQUIRE ISSUER '/CN=Writer'",
      "CREATE TABLE test.data (i INT)",
      "GRANT SELECT ON test.data TO reader",
      "GRANT INSERT ON test.data TO writer"
  ],
  "ca": "-----BEGIN CERTIFICATE-----\n...-----BEGIN CERTIFICATE-----\n...",
  "recovery": "-----BEGIN CERTIFICATE-----\n..."
}
```
This manifest tells Edgeless DB to initialize the database using the given SQL commands and to use the given certificate authorities (CAs) for the identification of clients. Here, there are two CAs: one for readers and one for writers. The `recovery` field holds the certificate of the recoverer.

### Step #3: Obtain and verify the public key of Edgeless DB

Edgeless DB exposes a REST-like HTTPS API under `edgeless.db:8080`. (You can use `docker inspect <edb container name> | grep IPAddress` to get the IP address under with Edgeless DB is reachable from the host.) The following functions are available.

* `/quote`: get a DCAP quote for the instance's X.509 certificate. The quote proves that your Edgeless DB instance is running in a secure SGX enclave. 
* `/manifest`: post the manifest to the instance. Returns the recovery key encrypted for the recoverer.
* `/signature`: once a manifest has been set, this gets a signature from your Edgeless DB instance for the manifest. In combination, the signature and the quote prove that the instance adheres to the manifest and is running in a secure enclave.

Use the `get-cert` tool to obtain the certificate of your Edgeless DB instance and automatically verify the quote against a given configuration with the help of the PCCS.

```bash
./get-cert -c quote_config.json -h edgeless.db:8080 -o edb.pem
```
On success, this writes your instance's certificate to `edb.pem`. The file `quote_config.json` contains the expected parameters of the running enclave.

```json
{
  "SignerID": "10c62e3cf357b07cdee83aa21838df9a3ea38754d761723e3f327dbcbd6cd6d4", 
  "ProductID": 2,
  "SecurityVersion": 1 
}
```

The following enclave parameters exist:
* `SignerID`: the ID of the signing key used by Edgeless Systems GmbH for the enclave package (aka MRSIGNER)
* `UniqueID`: the hash of the enclave package (aka MRENCLAVE); *not used here*
* `ProductID`: the product ID of Edgeless DB
* `SecurityVersion`: the security patch version of Edgeless DB

You are now ready to talk securely to your Edgeless DB instance.

### Step #4: Install the manifest

Installing the manifest (created in Step #2) over a secure connection via the HTTPS API `/manifest` is straightforward.

```bash
curl --cacert edb.pem --data-binary '@manifest.json' https://edgeless.db:8080/manifest --output recovery.bin
```
Passing the verified `edb.pem` to `curl` ensures that it is talking securely to the right Edgeless DB instance. On success, Edgeless DB returns the recovery key encrypted for the recoverer. This is written to `recovery.bin` here. Make sure not to lose it!

### Step #5: Use it!

Finally, authorized readers and writers can connect securely, for example using the `mysql` client. 

```bash
mysql -u reader --ssl-ca=edb.pem --ssl-cert=reader/cert.pem --ssl-key=reader/key.pem -h edgeless.db
```