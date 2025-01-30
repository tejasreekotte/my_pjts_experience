# Google Garage - GCP Infrastructure Automation with Terraform & Cloud Build

This project automates the provisioning of GCP infrastructure using **Terraform**, **Cloud Build**, **Artifact Registry**, and **Source Repository** for version control. It automates the process of creating Google Cloud resources like compute instances, disks, and static IPs, significantly reducing manual intervention.

## Project Overview

- **Terraform Automation**: Using Terraform scripts, GCP infrastructure is provisioned and managed efficiently.
- **Cloud Build Pipelines**: Automated workflows for provisioning resources through Cloud Build.
- **Artifact Registry**: For Docker image management.
- **Source Repository**: For version control and collaborative work.
- **ServiceNow Integration**: Cloud Build can be triggered based on ServiceNow incidents, enabling easy resource provisioning based on requests.

---

## Key Components

### 1. **Terraform Script: `main.tf`**

The following script provisions a Compute Instance, static IP, and additional disks in GCP:

```hcl
resource "google_compute_instance" "default" {
  provider     = google.google_container_registry_fix
  project      = var.project
  name         = var.instance_name
  machine_type = var.machine_type
  zone         = var.zone
  boot_disk {
      auto_delete = true
      device_name = var.device_name
      initialize_params {   
          type  = var.type
          image = var.image
          size  = var.size
      }
  }
  network_interface {
    network    = var.network
    access_config {
      nat_ip      = google_compute_address.static-ip.address
      network_tier = var.network_tier
    }
  }
  service_account {
    email  = "ssh-instance@exponense-automation.iam.gserviceaccount.com"
    scopes = ["storage-full"]
  }
}

resource "google_compute_address" "static-ip" {
  provider     = google.google_container_registry_fix
  name         = var.address_name
  address_type = var.address_type
  region       = var.region
  network_tier = var.network_tier
}

resource "google_compute_disk" "disks" {
  provider       = google.google_container_registry_fix
  name           = var.additional_disk_name
  type           = var.additional_disk_type
  size           = var.additional_disk_size
  zone           = var.zone
}

resource "google_compute_attached_disk" "disk1" {
  provider       = google.google_container_registry_fix
  disk           = google_compute_disk.disks.name
  instance       = google_compute_instance.default.self_link
}
```

### 2. **Provider Configuration: `provider.tf`**

The provider configuration specifies the credentials and the GCP project.

```hcl
provider "google" {
  credentials = "key.json"
  project     = var.project
}
```

### 3. **Variables: `vars.tf`**

This file contains all the necessary variables used in the Terraform configuration, such as instance name, machine type, disk size, etc.

```hcl
variable "project" {
  type        = string
  description = "GCP project ID"
}

variable "network" {
  type        = string
  description = "Name of VPC network"
}

variable "instance_name" {
  type        = string
  description = "Name of the compute instance"
}

variable "device_name" {
  type        = string
  description = "Name of boot disk"
}

# Additional variables for resources like disks, addresses, etc.
```

### 4. **Resource Creation Script: `resourcecreation.sh`**

This script automates the execution of Terraform commands and passes variables for dynamic resource creation.

```bash
#!/bin/bash
echo "Provisioning resources using Terraform"

instance_name=${1} 
project=${2}
network=${3}
zone=${4}
size=${5}
image=${6}
machine_type=${7}
device_name=${8}
type=${9}

# Address parameters
address_name=${10} 
address_type=${11}
region=${12}     
network_tier=${13}

# Additional disk parameters
additional_disk_name=${14}
additional_disk_type=${15}
additional_disk_size=${16}

terraform init
terraform apply --auto-approve \
  -var "project=$project" \
  -var "network=$network" \
  -var "instance_name=$instance_name" \
  -var "device_name=$device_name" \
  -var "type=$type" \
  -var "machine_type=$machine_type" \
  -var "image=$image" \
  -var "size=$((size))" \
  -var "zone=$zone" \
  -var "address_name=$address_name" \
  -var "address_type=$address_type" \
  -var "region=$region" \
  -var "network_tier=$network_tier" \
  -var "additional_disk_name=$additional_disk_name" \
  -var "additional_disk_type=$additional_disk_type" \
  -var "additional_disk_size=$((additional_disk_size))" 
```

### 5. **Cloud Build Configuration: `buildspec.yaml`**

The Cloud Build configuration runs the `resourcecreation.sh` script to provision resources, utilizing parameters passed as substitution variables.

```yaml
steps:
  - name: gcr.io/exponense-automation/terraform-build:latest
    entrypoint: "bash"
    args:
      - "-c"
      - |
        echo "Terraform resource provisioning" \
        && chmod 777 resourcecreation.sh \
        && ./resourcecreation.sh ${_INSTANCE_NAME} ${_PROJECT} ${_NETWORK} ${_ZONE} ${_SIZE} ${_IMAGE} ${_MACHINE_TYPE} ${_DEVICE_NAME} ${_TYPE} ${_ADDRESS_NAME} ${_ADDRESS_TYPE} ${_REGION} ${_NETWORK_TIER} ${_ADDITIONAL_DISK_NAME} ${_ADDITIONAL_DISK_TYPE} ${_ADDITIONAL_DISK_SIZE}
    id: Build task
```

---

## Cloud Build Trigger Setup

### Step 1: **Create a Cloud Build Trigger**

1. **Go to Cloud Build** in the GCP Console.
2. **Create a Trigger** with the following settings:
   - **Trigger Name**: `deploy-resources-trigger`
   - **Event Type**: Choose either **Push to a branch** or **Manual trigger**.
   - **Source Repository**: Select the GCP Source Repository where the Terraform code and `buildspec.yaml` reside.
   - **Build Configuration**: Select the `buildspec.yaml` file.
   - **Substitution Variables**: Add variables like `_INSTANCE_NAME`, `_PROJECT`, `_ZONE`, etc., to be passed to Terraform.
   - **Service Account**: Set the appropriate service account for executing the build and provisioning resources.

### Step 2: **Manual or Automatic Build Trigger**

- For **Push to Branch**, the build will be automatically triggered on code changes.
- For **Manual Trigger**, you can initiate the build manually via the Google Cloud Console or API.

---

## ServiceNow Integration (Optional)

ServiceNow can be integrated to automatically trigger Cloud Build based on an incident. This is achieved by:

1. **ServiceNow Incident Creation**: When an incident is created (e.g., a request to provision resources), the required parameters (e.g., instance name, region, etc.) are gathered.
2. **Trigger Cloud Build via API**: Use ServiceNow's REST API to trigger the Cloud Build job and pass necessary parameters as substitution variables.

Example API request:

```bash
curl -X POST \
  -H "Authorization: Bearer [YOUR_SERVICE_ACCOUNT_TOKEN]" \
  -H "Content-Type: application/json" \
  --data '{
    "source": {
      "repoSource": {
        "repoName": "your-repo-name",
        "branchName": "main"
      }
    },
    "substitutions": {
      "_INSTANCE_NAME": "new-instance",
      "_REGION": "us-central1",
      "_SIZE": "small"
    }
  }' \
  https://cloudbuild.googleapis.com/v1/projects/[PROJECT_ID]/triggers/[TRIGGER_ID]:run
```

---

## Conclusion

This project automates the provisioning of GCP resources using **Terraform** and **Cloud Build**, and can be triggered either automatically through repository changes or manually via ServiceNow incidents. By automating these processes, we reduce the complexity and time required to deploy GCP infrastructure, making it faster and more efficient.

