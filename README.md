# Terraform3
Terraform

```bash
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
```tf
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
```tf
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
```tf
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
```tf
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
```tf
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
```tf
  metadata = {
    user-data = templatefile("my.tpl",{
    f_name = "John",
    l_name = "Os",
    names = ["Vasja","Petja","Tom","Peter","Uta" ]
    })
  }
```
my.tpl
```tpl
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
```tf
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
```tf
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
```tf
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
```tf
Создаем из переменных в variables
locals {
full_name = "${var.environment} - ${var.project_name}"
}

Используем
Project = local.full.name
```
**Запуск локальных команд - exec-local** (на компе с терраформом)
```tf
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
**Использовние Conditions и Lookups**

**Conditions**
```tf
variable "env" {
  default = "prod"
}

 platform_id = var.env == "prod" ? "standard-v3" : "standard-v2"

count = var.env == "prod" ? 1 : 0
```
**Lookups**
```tf
variable "look" {
    default = {
        "prod" = "standard-v3"
        "staging" = "standard-v2"
        "dev" = "standard-v1"
    }

platform_id = lookup(var.look, "prod") = platform_id = lookup(var.look, var.env)

```
**Использование циклов - count, for**

**count**
```tf
variable "aws_users" {
  default = ["petja", "vasja", "kolja", "otto", "maga", "john", "joos"]
}


resource "aws_iam_user" "users" {
  count = length(var.aws_users)
  name = element(var.aws_users, count.index)
}
```
**for**
```tf
output "created_iam_users_all" {
  value = aws_iam_user.users
}

output "created_iam_users_id" {
  value = aws_iam_user.users[*].id
}

output "created_iam_users_custom" {
  value = [
    for user in aws_iam_user.users:
      "Username: ${user.name} has ARN:${user.arn}"
  ]
}

output "created_iam_users_custom" {
  value = [
    for user in aws_iam_user.users:
       user.unique_id => user.id
  ]
}

output "created_iam_users_custom" {
  value = [
    for x in aws_iam_user.users:
      x.user
    if length(x.name) == 4
  ]
}

output "server_id_ip" {
  value = {
    for server in aws_instance.servers :
      server.id => server.public.ip
  }
}

output "all_info_server" {
  value = yandex_compute_instance.vm-1[*]
}

output "id_ip_servers" {
  value = {
    for server in yandex_compute_instance.vm-1:
    server.id => server.network_interface.0.nat_ip_address
  }
}
```
**Использование Terraform Remote State** - для работы нескольких человек / сохранение файла о состоянии в облаке
```tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"

    backend "s3" {
    endpoint   = "storage.yandexcloud.net"
    bucket     = "test-joos"
    region     = "ru-central1"
    key        = "terraform1.tfstate"
    shared_credentials_file = "storage.key" # - у пользователя создаем новый статический ключ и копируем в файл

    skip_region_validation      = true
    skip_credentials_validation = true
  }
}
```
**Создание Модулей**

https://cloud.yandex.ru/docs/tutorials/infrastructure-management/terraform-modules

https://github.com/terraform-yc-modules

**Global Variables**

Делаем outputs через бакеты и потом обращаемся к ним.

https://www.youtube.com/watch?v=nFmOqZxah_Q&list=PLg5SS_4L6LYujWDTYb-Zbofdl44Jxb2l8&index=34

**Как управлять ресурсами созданными вручную**
```
terraform import aws_instans.myname server-id

terraform import yandex_compute_instance.test epdsei3idc7heqpqonis
```
Создает файл .tfstate, но в данной версии не меняет main.tf, для дальнейшей работы надо редактировать main.tf
```
terraform apply
```
Смотрим ошибки которые выдает - добавляем нужное, потом смотрим из-за чего хочет убить сервер и правим файл чтобы apply был без изменений

**Как пересоздать ресурс до v0.15.2**
```
terraform taint yandex_compute_instance.vm
помечаем ресурс taint - при следующем apply терраформ удалит и создаст его
```
**Как пересоздать ресурс после v0.15.2**
```
terraform apply -replace  yandex_compute_instance.vm
сразу пересоздает ресурс
```
**terraform state команды**
```
terraform state show yandex_compute_instance.vm - покажет конфигурацию выбранного ресурса
terraform state list - покажет все ресурсы которые есть
terraform state pull - покажет все конфигурации по всем ресурсам
terraform state rm yandex_vpc_network.network-1 -  удаление ресурса из удаленного репозитория
terraform state mv -state-out="terraform.tfstate" yandex_vpc_network.network-1 yandex_vpc_network.network-1 - перемещение (и удаление) из удаленного файла (в бакете) в локальный terraform.tfstate

for i in $(terraform state list); do terraform state mv -state-out="terraform.tfstate" $i $i; done - в linux перебираем все ресурсы и перемещаем их в локальный terraform.tfstate

terraform state push terraform.tfstate - перезаписывает удаленный файл из локального
```



