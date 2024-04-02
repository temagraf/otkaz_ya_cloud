# Домашнее задание к занятию 10.7 «Отказоустойчивость в облаке»

 ---

## Задание 1 

Возьмите за основу [задание 1 из модуля 7.3 «Подъём инфраструктуры в Яндекс Облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1).

Теперь вместо одной виртуальной машины сделайте terraform playbook, который:

- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://www.terraform.io/docs/language/meta-arguments/count.html) для создания таких ресурсов;
- создаст [таргет-группу](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_target_group). Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.

Рекомендую почитать [документацию сетевого балансировщика](https://cloud.yandex.ru/docs/network-load-balancer/quickstart) нагрузки для того, чтобы было понятно, что вы сделали.

Далее установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

Далее перейдите в веб-консоль Yandex Cloud и убедитесь, что: 

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

*В качестве результата пришлите:*

*1. Terraform Playbook.*
main.tf:
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
  token = "y0_AgAAAABEezXuAATuwQAAAADXV8wjHipDbw-1ScaUnNdqKClO2Z3Ykoo"
  cloud_id = "b1gm7bp2grqbho73ke1l"
  folder_id = "b1g7l8ost873t9a9r3g3"
  zone = "ru-central1-b"
}
resource "yandex_compute_instance" "vm" {
  count = 2
  name = "vm${count.index}"


  resources {
    core_fraction = 20
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8a67rb91j689dqp60h"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }
  
  metadata = {
    user-data = "${file("./meta.yaml")}"
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
  name      = "target-1"

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
    name = "listener"
    port = 80
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

meta.yaml:
```
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
  - name: golpa
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC08CZafmwhOejaF/aDJBhWPT6KJeZ1vLVd+rryu7oMl3v7yLAgtMJca5tDI61cv+KA3V4fGi9YeTRUiWeXvvU47xbADb9GXslaA4aM1jj9SjBWXfcUyQFxCVlv/IE/u7o9c0spSzH3QFHMYBrTU912p9LQz8J23+mj0Xsloxz911AqMZVsjyhNIEqJPUWkmOjzjMzcqP5HThmEOEVkoveDmOgVU/20jZEHQsgOAgg66tWzH15AvwIWfjzLihABXQBSGsWnSxfgeGpQkfOBREWbwpsDXHEmg6vGdDhX1r34CPC33YqA9WMeJo+a+mafKEdUBT0gcjehj1lpN4BZC39CaFFLOeithukg1fXtrgAmPwEMxkV+hfb3JBc8kXD688Rw/hyOMRHbqChAerjPjSQnUdB/ulHLIVxnCeb9LSrUcpPxUD8GF8jHF9TRx5Yu9McY7RoBT3V834fCOD1ggOZ1OVnID1Hk3v3DYOialWJf7ZwoUnZBzd051oE5vIPMx70= golpa@#cloud-config
 users:
  - name: golpa
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC08CZafmwhOejaF/aDJBhWPT6KJeZ1vLVd+rryu7oMl3v7yLAgtMJca5tDI61cv+KA3V4fGi9YeTRUiWeXvvU47xbADb9GXslaA4aM1jj9SjBWXfcUyQFxCVlv/IE/u7o9c0spSzH3QFHMYBrTU912p9LQz8J23+mj0Xsloxz911AqMZVsjyhNIEqJPUWkmOjzjMzcqP5HThmEOEVkoveDmOgVU/20jZEHQsgOAgg66tWzH15AvwIWfjzLihABXQBSGsWnSxfgeGpQkfOBREWbwpsDXHEmg6vGdDhX1r34CPC33YqA9WMeJo+a+mafKEdUBT0gcjehj1lpN4BZC39CaFFLOeithukg1fXtrgAmPwEMxkV+hfb3JBc8kXD688Rw/hyOMRHbqChAerjPjSQnUdB/ulHLIVxnCeb9LSrUcpPxUD8GF8jHF9TRx5Yu9McY7RoBT3V834fCOD1ggOZ1OVnID1Hk3v3DYOialWJf7ZwoUnZBzd051oE5vIPMx70= golpa@DESKTOP-U7UHP4K

```

*2. Скриншот статуса балансировщика и целевой группы.*
![10-7-1](./hw-10-7/10-7-1.png)
![10-7-2](./hw-10-7/10-7-2.png)
![10-7-4](./hw-10-7/10-7-4.png)

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*
![10-7-3](./hw-10-7/10-7-3.png)

---


