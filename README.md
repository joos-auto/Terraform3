# Terraform3
Terraform

```
wget https://hashicorp-releases.yandexcloud.net/terraform/1.5.4/terraform_1.5.4_linux_amd64.zip
zcat terraform_1.5.4_linux_amd64.zip > terraform
chmod -x terraform
chmod 766 terraform
./terraform -v
cp terraform /usr/local/bin/
cd ~
nano .terraformrc
---
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```
main.tf
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  zone = "ru-central1-a"
}
```
```
terraform init
```

main.tf - пример
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  token = "oauth_token"
  cloud_id = "cloud_id"
  folder_id = "folder_id"
  zone = "ru-central1-a"
}

resource "yandex_compute_instance" "vm-1" {
  name = "terraform1"
  platform_id = "standard-v3"
  resources {
    core_fraction = 20
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8sqojvm458b3jr5nfd"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }

}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}
```
Создаем 3 компьютера
```
variable "name" {
    default = ["name1", "name2", "name3"]
}

resource "yandex_compute_instance" "vm-1" {
  for_each   = toset(var.name)
  name       = each.key
  platform_id = "standard-v3"
  resources {
    core_fraction = 20
    cores  = 2
    memory = 2
  }

```
Или
```
resource "yandex_compute_instance" "vm-1" {
  count = 3
  name  = "terr-${count.index}"
  platform_id = "standard-v3"
  resources {
    core_fraction = 20
    cores  = 2
    memory = 2
  }
```
