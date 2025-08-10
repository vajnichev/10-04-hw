# Домашнее задание к занятию "Отказоустойчивость в облаке" - `Важничев Георгий`


### Задание 1
Возьмите за основу [решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1).

1. Теперь вместо одной виртуальной машины сделайте terraform playbook, который:

- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://www.terraform.io/docs/language/meta-arguments/count.html) для создания таких ресурсов;
- создаст [таргет-группу](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_target_group). Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.

Рекомендуем изучить [документацию сетевого балансировщика нагрузки](https://cloud.yandex.ru/docs/network-load-balancer/quickstart) для того, чтобы было понятно, что вы сделали.

2. Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

3. Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

*В качестве результата пришлите:*

*1. Terraform Playbook.*

*2. Скриншот статуса балансировщика и целевой группы.*

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*


#### Решение:
1. `Terraform Playbook.`
[main.tf](https://github.com/vajnichev/10-04-hw/blob/main/main.tf) 
```bash
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token     = "y0__xDBzdQ7GMHdEyDrrp-GFIjaYdpV4lGB89Z-29EdyfUQQIGw"
  cloud_id  = "b1gu77i5umb2gemb1umu"
  folder_id = "b1gvj3vtb2f3jplh1ne7"
  zone      = "ru-central1-b"
}

resource "yandex_compute_instance" "vm" {
  count = 2
  name  = "vm${count.index}"
  hostname = "vm${count.index}"
  resources {
    cores         = 2
    memory        = 4
  }

  boot_disk {
    initialize_params {
      image_id = "fd8huqdhr65m771g1bka"
      size     = 30
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }

  metadata = {
    user-data = "${file("./meta.yml")}"
  }

}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_lb_target_group" "target-1" {
  name = "target-1"

  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[0].network_interface.0.ip_address
  }

  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[1].network_interface.0.ip_address
  }

}

resource "yandex_lb_network_load_balancer" "lb-1" {
  name = "lb1"

  listener {
    name                = "listener"
    port                = 80
    protocol            = "tcp"
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.target-1.id
    healthcheck {
      name = "http"
        http_options {
          port = 80
          path = "/"
        }
    }
  }
}

output "internal_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface.0.ip_address
}
output "external_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface.0.nat_ip_address
}
output "internal_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface.0.ip_address
}
output "external_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface.0.nat_ip_address
}

```
[meta.yml](https://github.com/vajnichev/10-04-hw/blob/main/meta.yml)
```bash

#cloud-config
disable_root: true
timezone: Europe/Moscow
repo_update: true
apt:
  preserve_sources_list: true
packages:
 - nginx
runcmd:
 - [ systemctl, nginx-reload ]
 - [ systemctl, enable, nginx.service ]
 - [ systemctl, start, --no-block, nginx.service ]
users:
 - name: vajnichev
   groups: sudo
   shell: /bin/bash
   sudo: ['ALL=(ALL) NOPASSWD:ALL']
   ssh-authorized-keys:
     - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwIeWawrtHFqGg1vFo6kzvv5ZqaPaRjoWaHTu5tqSivUm79LgPM4dK/ydcNeVykO6WNSjYcaCRa5l25EIJHkHRM8+9Zy1SqsIjqN4faV3tXbMMZNHyfh+kp+vSHRADfSjRbptRsebT8SdBuw1YAJf9g4M+G4J/eiysp4e9ARIb7BRWyaJg6LZ35T09QSeutO8KZHDHEzQ6TiLtbFHo9OnFSC4HRORecGN9YQJmIOPkCB31mXLSm9cjFqHD7Vugpx3SwKufp+wt3C7zBh/jsMRgsF3NtOAigH0HhGKvGB+BeQgLH3/Oro7aFgbE8GAoFxaPyqWkAZMdTXISljOlyBuZ georgiy@georgiy-ubuntu

```
2. `Скриншот статуса балансировщика и целевой группы.`
![png](https://github.com/vajnichev/10-04-hw/blob/main/IMG/10.4.1.png)

3. `Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.`
![png](https://github.com/vajnichev/10-04-hw/blob/main/IMG/10.4.2.png)
![png](https://github.com/vajnichev/10-04-hw/blob/main/IMG/10.4.3.png)
![png](https://github.com/vajnichev/10-04-hw/blob/main/IMG/10.4.4.png)
![png](https://github.com/vajnichev/10-04-hw/blob/main/IMG/10.4.5.png)
