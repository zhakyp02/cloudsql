# Managing Cloud SQL failover in Terrafrom

+ You need to add random_id and outputs to your master and read-replica in the child module.
+ You can also remove all zones in both the child and root modules.

outputs.tf

```hcl
output "master_instance_name" {
  description = "The name for Cloud SQL instance"
  value       = google_sql_database_instance.postgres_db_instance.name
}

output "master_region" {
  description = "The name for Cloud SQL instance"
  value       = google_sql_database_instance.postgres_db_instance.region
}

```

main.tf

```hcl
resource "random_id" "db_name_suffix_master" {
  byte_length = 2
}

resource "google_sql_database_instance" "postgres_db_instance" {
  provider = google-beta
  project  = local.project_id
  name     = "${local.instance_name}-${random_id.db_name_suffix_master.hex}"

  database_version    = var.database_version
  region              = var.region
  encryption_key_name = var.encryption_key_name
  deletion_protection = var.deletion_protection
  settings {
    tier              = var.tier
    activation_policy = var.activation_policy
    availability_type = var.availability_type


    dynamic "backup_configuration" {
      for_each = [var.backup_configuration]
      content {
        binary_log_enabled             = false
        enabled                        = local.backups_enabled
        start_time                     = lookup(backup_configuration.value, "start_time", null)
        location                       = lookup(backup_configuration.value, "location", null)
        point_in_time_recovery_enabled = local.point_in_time_recovery_enabled
        transaction_log_retention_days = lookup(backup_configuration.value, "transaction_log_retention_days", null)

        dynamic "backup_retention_settings" {
          for_each = local.retained_backups != null || local.retention_unit != null ? [var.backup_configuration] : []
          content {
            retained_backups = local.retained_backups
            retention_unit   = local.retention_unit
          }
        }
      }
    }

    dynamic "ip_configuration" {
      for_each = [local.ip_configurations[local.ip_configuration_enabled ? "enabled" : "disabled"]]
      content {
        ipv4_enabled       = lookup(ip_configuration.value, "ipv4_enabled", null)
        private_network    = lookup(ip_configuration.value, "private_network", null)
        require_ssl        = lookup(ip_configuration.value, "require_ssl", true)
        allocated_ip_range = lookup(ip_configuration.value, "allocated_ip_range", null)

        dynamic "authorized_networks" {
          for_each = lookup(ip_configuration.value, "authorized_networks", [])
          content {
            expiration_time = lookup(authorized_networks.value, "expiration_time", null)
            name            = lookup(authorized_networks.value, "name", null)
            value           = lookup(authorized_networks.value, "value", null)
          }
        }
      }
    }



    dynamic "insights_config" {
      for_each = var.insights_config != null ? [var.insights_config] : []
      content {
        query_insights_enabled  = true
        query_string_length     = lookup(insights_config.value, "query_string_length", 1024)
        record_application_tags = lookup(insights_config.value, "record_application_tags", false)
        record_client_address   = lookup(insights_config.value, "record_client_address", false)
      }
    }
    disk_autoresize       = var.disk_autoresize
    disk_autoresize_limit = var.disk_autoresize_limit
    disk_size             = var.disk_size
    disk_type             = var.disk_type
    pricing_plan          = var.pricing_plan
    dynamic "database_flags" {
      for_each = local.database_flags
      content {
        name  = lookup(database_flags.value, "name", null)
        value = lookup(database_flags.value, "value", null)
      }
    }

    user_labels = var.user_labels



    maintenance_window {
      day          = var.maintenance_window_day
      hour         = var.maintenance_window_hour
      update_track = var.maintenance_window_update_track
    }
  }
  # Other configurations ...

  lifecycle {
    ignore_changes = all

  }

  timeouts {
    create = var.create_timeout
    update = var.update_timeout
    delete = var.delete_timeout
  }

  depends_on = [
    null_resource.module_depends_on
  ]
}
```

read_replica.tf

```hcl
resource "random_id" "db_name_suffix_replica" {
  byte_length = 2
}

resource "google_sql_database_instance" "replicas" {
  provider             = google-beta
  for_each             = local.replicas
  project              = var.project_id
  name                 = "${each.value.name}-${random_id.db_name_suffix_replica.hex}"
  database_version     = var.database_version
  region               = join("-", slice(split("-", lookup(each.value, "region", var.region)), 0, 2))
  master_instance_name = var.master_instance_name
  deletion_protection  = var.deletion_protection
  encryption_key_name  = var.region == join("-", slice(split("", lookup(each.value, "region", var.region)), 0, 2)) ? each.value.encryption_key : null

  replica_configuration {
    failover_target = false
  }

  settings {
    tier              = lookup(each.value, "tier", var.tier)
    activation_policy = "ALWAYS"
    availability_type = lookup(each.value, "availability_type", var.availability_type)

    dynamic "ip_configuration" {
      for_each = [lookup(each.value, "ip_configuration", {})]
      content {
        ipv4_enabled       = lookup(ip_configuration.value, "ipv4_enabled", null)
        private_network    = lookup(ip_configuration.value, "private_network", null)
        require_ssl        = lookup(ip_configuration.value, "require_ssl", true)
        allocated_ip_range = lookup(ip_configuration.value, "allocated_ip_range", null)

        dynamic "authorized_networks" {
          for_each = lookup(ip_configuration.value, "authorized_networks", [])
          content {
            expiration_time = lookup(authorized_networks.value, "expiration_time", null)
            name            = lookup(authorized_networks.value, "name", null)
            value           = lookup(authorized_networks.value, "value", null)
          }
        }
      }
    }
    dynamic "insights_config" {
      for_each = var.insights_config != null ? [var.insights_config] : []
      content {
        query_insights_enabled  = true
        query_string_length     = lookup(insights_config.value, "query_string_length", 1024)
        record_application_tags = lookup(insights_config.value, "record_application_tags", false)
        record_client_address   = lookup(insights_config.value, "record_client_address", false)
      }
    }

    disk_autoresize       = lookup(each.value, "disk_autoresize", var.disk_autoresize)
    disk_autoresize_limit = lookup(each.value, "disk_autoresize_limit", var.disk_autoresize_limit)
    disk_size             = lookup(each.value, "disk_size", var.disk_size)
    disk_type             = lookup(each.value, "disk_type", var.disk_type)
    pricing_plan          = "PER_USE"
    user_labels           = lookup(each.value, "user_labels", var.user_labels)

    dynamic "database_flags" {
      for_each = lookup(each.value, "database_flags", [])
      content {
        name  = lookup(database_flags.value, "name", null)
        value = lookup(database_flags.value, "value", null)
      }
    }



  }

  lifecycle {
    ignore_changes = all
  }

  timeouts {
    create = var.create_timeout
    update = var.update_timeout
    delete = var.delete_timeout
  }

}
```

outputs.tf

```hcl
output "replica_instance_name" {
  description = "The names for Cloud SQL replica instances"
  value = join(", ", [for replica in google_sql_database_instance.replicas : replica.name])
}

output "replica_region" {
  description = "The regions for Cloud SQL replica instances"
  value = join(", ", [for replica in google_sql_database_instance.replicas : replica.region])
}

```

+ root module tfvars and main.tf

terraform.tfvar

```hcl
region2          = "us-central1"
region1          = "us-east4"
```

main.tf

```hcl
module "cloudsql_postgres_sync_test" {
  source     = "../../child/master"
  project_id = var.project_id
  region     = var.region1
  # network            = var.network
  database_version = var.database_version
  # proxy_subnet                       = var.subnet_link
  tier                               = var.tier
  identifier                         = "03"
  additional_access_service_accounts = ["sa-proj-default@${var.project_id}.iam.gserviceaccount.com"]
  user_labels                        = var.user_labels
  database_flags                     = var.database_flags

  ip_configuration = {
    "authorized_networks" : var.authorized_networks
    "ipv4_enabled" : true,
    "private_network" : null #var.network,
    "require_ssl" : var.require_ssl
    "allocated_ip_range" : null
  }
  additional_database    = var.additional_database
  add_database_name      = var.add_database_name
  add_database_charset   = var.add_database_charset
  add_database_collation = var.add_database_collation

}


module "cloudsql_postgres_rr_test" {
  source                   = "../../child/readreplica-child"
  project_id               = var.project_id
  region                   = var.region2
  database_version         = var.database_version
  master_instance_name     = module.cloudsql_postgres_sync_test.master_instance_name
  tier                     = var.tier
  read_replicas            = var.read_replicas
  read_replica_name_suffix = var.read_replica_name_suffix
}

```

variables.tf

```hcl
variable "region1" {
  type        = string
  description = "Region of the GCP project"
}

variable "region2" {
  type        = string
  description = "Region of the GCP project"
  default     = "us-central1"
}

```

Add these stages to your GitLab pipeline.

.gitlab-ci.yml

```yml
image:
name: hashicorp/terraform:light
entrypoint: - '/usr/bin/env' - 'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'

before_script:
- terraform --version
- export GOOGLE_APPLICATION_CREDENTIALS=${SERVICEACCOUNT}
- cd ${TF_ROOT}
- terraform init

stages:

- promote-tf-reconcile
- tf-new-read-replica


promote-tf-reconcile:
  stage: promote-tf-reconcile
  script: |
     read_replica=$(terraform output replica_instance_name)
     read_replica=$(echo $read_replica | tr -d '"')
     terraform state rm module.cloudsql_postgres_sync_test.google_sql_database_instance.postgres_db_instance
     terraform state rm module.cloudsql_postgres_rr_test.google_sql_database_instance.replicas
     terraform state rm module.cloudsql_postgres_sync_test.google_sql_database.additional_databases
     terraform state rm module.cloudsql_postgres_sync_test.random_password.postgres_user_password
     terraform state rm module.cloudsql_postgres_rr_test.random_id.db_name_suffix_replica
     terraform state list
     terraform import "module.cloudsql_postgres_sync_test.google_sql_database_instance.postgres_db_instance" "projects/<google_project_id>/instances/$read_replica"
     terraform import "module.cloudsql_postgres_sync_test.google_sql_database.additional_databases[0]" "projects/<google_project_id>/instances/${read_replica}/databases/additional-database"
      terraform plan -var region1=us-central1 -var region2=us-east4 -out "planfile1"
  artifacts:
    paths:
      - planfile1
  when: manual

tf-new-read-replica:
  stage: tf-new-read-replica
  script:
    - terraform apply -input=false "planfile1"
  dependencies:
    #- promote-tf-reconcile
  when: manual


```
