# Лабораторная работа №5

**Тема:** Проектирование модели управления версиями ПО, конфигурациями и
непрерывной поставкой
**Дисциплина:** Непрерывное интегрирование и сборка программного обеспечения
**Автор:** Ращинский Назар Андреевич, группа 11
**Год:** 2026

---

## 1. Цель работы

Изучение решений для автоматизации управления инфраструктурой и установки
приложений: Terraform, Ansible и Kubernetes.

## 2. Вариант задания

Группа **11**: согласно критериям оценивания для групп 10–12 выполняются
**все три задания** лабораторной работы.

Задание 3 выполняется по **варианту 2** (kind, Docker-in-Docker): хостовая
машина работает на Apple Silicon, поэтому локальные гипервизоры варианта 1
(VirtualBox, Libvirt/KVM) недоступны. Обязательное условие варианта 2 —
использование Terraform, Ansible, Kubernetes и Docker — выполняется.

---

## 3. Задание 1. Основы управления конфигурацией инфраструктуры с Terraform

Изучено официальное руководство [Terraform Tutorial — Get Started: Docker](https://developer.hashicorp.com/terraform/tutorials/docker-get-started)
и выполнены все 7 руководств. Для каждого руководства в каталоге
[`task1/`](../task1/) лежит отдельная папка с конфигурацией и выводом команд.

### 3.1. Окружение

| Инструмент | Версия | Назначение |
|------------|--------|------------|
| Terraform | v1.16.5 (darwin_arm64) | управление инфраструктурой |
| Docker Desktop | 29.8.0 (aarch64) | запуск контейнеров (провайдер `kreuzwerker/docker`) |
| macOS | Apple Silicon | хостовая система |

Две особенности окружения:

1. `registry.terraform.io` и `releases.hashicorp.com` недоступны из сети
   выполнения работы, поэтому Terraform установлен с зеркала
   `hashicorp-releases.yandexcloud.net`, а провайдеры загружаются через
   network mirror `terraform-mirror.yandexcloud.net` (настраивается в
   `~/.terraformrc`).
2. Docker-провайдер подключается к сокету Docker Desktop:
   `export DOCKER_HOST=unix://$HOME/.docker/run/docker.sock`.

Чтобы не конфликтовать с уже существующими локальными контейнерами на
`nginx:latest`, в конфигурациях зафиксирован образ `nginx:1.28`.

### 3.2. Руководство 1. What is Infrastructure as Code with Terraform?

Вводное интерактивное руководство. Конфигурационных файлов не содержит:
объясняет принципы Infrastructure as Code (инфраструктура описывается
декларативно, хранится в системе контроля версий, изменения проходят через
`plan` и `apply`, состояние отслеживается в state-файле).

### 3.3. Руководство 2. Install Terraform

Установка и проверка версии:

```bash
$ terraform version
Terraform v1.16.5
on darwin_arm64
```

### 3.4. Руководство 3. Build infrastructure

Конфигурация [`task1/tutorial-3-build/main.tf`](../task1/tutorial-3-build/main.tf):

```hcl
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
    }
  }
}

provider "docker" {}

resource "docker_image" "nginx" {
  name         = "nginx:1.28"
  keep_locally = false
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "tutorial"

  ports {
    internal = 80
    external = 8000
  }
}
```

Команды и результат:

```bash
$ terraform init
Terraform has been successfully initialized!

$ terraform plan
Plan: 2 to add, 0 to change, 0 to destroy.

$ terraform apply -auto-approve
docker_image.nginx: Creation complete after 0s [id=sha256:146adea4768b...nginx:1.28]
docker_container.nginx: Creation complete after 0s [id=58272821e4e2...]
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

$ terraform state list
docker_container.nginx
docker_image.nginx

$ docker ps --filter name=tutorial
tutorial   146adea4768b   0.0.0.0:8000->80/tcp

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000
200
```

Terraform создал образ и контейнер, состояние сохранено в `terraform.tfstate`.

### 3.5. Руководство 4. Change infrastructure

Внешний порт контейнера изменён с 8000 на 8001:

```bash
$ terraform apply -auto-approve
  # docker_container.nginx must be replaced
      ~ external = 8000 -> 8001 # forces replacement
Plan: 1 to add, 0 to change, 1 to destroy.
Apply complete! Resources: 1 added, 0 changed, 1 destroyed.

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8001
200
```

Terraform показал, что изменение порта требует пересоздания контейнера, и
выполнил его.

### 3.6. Руководство 5. Destroy infrastructure

```bash
$ terraform destroy -auto-approve
  # docker_container.nginx will be destroyed
  # docker_image.nginx will be destroyed
Plan: 0 to add, 0 to change, 2 to destroy.
docker_container.nginx: Destruction complete after 0s
docker_image.nginx: Destruction complete after 0s
Destroy complete! Resources: 2 destroyed.
```

### 3.7. Руководство 6. Define input variables

Имя контейнера вынесено в переменную `container_name`:

```hcl
variable "container_name" {
  description = "Value of the name for the Docker container"
  type        = string
  default     = "ExampleNginxContainer"
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = var.container_name
  # ...
}
```

Значение по умолчанию:

```bash
$ terraform apply -auto-approve
Plan: 2 to add, 0 to change, 0 to destroy.
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
ExampleNginxContainer   0.0.0.0:8000->80/tcp
```

Переопределение через флаг `-var`:

```bash
$ terraform apply -auto-approve -var='container_name=CustomNginxContainer'
Plan: 1 to add, 0 to change, 1 to destroy.
Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
CustomNginxContainer   0.0.0.0:8000->80/tcp
```

Также значение можно задать через `terraform.tfvars` (файл в каталоге
руководства) или переменную окружения `TF_VAR_container_name`.

### 3.8. Руководство 7. Query data with outputs

Добавлены выходные значения:

```hcl
output "container_name" {
  value       = docker_container.nginx.name
  description = "The name of the Docker container"
}

output "container_id" {
  value       = docker_container.nginx.id
  description = "The ID of the Docker container"
}
```

```bash
$ terraform output
container_id   = "a830138849c5f88afd63f2ab02054836d8a19b848ca29aaef4a61b187916262b"
container_name = "ExampleNginxContainer"

$ terraform output -raw container_name
ExampleNginxContainer

$ terraform output -raw container_id
a830138849c5f88afd63f2ab02054836d8a19b848ca29aaef4a61b187916262b
```

### 3.9. Вывод по заданию 1

Выполнены все 7 руководств Terraform Get Started — Docker: установка Terraform,
создание инфраструктуры через Docker-провайдер, изменение ресурсов, уничтожение
инфраструктуры, работа с входными переменными и выходными значениями. Код и
вывод команд опубликованы в каталоге [`task1/`](../task1/).

---

## 4. Задание 2. Управление конфигурацией серверов с помощью Ansible

_(раздел будет добавлен после выполнения задания 2)_

---

## 5. Задание 3. Создание кластера с помощью Terraform и Ansible локально

_(раздел будет добавлен после выполнения задания 3)_

---

## 6. Контрольные вопросы

_(ответы будут добавлены в конце работы)_
