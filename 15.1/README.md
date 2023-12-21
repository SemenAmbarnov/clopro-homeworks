# Домашнее задание к занятию «Организация сети»

### Подготовка к выполнению задания

1. Домашнее задание состоит из обязательной части, которую нужно выполнить на провайдере Yandex Cloud, и дополнительной части в AWS (выполняется по желанию). 
2. Все домашние задания в блоке 15 связаны друг с другом и в конце представляют пример законченной инфраструктуры.  
3. Все задания нужно выполнить с помощью Terraform. Результатом выполненного домашнего задания будет код в репозитории. 
4. Перед началом работы настройте доступ к облачным ресурсам из Terraform, используя материалы прошлых лекций и домашнее задание по теме «Облачные провайдеры и синтаксис Terraform». Заранее выберите регион (в случае AWS) и зону.

---
### Задание 1. Yandex Cloud 

**Что нужно сделать**

1. Создать пустую VPC. Выбрать зону.
2. Публичная подсеть.

 - Создать в VPC subnet с названием public, сетью 192.168.10.0/24.
 - Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать fd80mrhj8fl2oe87o4e1.
 - Создать в этой публичной подсети виртуалку с публичным IP, подключиться к ней и убедиться, что есть доступ к интернету.
3. Приватная подсеть.
 - Создать в VPC subnet с названием private, сетью 192.168.20.0/24.
 - Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс.
 - Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее, и убедиться, что есть доступ к интернету.

Resource Terraform для Yandex Cloud:

- [VPC subnet](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_subnet).
- [Route table](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_route_table).
- [Compute Instance](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/compute_instance).

---
### Задание 2. AWS* (задание со звёздочкой)

Это необязательное задание. Его выполнение не влияет на получение зачёта по домашней работе.

**Что нужно сделать**

1. Создать пустую VPC с подсетью 10.10.0.0/16.
2. Публичная подсеть.

 - Создать в VPC subnet с названием public, сетью 10.10.1.0/24.
 - Разрешить в этой subnet присвоение public IP по-умолчанию.
 - Создать Internet gateway.
 - Добавить в таблицу маршрутизации маршрут, направляющий весь исходящий трафик в Internet gateway.
 - Создать security group с разрешающими правилами на SSH и ICMP. Привязать эту security group на все, создаваемые в этом ДЗ, виртуалки.
 - Создать в этой подсети виртуалку и убедиться, что инстанс имеет публичный IP. Подключиться к ней, убедиться, что есть доступ к интернету.
 - Добавить NAT gateway в public subnet.
3. Приватная подсеть.
 - Создать в VPC subnet с названием private, сетью 10.10.2.0/24.
 - Создать отдельную таблицу маршрутизации и привязать её к private подсети.
 - Добавить Route, направляющий весь исходящий трафик private сети в NAT.
 - Создать виртуалку в приватной сети.
 - Подключиться к ней по SSH по приватному IP через виртуалку, созданную ранее в публичной подсети, и убедиться, что с виртуалки есть выход в интернет.

Resource Terraform:

1. [VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc).
1. [Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet).
1. [Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway).

### Правила приёма работы

Домашняя работа оформляется в своём Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

# Выполнение

Терраформ мы настраивали в одном из прошлых занятий.  
Для выполнения ДЗ создадим манифесты.  
В манифесте **nat-instance.auto.tfvars** описали некоторые переменные

<details>

  <summary><b>nat-instance.auto.tfvars</b></summary>
  
```yml
folder_id    = "b1gcj17iv37qg7h91dfe"
vm_user      = "sam"
vm_user_nat  = "sam"
ssh_key_path = "/home/sam/.ssh/id_rsa.pub"

```
</details>



В манифесте **main.tf** опишем всё остальное по заданию.  
Мы описали создание подсетей, роутов, нат-инстансов, задали необходимые IP и привязали роут к нужной подсети

<details>

  <summary><b>main.tf</b></summary>
  
```yml
# Объявление переменных для пользовательских параметров

variable "folder_id" {
  type = string
}

variable "vm_user" {
  type = string
}

variable "vm_user_nat" {
  type = string
}

variable "ssh_key_path" {
  type = string
}

# Добавление прочих переменных

locals {
  network_name     = "netology"
  subnet_name1     = "public"
  subnet_name2     = "private"
  vm_test_name     = "test-vm"
  vm_nat_name      = "nat-instance"
  route_table_name = "nat-instance-route"
}


# Создание облачной сети

resource "yandex_vpc_network" "netology" {
  name = local.network_name
}

# Создание подсетей

resource "yandex_vpc_subnet" "public" {
  name           = local.subnet_name1
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.netology.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_vpc_subnet" "private" {
  name           = local.subnet_name2
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.netology.id
  v4_cidr_blocks = ["192.168.20.0/24"]
  route_table_id = yandex_vpc_route_table.nat-instance-route.id  # Привязываем наш роут к подсети
}



# Добавление готового образа ВМ

resource "yandex_compute_image" "ubuntu-1804-lts" {
  source_family = "ubuntu-1804-lts"
}

resource "yandex_compute_image" "nat-instance-ubuntu" {
  source_family = "nat-instance-ubuntu"
}

# Создание ВМ

resource "yandex_compute_instance" "test-vm" {
  name        = local.vm_test_name
  platform_id = "standard-v3"
  zone        = "ru-central1-a"

  resources {
    core_fraction = 20
    cores         = 2
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = yandex_compute_image.ubuntu-1804-lts.id
    }
  }

  network_interface {
    subnet_id          = yandex_vpc_subnet.private.id
  }

  metadata = {
    user-data = "#cloud-config\nusers:\n  - name: ${var.vm_user}\n    groups: sudo\n    shell: /bin/bash\n    sudo: ['ALL=(ALL) NOPASSWD:ALL']\n    ssh-authorized-keys:\n      - ${file("${var.ssh_key_path}")}"
  }
}

# Создание ВМ NAT

resource "yandex_compute_instance" "nat-instance" {
  name        = local.vm_nat_name
  platform_id = "standard-v3"
  zone        = "ru-central1-a"

  resources {
    core_fraction = 20
    cores         = 2
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd80mrhj8fl2oe87o4e1"
    }
  }

  network_interface {
    subnet_id          = yandex_vpc_subnet.public.id
    nat                = true
    ip_address         = "192.168.10.254"
  }

  metadata = {
    user-data = "#cloud-config\nusers:\n  - name: ${var.vm_user_nat}\n    groups: sudo\n    shell: /bin/bash\n    sudo: ['ALL=(ALL) NOPASSWD:ALL']\n    ssh-authorized-keys:\n      - ${file("${var.ssh_key_path}")}"
  }
}

# Создание таблицы маршрутизации и статического маршрута

resource "yandex_vpc_route_table" "nat-instance-route" {
  name       = "nat-instance-route"
  network_id = yandex_vpc_network.netology.id
  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = yandex_compute_instance.nat-instance.network_interface.0.ip_address
  }
}


```
</details>

Так же создадим аутпуты, что бы понимать, что именно у нас создалось  
<details>

  <summary><b>output.tf</b></summary>
  
```yml
output "internal_ip_address_nat-instance_yandex_cloud" {
  value = "${yandex_compute_instance.nat-instance.network_interface.0.ip_address}"
}

output "external_ip_address_nat-instance_yandex_cloud" {
  value = "${yandex_compute_instance.nat-instance.network_interface.0.nat_ip_address}"
}

output "internal_ip_address_test-vm_yandex_cloud" {
  value = "${yandex_compute_instance.test-vm.network_interface.0.ip_address}"
}

output "external_ip_address_test-vm_yandex_cloud" {
  value = "${yandex_compute_instance.test-vm.network_interface.0.nat_ip_address}"
}

```
</details>


```
terraform apply -y
```
```
Outputs:

external_ip_address_nat-instance_yandex_cloud = "130.193.37.55"
external_ip_address_test-vm_yandex_cloud = ""
internal_ip_address_nat-instance_yandex_cloud = "192.168.10.254"
internal_ip_address_test-vm_yandex_cloud = "192.168.20.6"
```
Скопируем наш закрытый ключ на NAT машину. Это необходимо для того, что бы подключаться с неё ко второй машине
```
scp /home/sam/.ssh/id_rsa sam@130.193.37.55:~/.ssh/
```
Подключимся к ней
```
ssh sam@130.193.37.55
```
Пинганём яндекс
```
ping ya.ru
```
```
ING ya.ru (5.255.255.242) 56(84) bytes of data.
64 bytes from ya.ru (5.255.255.242): icmp_seq=1 ttl=58 time=0.411 ms
64 bytes from ya.ru (5.255.255.242): icmp_seq=2 ttl=58 time=1.85 ms
64 bytes from ya.ru (5.255.255.242): icmp_seq=3 ttl=58 time=0.436 ms
64 bytes from ya.ru (5.255.255.242): icmp_seq=4 ttl=58 time=0.284 ms
```
Машина в интернете.  
C NAT инстанса подключимся к нашей машине в приватной подсети и посмотрим, выходит ли она в интернет
```
ssh sam@192.168.20.6
```
```
ping ya.ru
```
```
PING ya.ru (5.255.255.242) 56(84) bytes of data.
64 bytes from ya.ru (5.255.255.242): icmp_seq=1 ttl=56 time=0.808 ms
64 bytes from ya.ru (5.255.255.242): icmp_seq=2 ttl=56 time=0.516 ms
64 bytes from ya.ru (5.255.255.242): icmp_seq=3 ttl=56 time=0.522 ms
64 bytes from ya.ru (5.255.255.242): icmp_seq=4 ttl=56 time=0.496 ms
```
Машина в интернете.  
Мы можем проверить шлюз, через который она ходит в интернет с помощью команды
```
curl ifconfig.co
```
```
130.193.37.55
```
Это ip адресс нашего ната

