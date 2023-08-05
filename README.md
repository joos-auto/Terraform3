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
**main.tf - пример**
```
terraform {
  required_providers {
    yandex   = {
      source = "yandex-cloud/yandex"
    }
  }
}
provider "yandex" {
  token     = "oauth_token"
  cloud_id  = "cloud_id"
  folder_id = "folder_id"
  zone = "ru-central1-b"
}
resource "yandex_compute_instance" "vm-1" {
  name        = "terraform1"
  platform_id = "standard-v3"
  resources {
    core_fraction = 20
    cores     = 2
    memory    = 2
  }
  boot_disk {
    initialize_params {
      image_id = "fd8dvus8s5qjad7td8p4"
    }
  }
  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }
  metadata = {
    user-data = "${file("./my.sh")}"
  }
}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  v4_cidr_blocks = ["192.168.10.0/24"]
  network_id     = "${yandex_vpc_network.network-1.id}"
}

```
**Создаем 3 компьютера**
```
variable "name" {
    default = ["name1", "name2", "name3"]
}

resource "yandex_compute_instance" "vm-1" {
  for_each    = toset(var.name)
  name        = each.key
  platform_id = "standard-v3"
  resources {
    core_fraction = 20
    cores     = 2
    memory    = 2
  }

```
**Или**
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
**Подключаем исполняемый файл**
```
  metadata = {
    user-data = "${file("./my.sh")}"
  }
```
my.sh
```bash
#!/bin/bash
cat /etc/*rel*
yum -y update
yum -y install httpd
yum -y curl
myip=`curl http://ifconfig.me`
echo "<h2>WebServer with IP: $myip</h2><br>Build by Terraform!" > /var/www/html>
sudo service httpd start
chkconfig httpd on
```
**Использование Динамичных внешних файлов - templatefile**
```
  metadata = {
    user-data = templatefile("my.tpl",{
    f_name = "John",
    l_name = "Os",
    names = ["Vasja","Petja","Tom","Peter","Uta" ]
    })
  }
```
my.tpl
```
#!/bin/bash
cat /etc/*rel*
yum -y update
yum -y install httpd

cat <<EOF > /var/www/html/index.html
<html>
<h2>Build by Power of Terraform <font color="red"> v0.999</font></h2><br>
Owner ${f_name} ${l_name} <br>
%{ for x in names ~}
Hello to ${x}
%{ endfor ~}
</html>
EOF

sudo service httpd start
chkconfig httpd on
```
Проверка без сервера
```
terraform console - Запускает консоль
templatefile("my.tpl",{f_name = "John", l_name = "Os", names = ["Vasja","Petja","Tom","Peter","Uta" ] }) - выведет что будет изменено
```
**LifeCycle - AWS**
```
не дает удалять сервер или компоненты
lifecycle {
prevent_destroy = true
}

игнорирует изменения и не перезапускает сервер
lifecycle {
ignore_changes = ["user_date"]
}

Создаем и привязываем статический ip и настройка - сначало поднимаем, потом убиваем сервер
resource "aws_eip" "my_static_ip" {
instance = aws_instance.my_server.id
}
lifecycle {
create_before_destroy = true
}
```
**outputs.tf**
```
output "internal_ip_address_vm-1" {
  value = yandex_compute_instance.vm-1.network_interface.0.ip_address
}
output "external_ip_address_vm-1" {
  value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
}
```
```
terraform show - покажет что получилось apply в том числе и outputs
```
**Порядок создания Ресурсов**
```
depends_on = [yandex_compute_instance.vm-1, server_name2]
```
**Получение данных с помощью Data Source**

https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/data-sources/

**Использование Переменных - variables**

variables.tf
```
variable "region" {
description = "Please enter Region AWS"
default     = "ca-central-1"
}

variable "platform_id" {
description = "Enter platform Yandex"
type = string
default = "standard-v3"
}
```
**Автозаполнение Переменных - tfvars**
```
terraform apply -var="platform_id=standart-v2" - при запуске

EXPORT TF_VAR_platform_id=standart-v2 - пишем в сессию терминала и можем при запуске ничего не вводить, терраформ подхватит сам
ent | grep TF_VAR_ - смотрим
unset TF_VAR_platform_id - удаляет var из сессии
```
terraform.tfvars - приоритетнее variables.tf - может использоваться для продакшена

**Локальные Переменные - locals**
```
Создаем из переменных в variables
locals {
full_name = "${var.environment} - ${var.project_name}"
}

Используем
Project = local.full.name
```
**Запуск локальных команд - exec-local** (на компе с терраформом)
```
resource "null_resource" "command1" {
    provisioner "local-exec" {
        command = "echo Terraform START: $(date) >> log.txt"
    }
}
Запуск через Python
resource "null_resource" "command2" {
    provisioner "local-exec" {
        command = "print('Hello World!')"
        interpreter = ["python", "-c"]
    }
}

resource "null_resource" "command2" {
    provisioner "local-exec" {
        command = "echo $NAME1 $NAME2 $NAME3 >> names.txt"
        environment {
          NAME1 = "Vasya"
          NAME2 = "Petja"
          NAME3 = "Tanja"
        }
    }
}

provisioner можно запихнуть в создание компьютера и он выведет при создании эту команду
```


