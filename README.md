**Prerequisites**

* **Docker & Docker Compose** installed.  
* **4GB+ RAM** available (minimum).  
* **Root/Sudo access** on the host.


**Phase 1: Certificates**

We cannot start the stack without encryption keys. We will use a temporary container to generate a Certificate Authority (CA) and server certificates.

1. **Create a project folder:**  
   ```
   mkdir elastic-lab && cd elastic-lab  
   mkdir certs
   ```

2. Generate the Certificates:  
   Run this "one-off" command to create the keys.  
   Note: We explicitly add fleet-server and es01 to the DNS list.
   ```
   docker run --rm -v $(pwd)/certs:/certs docker.elastic.co/elasticsearch/elasticsearch:9.2.2 /usr/share/elasticsearch/bin/elasticsearch-certutil cert --silent --pem --keep-ca-key --dns es01 --dns kibana --dns fleet-server --dns localhost --ip 127.0.0.1 -out /certs/certs.zip
   ```
4. Unzip and Fix Permissions:  
   Elasticsearch runs as user 1000. We must ensure it can read these files.  
   ```
   unzip certs/certs.zip -d certs  
   # Crucial: Allow user 1000 to read the keys  
   sudo chown -R 1000:0 certs  
   sudo chmod -R 755 certs
   ```

   *Result: You should now have `certs/ca/ca.crt` and `certs/instance/instance.crt.`*


**Phase 2: The Core (Elasticsearch & Kibana)**

### **1. Configure the Environment**

1. Copy the example file:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and set your variables:
   ```ini
   STACK_VERSION=9.2.2
   STACK_PASSWORD=changeme123       # Set your superuser password here
   ELASTIC_HOST=https://es01:9200   # Internal Docker host
   FLEET_TOKEN=<Generated_Later>    # We will fill this in Phase 3
   ```

### **2. Review docker-compose.yml (Base Layer)**

*Start with just ES and Kibana to establish the database.*

```
version: "3.8"

services:  
  es01:  
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}  
    container_name: es01  
    environment:  
      - node.name=es01  
      - cluster.name=es-docker-cluster  
      - discovery.type=single-node  
      - ELASTIC_PASSWORD=${STACK_PASSWORD}  
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"  
      # SSL & Security  
      - xpack.security.enabled=true  
      - xpack.security.http.ssl.enabled=true  
      - xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/instance/instance.key  
      - xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/instance/instance.crt  
      - xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca/ca.crt  
      - xpack.security.transport.ssl.enabled=true  
      - xpack.security.transport.ssl.key=/usr/share/elasticsearch/config/certs/instance/instance.key  
      - xpack.security.transport.ssl.certificate=/usr/share/elasticsearch/config/certs/instance/instance.crt  
      - xpack.security.transport.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca/ca.crt  
    volumes:  
      - es_data:/usr/share/elasticsearch/data  
      - ./certs:/usr/share/elasticsearch/config/certs  
    ports:  
      - "9200:9200"  
    networks:  
      - elastic  
    mem_limit: 4g

  kibana:  
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}  
    container_name: kibana  
    ports:  
      - "5601:5601"  
    environment:  
      - SERVERNAME=kibana  
      - ELASTICSEARCH_HOSTS=https://es01:9200  
      - ELASTICSEARCH_USERNAME=kibana_system  
      - ELASTICSEARCH_PASSWORD=${STACK_PASSWORD}  
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=/usr/share/kibana/config/certs/ca/ca.crt  
      \# Encryption Keys (Generate random strings for Prod\!)  
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=d1a665977508479389531584933909a8  
      - XPACK_REPORTING_ENCRYPTIONKEY=d1a665977508479389531584933909a8  
      - XPACK_SECURITY_ENCRYPTIONKEY=d1a665977508479389531584933909a8  
    volumes:  
      - ./certs:/usr/share/kibana/config/certs  
    networks:  
      - elastic  
    depends_on:  
      - es01  
    mem_limit: 2g

volumes:  
  es_data:  
    driver: local

networks:  
  elastic:  
    driver: bridge
```

### **3. Initialize the Stack**

1. **Start Elasticsearch Only:**   
   `docker compose up -d es01`

2. **Set the kibana_system Password:**  
   Wait 30s for ES to boot, then run:  
   `docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -i`

   *(Enter the STACK_PASSWORD you put in your .env file)*  
3. **Start Kibana:**  
   `docker compose up -d kibana`

4. **Verify:** 
   Login at `https://<YOUR_IP>:5601` with user elastic.


**Phase 3: The Fleet Server**

*We avoid the "Auto-Setup" scripts because they often timeout on lab hardware. We will generate the token manually.*

1. **Generate Service Token:**  
   * In Kibana: **Management** -> **Fleet** -> **Add Fleet Server**.  
   * **Do not run the command.** Just copy the **Service Token** (AAEAA...).  
2. Add Fleet to docker-compose.yml:  
   Append this service to your file (if you're using the example file, it's already there). Paste your token where indicated.
```
  fleet-server:  
    image: docker.elastic.co/elastic-agent/elastic-agent:${STACK_VERSION}  
    container_name: fleet-server  
    healthcheck:  
      test: ["CMD-SHELL", "curl -s -k https://localhost:8220/api/status | grep -q 'HEALTHY'"]  
      retries: 300  
      interval: 5s  
    environment:  
      # 1. The Token (Crucial to avoid timeouts)  
      - FLEET_SERVER_SERVICE_TOKEN=${FLEET_TOKEN}  
        
      \# 2\. The "Homelab Cheat" (Trust self-signed ES certs)  
      - FLEET_SERVER_ELASTICSEARCH_INSECURE=true  
        
      \# Standard Configs  
      - FLEET_SERVER_ENABLE=1  
      - FLEET_SERVER_POLICY_ID=fleet-server-policy  
      - FLEET_URL=https://fleet-server:8220  
      - ELASTICSEARCH_HOST=https://es01:9200  
      - ELASTICSEARCH_USERNAME=elastic  
      - ELASTICSEARCH_PASSWORD=${STACK_PASSWORD}  
      - ELASTICSEARCH_CA=/usr/share/elastic-agent/certs/ca/ca.crt  
      - FLEET_SERVER_ELASTICSEARCH_HOST=https://es01:9200  
      - FLEET_SERVER_ELASTICSEARCH_CA=/usr/share/elastic-agent/certs/ca/ca.crt  
      - FLEET_SERVER_CERT=/usr/share/elastic-agent/certs/instance/instance.crt  
      - FLEET_SERVER_CERT_KEY=/usr/share/elastic-agent/certs/instance/instance.key  
    ports:  
      - "8220:8220"  
    volumes:  
      - ./certs:/usr/share/elastic-agent/certs  
    networks:  
      - elastic  
    depends_on:  
      - es01  
      - kibana  
    mem_limit: 1g
```

3. **Launch Fleet:**  
   `docker compose up -d fleet-server`

   *Check logs: `docker logs -f fleet-server`. It should be HEALTHY immediately.*


**Phase 4: Final Configuration (Networking)**

We must tell Agents how to find the server from *outside* Docker.

1. **Go to Kibana:** **Fleet** -> **Settings**.  
2. **Update Fleet Server Host:**  
   * Change to: `https://<YOUR_UBUNTU_IP>:8220`
3. **Update Output (Elasticsearch):**  
   * Edit the default output.  
   * **Host:** `https://<YOUR_UBUNTU_IP>:9200`
   * **Advanced YAML (Crucial):**  
     `ssl.verification_mode: none`

   * *Why? Because the cert says "es01" but agents connect via IP. We must ignore the name mismatch.*


**Phase 5: Enrolling Agents (Laptops/Servers)**

To monitor other devices, install the Elastic Agent.

The Command (Linux/Mac):  
You MUST append `--insecure` because we are using self-signed certs.

```
sudo ./elastic-agent install \ 
  --url=https://<YOUR_UBUNTU_IP>:8220 \ 
  --enrollment-token=<YOUR_TOKEN> \  
  --insecure
```

> **SECURITY WARNING:** We are using `--insecure` and `ssl.verification_mode: none` because we are using self-signed certificates generated in Phase 1. In a production environment, you should use a proper PKI infrastructure and verify certificates to prevent Man-in-the-Middle (MITM) attacks.

**Troubleshooting Cheat Sheet**

| Symptom | Cause | Fix |
| :---- | :---- | :---- |
| **Fleet Server: 503 / Timeout** | Container user (1000) cannot read certs. | sudo chown -R 1000:0 certs and chmod 644 the keys. |
| **Fleet Server: Connection Refused** | Agent trying to hit localhost. | Update Fleet Policy Output in Kibana to use the Container Name (es01) or Host IP. |
| **Agent: "Certificate Verify Failed"** | SSL mismatch (Name vs IP). | Add ssl.verification_mode: none to the Output Policy YAML. |
| **Logs: "SSL Verification Disabled" spam** | Warning level logging. | Change Agent Policy Logging Level to **Error**. |
| **Kibana: Fleet Plugin Disabled** | Changed encryption keys after first boot. | Run docker compose down -v to wipe the corrupted Kibana DB. |
