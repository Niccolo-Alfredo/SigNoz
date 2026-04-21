# Ansible SigNoz Stack

Playbook Ansible per il deploy e la gestione dello stack di osservabilità SigNoz su Rocky Linux 9.
Sviluppato come playbook standalone testabile indipendentemente, predisposto per l'integrazione nel repository `ansible-infra`.

## Stack

| Componente | Versione | Descrizione |
|---|---|---|
| Apache ZooKeeper | 3.8.6 (LTS) | Coordination service per ClickHouse replication |
| ClickHouse | 25.8.x (LTS) | Backend OLAP per traces, metrics e logs |
| SigNoz | v0.119.0 | Piattaforma di osservabilità OpenTelemetry-native |
| SigNoz OTel Collector | v0.129.13 | Collector per ingestione traces, metrics e logs |

## Struttura

```
ansible-signoz/
├── ansible.cfg
├── site.yml                          # Playbook orchestratore
├── inventory/
│   └── hosts.ini                     # Host del gruppo [signoz]
├── group_vars/
│   ├── signoz.yml                    # Variabili stack (single source of truth)
│   └── vault.yml                     # Secrets cifrati con ansible-vault
├── playbooks/
│   └── roles/
│       ├── zookeeper/                # Installazione ZooKeeper via tarball
│       ├── clickhouse/               # Installazione ClickHouse via RPM
│       │   └── tasks/
│       │       └── retention.yml     # TTL/retention ClickHouse
│       ├── signoz/                   # Installazione SigNoz + migrazioni ClickHouse
│       │   └── tasks/
│       │       └── migrations.yml    # Schema migrations via signoz-otel-collector
│       └── signoz_otel_collector/    # Installazione SigNoz OTel Collector
└── playbooks/
    └── signoz.yml
```

## Prerequisiti

### Controller Ansible

- Ansible Core >= 2.9
- Python >= 3.6
- Accesso SSH alla macchina target con sudo senza password
- `ssh-agent` configurato con la chiave SSH

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
```

### Target

- Rocky Linux 9 (x86_64)
- Accesso a internet per download pacchetti e binary
- Disco >= 100 GB (raccomandato 950 GB per retention configurata)
- RAM >= 8 GB

## Configurazione Iniziale

### 1. Inventory

Modifica `inventory/hosts.ini` con l'IP del target:

```ini
[signoz]
signoz01    ansible_host=<IP>    ansible_user=sys-admin
```

### 2. Variabili

Le variabili principali sono in `group_vars/signoz.yml`. I valori più rilevanti da verificare prima del deploy:

```yaml
clickhouse_version: "25.8.22.28"      # versione ClickHouse pinnata
signoz_otel_collector_version: "v0.129.13"  # versione OTel Collector pinnata
signoz_version: "v0.119.0"              # versione signoz pinnata
clickhouse_keep_free_space_bytes: 21474836480  # 20GB headroom disco
```

### 3. Vault (Secrets)

Crea il file vault con le credenziali:

```bash
ansible-vault create group_vars/vault.yml
```

Contenuto del file vault:

```yaml
---
vault_clickhouse_password: "<password>"
vault_signoz_jwt_secret: "<secret>"
```

Opzionale — configura il vault password file per evitare di digitare la password ad ogni run:

```bash
echo "<vault-password>" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

E aggiungi in `ansible.cfg`:

```ini
vault_password_file = ~/.vault_pass
```

## Utilizzo

### Deploy completo

```bash
ansible-playbook site.yml -v
```

Con vault password interattiva:

```bash
ansible-playbook site.yml --ask-vault-pass -v
```

### Deploy per ruolo singolo

```bash
ansible-playbook site.yml --tags zookeeper -v
ansible-playbook site.yml --tags clickhouse -v
ansible-playbook site.yml --tags signoz -v
ansible-playbook site.yml --tags signoz_otel_collector -v
```

### Deploy per fase

```bash
# Solo installazione binari
ansible-playbook site.yml --tags install -v

# Solo configurazione
ansible-playbook site.yml --tags configure -v

# Solo verifica stato servizi
ansible-playbook site.yml --tags verify -v

# Solo retention ClickHouse
ansible-playbook site.yml --tags retention -v
```

### Tag granulari (ruolo + fase)

```bash
ansible-playbook site.yml --tags zookeeper_install -v
ansible-playbook site.yml --tags clickhouse_configure -v
ansible-playbook site.yml --tags signoz_verify -v
ansible-playbook site.yml --tags signoz_otel_collector_verify -v
```

### Saltare le migrazioni (re-run veloce)

```bash
ansible-playbook site.yml --skip-tags migrations -v
```

### Forzare TTL merges (recovery)

```bash
ansible-playbook site.yml --tags retention -e "clickhouse_start_ttl_merges=true" -v
```

### Test per ruolo singolo

```bash
ansible-playbook playbooks/test_zookeeper.yml -v
ansible-playbook playbooks/test_clickhouse.yml -v
ansible-playbook playbooks/test_signoz.yml -v
ansible-playbook playbooks/test_signoz_otel_collector.yml -v
```

## Retention ClickHouse

La retention è configurata in `group_vars/signoz.yml` e applicata via `ALTER TABLE MODIFY TTL` su ClickHouse.

| Dominio | Default | Stima spazio |
|---|---|---|
| Traces | 30 giorni | ~240 GB |
| Logs | 60 giorni | ~9 GB |
| Metrics | 90 giorni | ~18 GB |
| System logs ClickHouse | 14 giorni | ~28 GB |

> **Baseline di riferimento**: ~19 GB traces + ~4.9 GB system + ~480 MB metrics + ~330 MB logs ogni 2-3 giorni su ambiente TCOS Milano.

> **Nota**: la retention per `logs_v2` e `logs_v2_resource` usa il meccanismo `_retention_days` nativo di SigNoz (stesso usato dalla UI). Le altre tabelle usano TTL fissa via `MODIFY TTL`. La UI Settings → Retention Controls può mostrare valori disallineati rispetto a quelli impostati via Ansible — i dati vengono comunque cancellati secondo la TTL reale su ClickHouse.

## SELinux

Rocky Linux 9 ha SELinux enforcing di default. I binary estratti da tarball `/tmp` ereditano il contesto `user_tmp_t` che impedisce l'esecuzione da systemd. Il playbook esegue `restorecon -Rv` dopo ogni copia di binary per correggere il contesto. Non sono necessarie policy custom.

## Note Operative

**ClickHouse GPG check disabilitato**: i pacchetti LTS 25.x e 26.x non sono firmati upstream (issue noto al 2026). Il check è disabilitato nel ruolo con commento esplicito. Da riabilitare quando il signing viene ripristinato upstream.

**Versione OTel Collector pinnata**: non usare `latest` per `signoz_otel_collector_version` — il link `/releases/latest` può puntare a tag source-only senza binary compilati. Verificare sempre la disponibilità del tarball prima di aggiornare la versione.

**Swap**: il ruolo ClickHouse disabilita lo swap (`swapoff -a` + commento in `/etc/fstab`) come raccomandato da ClickHouse per ambienti di produzione.

**Migrazioni**: le migrazioni ClickHouse (`bootstrap`, `sync up`, `async up`) sono idempotenti e vengono eseguite ad ogni run del ruolo `signoz`. Per saltarle su re-run: `--skip-tags migrations`.

## Integrazione in ansible-infra

Per integrare nel repository `ansible-infra` esistente:

1. Copiare `playbooks/roles/zookeeper`, `clickhouse`, `signoz`, `signoz_otel_collector` in `ansible-infra/playbooks/roles/`
2. Aggiungere il gruppo `[signoz]` in `ansible-infra/inventory/prod/hosts.ini`
3. Copiare `group_vars/signoz.yml` in `ansible-infra/inventory/prod/group_vars/`
4. Aggiungere i secret vault al vault esistente o creare `group_vars/vault.yml` separato
5. Copiare `playbooks/signoz.yml` in `ansible-infra/playbooks/`