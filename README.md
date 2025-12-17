# Kessel Ansible Playbooks

Ansible playbooks to deploy Kessel infrastructure using Podman containers.

## Prerequisites

- Podman
- Ansible with `containers.podman` collection
- Go (for schema preloading)
- KSL compiler (for generating schema.zed)

Install the Ansible collection:

```
ansible-galaxy collection install containers.podman
```

## Services

The playbooks deploy the following services:

| Service | Ports | Description |
|---------|-------|-------------|
| PostgreSQL | 5432 | Database for inventory-api and SpiceDB |
| Kafka | 9092 (external), 9093 (internal) | Message broker (KRaft mode, no Zookeeper) |
| Kafka Connect | 8083 | Debezium CDC connector |
| Inventory API | 8081 (HTTP), 9081 (gRPC) | Kessel inventory service |
| SpiceDB | 50051 (gRPC), 8080 (HTTP), 9090 (metrics) | Authorization database |
| Relations API | 8000 (HTTP), 9000 (gRPC) | Kessel relations service |

## Usage

### 1. Generate Schema (first time or after changes)

Build the ZED schema from KSL files:

```
cd ksl
make build
```

Generate inventory-api schema cache:

```
ansible-playbook schema-setup.yaml
```

This playbook:
1. Creates the schema directory structure in inventory-api
2. Templates config.yaml and copies JSON schema files
3. Runs `go run main.go preload-schema` in the inventory-api directory

The preload-schema command reads the schema files from `data/schema/resources/` and generates `schema_cache.json` at the repository root. This cache file is mounted into the inventory-api container at runtime.

Requirements:
- Go installed and in PATH
- inventory-api repository at `../inventory-api` (relative to ansible directory)

### 2. Start Infrastructure

```
ansible-playbook kessel-infra.yaml
```

### 3. Stop Infrastructure

```
ansible-playbook kessel-infra-down.yaml
```

This removes all containers and volumes.

## Configuration

All configuration is in `vars/main.yaml`:

### Repository Paths

| Variable | Default |
|----------|---------|
| inventory_api_path | ../inventory-api |
| document_schema_path | templates/document |
| ksl_schema_path | ksl |

### Schema Configuration

| Variable | Default |
|----------|---------|
| resource_type | document |
| reporter_name | drive |
| namespace | drive |

### Network

| Variable | Default |
|----------|---------|
| network_name | kessel |

### PostgreSQL

| Variable | Default |
|----------|---------|
| postgres_image | postgres:16 |
| postgres_container | invdatabase |
| postgres_port | 5432 |
| postgres_user | postgres |
| postgres_password | postgres |
| postgres_db | spicedb |

### Kafka

| Variable | Default |
|----------|---------|
| kafka_image | quay.io/strimzi/kafka:latest-kafka-3.8.0 |
| kafka_container | kafka |
| kafka_internal_port | 9093 |
| kafka_external_port | 9092 |
| kafka_cluster_id | MkU3OEVBNTcwNTJENDM2Qk |

### Inventory API

| Variable | Default |
|----------|---------|
| inventory_api_image | quay.io/cloudservices/kessel-inventory:latest |
| invmigrate_container | invmigrate |
| inventory_api_container | inventory-api |
| inventory_http_port | 8081 |
| inventory_grpc_port | 9081 |
| inventory_config_path | templates/inventory-api.yaml |

### Kafka Connect (Debezium)

| Variable | Default |
|----------|---------|
| connect_image | quay.io/debezium/connect:3.1 |
| connect_container | connect |
| connect_port | 8083 |
| debezium_config_path | templates/debezium-connector.json |

### SpiceDB

| Variable | Default |
|----------|---------|
| spicedb_image | authzed/spicedb |
| spicedb_container | spicedb |
| spicedb_grpc_port | 50051 |
| spicedb_http_port | 8080 |
| spicedb_metrics_port | 9090 |
| spicedb_preshared_key | foobar |

### Relations API

| Variable | Default |
|----------|---------|
| relations_api_image | quay.io/cloudservices/kessel-relations:latest |
| relations_api_container | relations-api |
| relations_api_http_port | 8000 |
| relations_api_grpc_port | 9000 |
| relations_api_schema_path | ksl/schema.zed |

## File Structure

```
ansible/
  kessel-infra.yaml       # Main playbook to start services
  kessel-infra-down.yaml  # Teardown playbook
  schema-setup.yaml       # Schema generation playbook
  vars/
    main.yaml             # Variables
  templates/
    inventory-api.yaml    # Inventory API config template
    debezium-connector.json
    document/             # Document schema templates
  ksl/
    kessel.ksl            # KSL schema files
    rbac.ksl
    drive.ksl
    Makefile              # Builds schema.zed
```

## Startup Order

1. PostgreSQL
2. Kafka (KRaft mode)
3. Inventory API migration
4. SpiceDB migration
5. Kafka topics
6. Kafka Connect
7. Inventory API
8. Debezium connector
9. SpiceDB server
10. Schema write to SpiceDB
11. Relations API

## Troubleshooting

Check container logs:

```
podman logs <container-name>
```

List running containers:

```
podman ps
```

Check network:

```
podman network inspect kessel
```
