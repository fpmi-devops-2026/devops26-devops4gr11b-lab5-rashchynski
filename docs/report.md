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

Изучены статьи How To Set Up Ansible Inventories, How To Manage Multiple Servers
with Ansible Ad Hoc Commands, How To Write Ansible Playbooks, а также документация
Using Ansible command line tools и Using Ansible playbooks.

### 4.1. Целевые хосты

В варианте 2 (kind) виртуальных машин нет: ноды Kubernetes — это Docker-контейнеры.
Целевым кластером выбран кластер `lab4` из предыдущей лабораторной работы
(1 control-plane + 2 worker). Ansible подключается к контейнерам через
connection-плагин `community.docker.docker`:

```yaml
all:
  vars:
    ansible_connection: community.docker.docker
    ansible_python_interpreter: /usr/bin/python3
  children:
    control_plane:
      hosts:
        lab4-control-plane:
    workers:
      hosts:
        lab4-worker:
        lab4-worker2:
    k8s_cluster:
      children:
        control_plane:
        workers:
```

Образ `kindest/node` не содержит Python, поэтому на ноды предварительно
установлен `python3` модулем `raw`:

```bash
$ ansible all -m raw -a "apt-get update -qq && apt-get install -y -qq python3"
```

### 4.2. Проверка инвентаря

```bash
$ ansible-inventory --graph
@all:
  |--@k8s_cluster:
  |  |--@control_plane:
  |  |  |--lab4-control-plane
  |  |--@workers:
  |  |  |--lab4-worker
  |  |  |--lab4-worker2
```

### 4.3. Ad-hoc команды

Ping группы узлов одной командой:

```bash
$ ansible k8s_cluster -m ping
lab4-worker | SUCCESS => {"changed": false, "ping": "pong"}
lab4-control-plane | SUCCESS => {"changed": false, "ping": "pong"}
lab4-worker2 | SUCCESS => {"changed": false, "ping": "pong"}
```

Установка пакета `zsh`:

```bash
$ ansible workers -m apt -a "name=zsh state=present update_cache=yes"
lab4-worker  | changed=true  ... Setting up zsh (5.9-8+b24) ...
lab4-worker2 | changed=true  ... Setting up zsh (5.9-8+b24) ...

$ ansible workers -m shell -a "zsh --version"
lab4-worker  | zsh 5.9 (aarch64-unknown-linux-gnu)
lab4-worker2 | zsh 5.9 (aarch64-unknown-linux-gnu)
```

Перезапуск службы:

```bash
$ ansible workers -m service -a "name=nginx state=restarted"
```

### 4.4. Playbook'и

`playbooks/ping.yml` — ping и вывод фактов о хостах:

```yaml
- name: Ping all nodes of the Kubernetes cluster
  hosts: k8s_cluster
  gather_facts: true
  tasks:
    - name: Ping host
      ansible.builtin.ping:
    - name: Show host information
      ansible.builtin.debug:
        msg: >-
          {{ inventory_hostname }}:
          {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}
          ({{ ansible_facts['architecture'] }}),
          CPU {{ ansible_facts['processor_vcpus'] }},
          RAM {{ ansible_facts['memtotal_mb'] }} MB
```

Результат:

```text
ok: [lab4-control-plane] => "lab4-control-plane: Debian 13.6 (aarch64), CPU 10, RAM 7934 MB"
ok: [lab4-worker]        => "lab4-worker: Debian 13.6 (aarch64), CPU 10, RAM 7934 MB"
ok: [lab4-worker2]       => "lab4-worker2: Debian 13.6 (aarch64), CPU 10, RAM 7934 MB"
```

`playbooks/system-info.yml` — пример из документации (память и CPU через
`ansible_facts`).

`playbooks/nginx.yml` — playbook с ролью `nginx`. Роль состоит из задач,
обработчика и шаблона:

```yaml
# roles/nginx/tasks/main.yml
- name: Ensure nginx is installed
  ansible.builtin.apt:
    name: "{{ nginx_package }}"
    state: present
    update_cache: true

- name: Deploy the index page from a template
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ nginx_document_root }}/index.html"
    mode: "0644"
  notify: Restart nginx

- name: Ensure nginx is running and enabled
  ansible.builtin.service:
    name: "{{ nginx_package }}"
    state: started
    enabled: true
```

Запуск:

```bash
$ ansible-playbook playbooks/nginx.yml
TASK [nginx : Deploy the index page from a template] ***
changed: [lab4-worker]
changed: [lab4-worker2]
RUNNING HANDLER [nginx : Restart nginx] ***
changed: [lab4-worker]
changed: [lab4-worker2]
PLAY RECAP
lab4-worker  : ok=6 changed=2 failed=0
lab4-worker2 : ok=6 changed=2 failed=0
```

Повторный запуск показывает идемпотентность — `changed=0`.

Проверка страницы на нодах:

```bash
$ ansible workers -m uri -a "url=http://localhost/ return_content=yes"
lab4-worker | SUCCESS => "content": "...<h1>Ansible + kind</h1>...<p>Хост: lab4-worker</p>..."
```

Логи выполнения: [`task2/ansible/logs/`](../task2/ansible/logs/).

### 4.5. Вывод по заданию 2

Освоены inventory и группы хостов, ad-hoc команды (ping, установка пакета,
перезапуск службы), написан и выполнен playbook с ролью, обработчиком и
шаблоном. Конфигурации опубликованы в каталоге [`task2/`](../task2/).

---

## 5. Задание 3. Создание кластера с помощью Terraform и Ansible локально

### 5.1. Вариант и инструменты

Выбран **вариант 2** — kind (Kubernetes in Docker). Хост работает на Apple
Silicon, поэтому вариант 1 (VirtualBox, Libvirt/KVM) недоступен. Обязательное
условие — Terraform, Ansible, Kubernetes и Docker — выполнено.

| Инструмент | Версия | Роль |
|------------|--------|------|
| Terraform | 1.16.5 | создание кластера kind |
| kind | v0.33.0 | ноды кластера как Docker-контейнеры |
| Ansible | 14.5 | развёртывание приложения |
| Kubernetes | v1.37.0 | оркестрация |
| Docker | 29.8.0 | контейнеры нод и образы приложений |

### 5.2. Создание кластера Terraform

Terraform формирует конфигурацию кластера и вызывает kind CLI через
`terraform_data` (провайдер `tehcyx/kind` в текущей версии падал на
`kubeadm init`, поэтому используется официальный CLI):

```hcl
locals {
  kind_config = {
    kind       = "Cluster"
    apiVersion = "kind.x-k8s.io/v1alpha4"
    nodes = [
      {
        role = "control-plane"
        extraPortMappings = [
          { containerPort = 30080, hostPort = 30080, protocol = "TCP" }
        ]
      },
      { role = "worker" },
      { role = "worker" },
    ]
  }
}

resource "local_file" "kind_config" {
  content  = yamlencode(local.kind_config)
  filename = "${path.module}/kind-config.yaml"
}

resource "terraform_data" "cluster" {
  triggers_replace = var.cluster_name

  provisioner "local-exec" {
    command = <<-EOT
      kind create cluster --name "${self.triggers_replace}" \
        --config "${local_file.kind_config.filename}"
      kind export kubeconfig --name "${self.triggers_replace}" \
        --kubeconfig "${path.module}/kubeconfig"
    EOT
  }

  provisioner "local-exec" {
    when    = destroy
    command = "kind delete cluster --name ${self.triggers_replace}"
  }
}
```

```bash
$ terraform apply -auto-approve
terraform_data.cluster: Creation complete after 16s
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

$ kubectl --kubeconfig kubeconfig get nodes
NAME                 STATUS   ROLES           AGE   VERSION
lab5-control-plane   Ready    control-plane   25s   v1.37.0
lab5-worker          Ready    <none>          15s   v1.37.0
lab5-worker2         Ready    <none>          16s   v1.37.0
```

### 5.3. Приложение и образы

Разворачивается проект из задания 5 лабораторной работы №1: FastAPI +
PostgreSQL + миграции Liquibase. Перед приложением стоит nginx (задание 4
лабораторной №1). Исходники — в [`task3/app/`](../task3/app/).

| Образ | Назначение |
|-------|------------|
| `nrashchynski/lab5-students-app:v1` | FastAPI-приложение |
| `nrashchynski/lab5-liquibase:v1` | Liquibase + JDBC-драйвер PostgreSQL |

Драйвер PostgreSQL добавлен в образ Liquibase при сборке: переменная
`INSTALL_POSTGRESQL_DRIVER` в актуальной версии образа драйвер не устанавливает,
а JAR дополнительно получает права `644` (контейнер работает под пользователем
`liquibase`).

### 5.4. Развёртывание через Ansible

Playbook [`task3/ansible/playbooks/deploy-app.yml`](../task3/ansible/playbooks/deploy-app.yml)
помечает ноды и применяет манифесты:

```yaml
- name: Label the worker node for the web server and application
  ansible.builtin.command:
    cmd: "kubectl --kubeconfig {{ kubeconfig }} label node lab5-worker node-role=web --overwrite"

- name: Label the worker node for the database
  ansible.builtin.command:
    cmd: "kubectl --kubeconfig {{ kubeconfig }} label node lab5-worker2 node-role=db --overwrite"

# ... применение 00-namespace ... 07-nginx и ожидание готовности
```

```bash
$ ansible-playbook playbooks/deploy-app.yml
PLAY RECAP
localhost : ok=15 changed=13 failed=0
```

### 5.5. Размещение сервисов по нодам

| Компонент | Нода | Как задано |
|-----------|------|------------|
| nginx | `lab5-worker` | `nodeSelector: node-role=web` |
| app (FastAPI) | `lab5-worker` | `nodeSelector: node-role=web` |
| db (PostgreSQL) | `lab5-worker2` | `nodeSelector: node-role=db` |
| liquibase (Job) | `lab5-worker` | планировщик |

```text
NAME                     READY   STATUS      NODE
app-7786797db4-8rbhz     1/1     Running     lab5-worker
db-69f6448f99-wbchw      1/1     Running     lab5-worker2
liquibase-ngdfd          0/1     Completed   lab5-worker
nginx-789dc46fc6-7sl98   1/1     Running     lab5-worker
```

Веб-сервер и приложение размещены на одной ноде, база данных — на другой,
как требует задание.

### 5.6. Манифесты

- `00-namespace.yaml` — namespace `lab5`;
- `01-secret.yaml` — параметры PostgreSQL и `DATABASE_URL`
  (`postgresql+psycopg2://...`, явный драйвер обязателен для SQLAlchemy 2.1+);
- `02-configmap-liquibase.yaml` — changelog Liquibase;
- `03-postgres.yaml` — PVC 1Gi, Deployment и Service `db`;
- `04-liquibase-job.yaml` — Job миграций с init-контейнером ожидания БД;
- `05-app.yaml` — Deployment и Service `app`;
- `06-nginx-configmap.yaml` — конфигурация reverse proxy;
- `07-nginx.yaml` — Deployment и Service `nginx` типа NodePort (30080).

### 5.7. Проверка

```bash
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:30080/docs
200

$ curl -s -X POST http://localhost:30080/students/enroll \
    -H "Content-Type: application/json" \
    -d '{"full_name":"Петров Пётр","specialty":"ИИ","status":"active"}'
{"full_name":"Петров Пётр","specialty":"ИИ","id":2,"status":"enrolled"}

$ curl -s http://localhost:30080/students
[{"full_name":"Иванов Иван","specialty":"ПМ","id":1,"status":"enrolled"},
 {"full_name":"Петров Пётр","specialty":"ИИ","id":2,"status":"enrolled"}]

$ curl -s http://localhost:30080/reports/summary
{"active_students_by_specialty":{"ИИ":1,"ПМ":1},"total_active_students":2,"total_expelled_students":0}
```

Логи: [`task3/logs/`](../task3/logs/).

### 5.8. Вывод по заданию 3

Кластер Kubernetes из трёх нод создан через Terraform (kind), приложение из
лабораторной №1 развёрнуто через Ansible, сервисы разнесены по нодам согласно
заданию, доступ проверен через NodePort.

---

## 6. Контрольные вопросы

### 1. Для каких задач применяется Ansible?

Ansible — система управления конфигурациями и автоматизации. Применяется для
настройки серверов, установки и обновления ПО, развёртывания приложений,
управления конфигурационными файлами, оркестрации многошаговых операций и
подготовки инфраструктуры. Работает без агентов: подключается по SSH (или через
другие connection-плагины) и выполняет задачи идемпотентно.

### 2. Какие способы управления серверами применяются в Ansible?

- **Ad-hoc команды** — разовые команды для быстрых операций (`ansible ... -m ...`).
- **Playbook'и** — декларативные YAML-сценарии из задач, выполняемые
  `ansible-playbook`.
- **Роли** — переиспользуемые наборы задач, шаблонов, файлов и обработчиков.
- **Inventory** — список хостов и групп, к которым применяются команды.
- **Модули и плагины подключения** — SSH, local, Docker и другие способы доступа.

### 3. Что входит в состав файла Ansible playbook?

Playbook — YAML-файл со списком пьес (plays). Каждая пьеса содержит:

- `hosts` — группу или хосты, к которым применяется пьеса;
- `become` — повышение привилегий;
- `vars` — переменные;
- `gather_facts` — сбор фактов;
- `roles` — список ролей;
- `tasks` — список задач с модулями;
- `handlers` — обработчики, вызываемые через `notify`.

### 4. Что такое группы узлов (хостов) в Ansible?

Группы — именованные объединения хостов в inventory (например, `workers`,
`control_plane`). Позволяют применять команды и playbook'и сразу к набору
хостов и использовать групповые переменные. Группы могут быть вложенными.

### 5. Как одной командой выполнить ping группы узлов (хостов)?

```bash
ansible <группа> -m ping
```

Например, `ansible k8s_cluster -m ping` проверяет доступность всех хостов группы
`k8s_cluster`.

### 6. Что такое роль Ansible?

Роль — структурированный переиспользуемый набор задач, переменных, шаблонов,
файлов и обработчиков с фиксированной структурой каталогов (`tasks/`, `handlers/`,
`templates/`, `files/`, `defaults/`, `vars/`, `meta/`). Роль подключается в
playbook через `roles:` и позволяет переиспользовать логику между проектами.

### 7. Что такое файл инвентаризации Ansible и как он используется?

Inventory — файл (INI или YAML) со списком управляемых хостов, их групп и
переменных (адрес, пользователь, способ подключения). Ansible читает его, чтобы
определить, к каким хостам применять команды и playbook'и. Путь задаётся в
`ansible.cfg` или флагом `-i`.

### 8. Что такое специальная команда Ansible?

Специальная (ad-hoc) команда — однократная команда Ansible без playbook'а:

```bash
ansible <группа> -m <модуль> -a "<аргументы>"
```

Используется для быстрых проверок и разовых операций, например
`ansible all -m ping` или `ansible workers -m apt -a "name=zsh state=present"`.

### 9. Как перезапустить службу с помощью Ansible?

Модулем `service` (или `systemd_service`):

```bash
ansible workers -m service -a "name=nginx state=restarted"
```

В playbook:

```yaml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

### 10. Что такое Hashicorp Terraform?

Terraform — инструмент Infrastructure as Code от HashiCorp. Позволяет описывать
инфраструктуру декларативно в HCL-файлах, строить план изменений (`plan`) и
применять его (`apply`), храня состояние ресурсов в state-файле. Поддерживает
множество провайдеров (облака, гипервизоры, Docker, Kubernetes и др.).

### 11. Что такое инфраструктура как код (Infrastructure as code — IaC)?

IaC — подход, при котором инфраструктура описывается в виде кода и хранится в
системе контроля версий. Это делает развёртывание воспроизводимым, проверяемым
и автоматизируемым: один и тот же код создаёт одинаковую инфраструктуру в разных
окружениях, а изменения проходят ревью и версионирование.

### 12. Как установить Terraform?

Скачать бинарник с официального сайта HashiCorp или установить пакетным
менеджером. Например, на macOS:

```bash
brew install hashicorp/tap/terraform
```

Проверка:

```bash
terraform version
```

В данной работе CDN HashiCorp был недоступен, поэтому бинарник установлен с
зеркала `hashicorp-releases.yandexcloud.net`.

### 13. Что такое провайдер (поставщик) в Terraform?

Провайдер — плагин, который реализует взаимодействие Terraform с конкретной
платформой (AWS, Yandex Cloud, Docker, Kubernetes, kind и т. д.). Он описывает
ресурсы и источники данных этой платформы. Провайдеры объявляются в блоке
`required_providers` и загружаются командой `terraform init`.

### 14. Объясните базовую структуру файла конфигурации Terraform.

Базовая структура:

```hcl
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
    }
  }
}

provider "docker" {}

resource "docker_container" "nginx" {
  image = "nginx:1.28"
  name  = "tutorial"
}
```

- блок `terraform` — версия Terraform и провайдеры;
- блок `provider` — настройки провайдера;
- блоки `resource` — описываемые ресурсы;
- блоки `variable`, `output`, `locals` — переменные и выходные значения.

### 15. Что такое файл состояния Terraform?

State-файл (`terraform.tfstate`) — хранилище соответствия между ресурсами в
конфигурации и реальными объектами инфраструктуры, а также их атрибутов.
Terraform использует state, чтобы понимать, что уже создано, и вычислять план
изменений. State может храниться локально или удалённо (backend).

### 16. Для чего нужна terraform destroy команда?

`terraform destroy` удаляет все ресурсы, описанные в конфигурации и
зафиксированные в state. Она показывает план удаления и после подтверждения
уничтожает инфраструктуру. Используется для очистки временных стендов и
экономии ресурсов.

### 17. Что такое состояние в terraform?

Состояние (state) — снимок созданной инфраструктуры: список ресурсов, их
идентификаторы и атрибуты. Оно позволяет Terraform сопоставлять конфигурацию с
реальными объектами, обнаруживать расхождения и выполнять инкрементальные
изменения. Состояние может содержать чувствительные данные, поэтому его
защищают и хранят удалённо при командной работе.

### 18. Как определить зависимости в Terraform?

Зависимости бывают явными и неявными:

- **Неявные** — через ссылки на атрибуты других ресурсов, например
  `image = docker_image.nginx.image_id`;
- **Явные** — метааргумент `depends_on`:

```hcl
resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "tutorial"
  depends_on = [docker_image.nginx]
}
```

Terraform строит граф зависимостей и выполняет операции в правильном порядке.
