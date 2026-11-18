<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.7 CI/CD: от коммита до продакшена](07-ci-cd-from-commit-to-production.md) · [📋 Этап 21](README.md) · [21.9 Метрики с Prometheus и Grafana →](09-metrics-with-prometheus-and-grafana.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) <!-- nav -->

# Урок 21.8 — Инфраструктура как код с OpenTofu

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 8 из 14 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое инфраструктура как код (IaC) и почему команда описывает серверы и кластеры в файлах, а не набирает команды.
- Как работает OpenTofu: провайдеры, ресурсы, переменные, выходные значения и цикл `init` → `plan` → `apply` → `destroy`.
- Что такое файл состояния, зачем он нужен OpenTofu и почему его нужно защищать как пароль.
- Как OpenTofu находит *дрейф* (изменения, сделанные вручную) и возвращает всё на место.
- Как проверять конфигурацию командами `tofu fmt`, `tofu validate` и автоматическими тестами `tofu test`.

## Перед началом

- Вы прошли [Урок 21.7 — CI/CD: от коммита до продакшена](07-ci-cd-from-commit-to-production.md).
- Вы умеете создавать кластер kind и пользоваться `kubectl` ([Урок 21.3](03-a-local-cluster-with-kind.md)) и знаете Deployment, Service, пространства имён и ConfigMap ([Уроки 21.2](02-kubernetes-concepts.md) и [21.5](05-configmaps-secrets-and-probes.md)).
- Docker Desktop запущен. Код этого урока лежит в `~/projects/rust/code/p21-infra`.
- Всё в этом уроке работает с локальным кластером kind. Облачный аккаунт не нужен, и ничего не стоит денег.

## Новые слова

| Слово | Значение |
|---|---|
| **IaC** (Infrastructure as Code, инфраструктура как код) | Описание инфраструктуры (кластеров, сетей, баз данных, объектов внутри кластера) в текстовых файлах, которые инструмент превращает в настоящие вещи. Файлы лежат в Git, как обычный код. |
| **декларативный** (declarative) | Вы описываете *результат*, который хотите получить («3 копии веб-сервера»), а не шаги к нему. Шаги инструмент находит сам. |
| **OpenTofu** | Инструмент IaC с открытым исходным кодом, которым управляет Linux Foundation. Его команда — `tofu`. Он появился как форк (копия, которая развивается самостоятельно) Terraform и читает такие же файлы. |
| **HCL** (HashiCorp Configuration Language, язык конфигурации HashiCorp) | Язык файлов OpenTofu (`.tf`): блоки с именами и строки `key = value`. |
| **провайдер** (provider) | Плагин, который учит OpenTofu работать с одной системой: Kubernetes, Docker, облаком. Он превращает ресурсы в вызовы API (Application Programming Interface, программный интерфейс). |
| **ресурс** (resource) | Одна вещь, которую OpenTofu создаёт и которой управляет, например одно пространство имён или один Deployment. |
| **состояние** (state) | Файл (`terraform.tfstate`), в котором OpenTofu помнит, какие настоящие объекты он создал для каких ресурсов. |
| **план** (plan) | Предпросмотр: список изменений, которые OpenTofu *сделал бы*, чтобы привести реальность к описанию. |
| **применение** (apply) | Выполнение изменений из плана по-настоящему. |
| **дрейф** (drift) | Расхождение между описанием и реальностью, обычно потому, что кто-то что-то изменил вручную. |
| **переменная / выходное значение** (variable / output) | Входное значение конфигурации / значение, которое конфигурация печатает и возвращает после `apply`. |
| **YAML** (YAML Ain't Markup Language, «YAML — не язык разметки») / **JSON** (JavaScript Object Notation, нотация объектов JavaScript) | Два текстовых формата для структурированных данных. Манифесты Kubernetes пишутся на YAML, состояние OpenTofu хранится в JSON. |
| **DNS** (Domain Name System, система доменных имён) | Система, которая превращает имена вроде `example.com` в сетевые адреса. |
| **S3** (Simple Storage Service, простой сервис хранения) | Популярный протокол для хранения файлов в «бакетах» (buckets) по сети; его понимают многие системы хранения. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически запускает проверки (а здесь и `tofu plan`) для каждого изменения (Урок 21.7). |
| **lock-файл** (lock file) | `.terraform.lock.hcl`: точные версии провайдеров (и их контрольные суммы), которые выбрал `tofu init`. Коммитьте его в Git. |

## 1. Зачем описывать инфраструктуру в файлах

До сих пор вы создавали объекты в кластере двумя способами: командами вроде `kubectl create namespace` и манифестами YAML (YAML Ain't Markup Language) с `kubectl apply`. Helm и Kustomize ([Урок 21.6](06-helm-and-kustomize.md)) упаковали эти манифесты. Это уже «инфраструктура в файлах» для того, что *внутри* одного кластера.

Настоящей системе нужно больше: сам кластер, сеть вокруг него, управляемая база данных, имена DNS (Domain Name System), бакеты хранилища, права доступа. Собрать всё это щелчками в веб-консоли можно один раз. Потом начинаются вопросы: *Кто изменил эту настройку? Можно ли построить то же самое ещё раз для тестового окружения? Что именно произойдёт, если я это поменяю?*

**IaC** (Infrastructure as Code) отвечает на эти вопросы так, как вы уже привыкли с кодом:

| Проблема инфраструктуры, собранной вручную | Что даёт IaC |
|---|---|
| Никто не знает, как всё устроено сейчас | Файлы *и есть* описание, и они лежат в Git |
| Изменения не видны | Каждое изменение — коммит и pull request, который кто-то проверяет |
| Второе окружение «почти такое же» | Те же файлы с другими переменными |
| Рискованные изменения | **План** показывает каждое изменение до того, как что-то произойдёт |
| Ручные изменения забываются | Инструмент находит **дрейф** и показывает его |

Инструменты IaC **декларативны**: вы пишете, *что* должно существовать, а инструмент сравнивает это с тем, *что* существует, и делает только разницу. Это та же идея, что у Deployment в Kubernetes: вы говорите «3 реплики», и контроллер поддерживает это. Разница в том, что контроллеры Kubernetes работают всё время, а OpenTofu работает только тогда, когда его запускаете вы (или конвейер CI).

```text
   your .tf files            state file               reality
  (what you want)      (what OpenTofu made)     (what exists now)
        │                      │                       │
        └──────────┬───────────┴───────────┬───────────┘
                   ▼                       ▼
              tofu plan  ── compare ──►  list of changes
                   │
                   ▼
              tofu apply ──► provider ──► API calls (Kubernetes, Docker, cloud…)
```

## 2. OpenTofu: установка и первая конфигурация

**OpenTofu** — инструмент IaC с открытым исходным кодом. Его файлы пишутся на **HCL** (HashiCorp Configuration Language). Сам OpenTofu ничего не знает о Kubernetes или облаках: настоящую работу делают **провайдеры** (плагины, которые скачиваются из реестра OpenTofu). Есть провайдеры для Kubernetes, Helm, Docker и всех больших облаков.

Установите его через Homebrew, как описано на официальной странице установки, и проверьте версию:

```bash
brew install opentofu
tofu version
```

```text
OpenTofu v1.13.9
on darwin_arm64
```

Создайте новый кластер для этого урока (вывод такой же, как в Уроке 21.3) и папку проекта:

```bash
kind create cluster --name course-21-08
mkdir -p ~/projects/rust/code/p21-infra
cd ~/projects/rust/code/p21-infra
```

OpenTofu читает **все** файлы `.tf` в текущей папке как одну конфигурацию. Имена файлов нужны только людям; обычно их делят так: `versions.tf`, `variables.tf`, `main.tf` и `outputs.tf`.

`versions.tf` говорит, какие версии инструмента и провайдера нужны и как провайдер подключается:

```hcl
terraform {
  required_version = ">= 1.13"

  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 3.5"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "kind-course-21-08"
}
```

**Что происходит:**

1. Блок называется `terraform` ради совместимости: OpenTofu читает файлы в стиле Terraform без изменений.
2. `required_providers` называет провайдер `hashicorp/kubernetes` из реестра. `~> 3.5` означает «3.5 или новее, но ниже 4.0»: новые исправления ошибок приветствуются, новая мажорная версия (которая может что-то сломать) — нет.
3. Блок `provider` настраивает его: использовать файл kubeconfig, который записал kind, и контекст *этого* кластера. Явно названный контекст защищает вас от изменений не в том кластере.

`variables.tf` объявляет входные значения:

```hcl
variable "namespace" {
  description = "The namespace for the web application."
  type        = string
  default     = "web"
}

variable "replicas" {
  description = "How many copies (pods) of the web server to run."
  type        = number
  default     = 2

  validation {
    condition     = var.replicas >= 1 && var.replicas <= 5
    error_message = "replicas must be between 1 and 5."
  }
}
```

У переменной есть тип, необязательное значение по умолчанию и необязательные правила **проверки** (validation). Внутри конфигурации её читают как `var.replicas`.

`main.tf` описывает три ресурса: пространство имён, ConfigMap с веб-страницей и Deployment официального образа веб-сервера `nginx`, который показывает эту страницу. Мы берём `nginx`, чтобы урок не зависел от вашего собственного образа; для вашего сервиса на Rust идеи точно такие же.

```hcl
resource "kubernetes_namespace_v1" "web" {
  metadata {
    name = var.namespace
  }
}

resource "kubernetes_config_map_v1" "page" {
  metadata {
    name      = "page"
    namespace = kubernetes_namespace_v1.web.metadata[0].name
  }

  data = {
    "index.html" = "<h1>Created by OpenTofu</h1>\n"
  }
}

resource "kubernetes_deployment_v1" "web" {
  metadata {
    name      = "web"
    namespace = kubernetes_namespace_v1.web.metadata[0].name
  }

  spec {
    replicas = var.replicas

    selector {
      match_labels = { app = "web" }
    }

    template {
      metadata {
        labels = { app = "web" }
      }

      spec {
        container {
          name  = "nginx"
          image = "nginx:1.30-alpine"

          port {
            container_port = 80
          }

          volume_mount {
            name       = "page"
            mount_path = "/usr/share/nginx/html"
          }
        }

        volume {
          name = "page"
          config_map {
            name = kubernetes_config_map_v1.page.metadata[0].name
          }
        }
      }
    }
  }
}
```

**Что происходит:**

1. У `resource "kubernetes_namespace_v1" "web"` две метки: **тип ресурса** (его определяет провайдер) и **локальное имя** (ваше). Вместе они образуют адрес `kubernetes_namespace_v1.web`.
2. Вложенные блоки повторяют знакомый вам YAML Kubernetes: `metadata`, `spec`, `template`, `container`. Отличается только написание (`match_labels` вместо `matchLabels`).
3. `kubernetes_namespace_v1.web.metadata[0].name` — **ссылка** на другой ресурс. OpenTofu видит её и понимает порядок: сначала пространство имён, потом ConfigMap, потом Deployment. Порядок вы никогда не пишете сами.
4. ConfigMap хранит файл `index.html`; Deployment монтирует его как том туда, где `nginx` ищет свою стартовую страницу.

`outputs.tf` говорит, что напечатать в конце:

```hcl
output "namespace" {
  value = kubernetes_namespace_v1.web.metadata[0].name
}

output "replicas" {
  value = kubernetes_deployment_v1.web.spec[0].replicas
}
```

## 3. Цикл: init, plan, apply

**`tofu init`** готовит папку: скачивает провайдеры и записывает lock-файл. Запустите его один раз и снова — каждый раз, когда добавляете провайдер или модуль.

```bash
tofu init
```

```text
Initializing provider plugins...
- Finding hashicorp/kubernetes versions matching "~> 3.5"...
- Installed hashicorp/kubernetes v3.5.0 (signed, key ID 0C0AF313E5FD9F80)
...
OpenTofu has created a lock file .terraform.lock.hcl to record the provider
...
OpenTofu has been successfully initialized!
```

Провайдер теперь лежит в скрытой папке `.terraform/`. Проверьте файлы, потом посмотрите план:

```bash
tofu fmt -check
tofu validate
tofu plan
```

Сокращённый вывод (настоящий план занимает около 100 строк, в основном со значениями по умолчанию):

```text
Success! The configuration is valid.
...
  # kubernetes_config_map_v1.page will be created
  + resource "kubernetes_config_map_v1" "page" {
      + data = {
          + "index.html" = <<-EOT
                <h1>Created by OpenTofu</h1>
            EOT
        }
      + id   = (known after apply)
  ...
  # kubernetes_deployment_v1.web will be created
          + replicas                  = "2"
  ...
  # kubernetes_namespace_v1.web will be created
  ...
Plan: 3 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + namespace = "web"
  + replicas  = "2"
```

**Что происходит:**

1. `tofu fmt -check` ничего не печатает, если файлы отформатированы стандартно (без `-check` он их исправляет, как `cargo fmt`). `tofu validate` проверяет синтаксис и ссылки, не обращаясь к кластеру.
2. `tofu plan` сравнивает файлы с состоянием (сейчас пустым) и с кластером. `+` означает «будет создан», `~` — «будет изменён на месте», `-` — «будет удалён», а `-/+` — «придётся удалить и создать заново».
3. `(known after apply)` отмечает значения, которые может дать только кластер, например уникальный идентификатор.
4. Итоговая строка — самая важная строка любого плана. Читайте её перед каждым `apply`.

Теперь сделаем всё по-настоящему. `tofu apply` ещё раз показывает тот же план и просит подтверждения; принимается только слово `yes`:

```bash
tofu apply
```

```text
...
Do you want to perform these actions?
  OpenTofu will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

kubernetes_namespace_v1.web: Creation complete after 0s [id=web]
kubernetes_config_map_v1.page: Creation complete after 0s [id=web/page]
kubernetes_deployment_v1.web: Still creating... [20s elapsed]
kubernetes_deployment_v1.web: Creation complete after 26s [id=web/web]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

namespace = "web"
replicas = "2"
```

(Мы убрали план в начале и строки «Creating...».)

Ресурсы создались в порядке зависимостей, а провайдер дождался, пока поды Deployment будут готовы. Проверьте знакомыми инструментами:

```bash
kubectl get deployments -n web
kubectl -n web port-forward deployment/web 8088:80
```

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    2/2     2            2           26s
```

Пока работает port-forward, `curl -s localhost:8088` во втором терминале печатает `<h1>Created by OpenTofu</h1>`. Остановите port-forward клавишами Ctrl+C.

> 📏 **Соглашение:** в команде никто не запускает `tofu apply` с ноутбука против продакшена. Конвейер CI (Continuous Integration) ([Урок 21.7](07-ci-cd-from-commit-to-production.md)) запускает `tofu plan` для каждого pull request и выкладывает план на проверку; после слияния конвейер запускает `tofu apply` ровно с этим сохранённым планом (`tofu plan -out=plan.bin`, затем `tofu apply plan.bin`).

## 4. Изменения и поиск дрейфа

Измените значение декларативно: скажите, чего вы хотите. Переменную можно задать в командной строке:

```bash
tofu apply -var replicas=3
```

```text
  # kubernetes_deployment_v1.web will be updated in-place
  ~ resource "kubernetes_deployment_v1" "web" {
        id               = "web/web"
      ~ spec {
          ~ replicas                  = "2" -> "3"
            # (4 unchanged attributes hidden)
            # (3 unchanged blocks hidden)
        }
        # (1 unchanged block hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

После `yes` вы увидите `Apply complete! Resources: 0 added, 1 changed, 0 destroyed.` Изменилось только одно поле; пространство имён и ConfigMap остались нетронутыми.

Значение из `-var` не запоминается: следующий `tofu plan` без него захочет вернуть 2. Значения, которые должны остаться, кладут в файл `terraform.tfvars` — OpenTofu читает его автоматически:

```bash
echo 'replicas = 3' > terraform.tfvars
tofu plan
```

```text
kubernetes_namespace_v1.web: Refreshing state... [id=web]
kubernetes_config_map_v1.page: Refreshing state... [id=web/page]
kubernetes_deployment_v1.web: Refreshing state... [id=web/web]

No changes. Your infrastructure matches the configuration.
```

«Refreshing state» означает, что OpenTofu спросил кластер о каждом ресурсе, которым управляет. Так он и замечает **дрейф**. Сделайте изменение вручную, как мог бы сделать коллега тяжёлой ночью, и снова постройте план:

```bash
kubectl -n web scale deployment web --replicas=1
tofu plan
```

```text
deployment.apps/web scaled
...
          ~ replicas                  = "1" -> "3"
...
Plan: 0 to add, 1 to change, 0 to destroy.
```

OpenTofu обнаружил, что реальность (1) отличается от описания (3), и `tofu apply` вернул бы всё назад. Поэтому команда договаривается об одном правиле: **управляемую инфраструктуру меняют только через файлы.** Ручное исправление, не записанное в файлы, отменится при следующем `apply`. Запустите сейчас `tofu apply` и введите `yes`, чтобы убрать дрейф.

> ⚠️ **Частая ошибка:** два инструмента управляют одним объектом. Если Deployment `web` управляют и Helm-чарт, *и* OpenTofu, каждый отменяет изменения другого. У каждого объекта должен быть ровно один владелец.

## 5. Файл состояния

После `apply` в папке появляется новый файл `terraform.tfstate`. Это JSON (JavaScript Object Notation): список адресов всех ресурсов и настоящих объектов за ними.

```bash
tofu state list
tofu state show kubernetes_config_map_v1.page
```

Сокращённый вывод:

```text
kubernetes_config_map_v1.page
kubernetes_deployment_v1.web
kubernetes_namespace_v1.web
# kubernetes_config_map_v1.page:
resource "kubernetes_config_map_v1" "page" {
    ...
    id          = "web/page"
    metadata {
        name             = "page"
        namespace        = "web"
        resource_version = "582"
        uid              = "ddf06963-6a2d-4f8c-b99d-2d8000f3c293"
    ...
```

Зачем OpenTofu вообще состояние, если он может спросить кластер? Из-за того, что вы *удаляете* из файлов. Если удалить блок ConfigMap, файлы больше о нём не упоминают. Только состояние помнит «я создал `web/page` для этого блока», и поэтому следующий план может сказать `- destroy`. Без состояния OpenTofu не смог бы отличить свои объекты от чужих.

Правила для состояния:

- **Никогда не редактируйте его вручную.** Если очень нужно что-то в нём изменить, используйте команды `tofu state`.
- **Никогда не теряйте его.** Без него OpenTofu забывает, чем владеет. В команде состояние хранится в общем **бэкенде** (backend), например в бакете хранилища, которое понимает протокол S3 (Simple Storage Service), или в базе данных PostgreSQL, а не на одном ноутбуке. Хороший бэкенд ещё и **блокирует** состояние, чтобы два человека не применяли изменения одновременно.
- **Не коммитьте его в Git.** Добавьте эти строки в `.gitignore` проекта:

```text
.terraform/
*.tfstate
*.tfstate.backup
```

А вот lock-файл `.terraform.lock.hcl` **коммитят**, как `Cargo.lock`.

> 🔒 **Безопасность:** состояние содержит все атрибуты всех ресурсов, *включая секретные значения*: данные Secret в Kubernetes, пароли баз данных, закрытые ключи. Кто может прочитать состояние, может прочитать и эти секреты. Храните его в бэкенде с контролем доступа, никогда в Git, никогда в публичном бакете. OpenTofu умеет и сам шифровать состояние (блок `encryption`, см. источники).

## 6. Автоматические проверки: `tofu test`

Код на Rust вы проверяете через `cargo fmt`, `cargo clippy` и `cargo test`. У OpenTofu те же три уровня: `tofu fmt -check`, `tofu validate` и `tofu test`.

Файл теста заканчивается на `.tftest.hcl` и лежит в папке `tests/`. Каждый блок `run` выполняет план (или применение) с заданными переменными и проверяет условия. Создайте `tests/web.tftest.hcl`:

```hcl
run "uses_the_replicas_variable" {
  command = plan

  variables {
    replicas = 4
  }

  assert {
    condition     = kubernetes_deployment_v1.web.spec[0].replicas == "4"
    error_message = "the Deployment must use var.replicas"
  }
}

run "rejects_zero_replicas" {
  command = plan

  variables {
    replicas = 0
  }

  expect_failures = [var.replicas]
}
```

**Что происходит:**

1. `command = plan` заставляет тест только *строить план*. В кластере ничего не создаётся и не меняется, поэтому тест быстрый и безопасный. (С `command = apply` OpenTofu создаёт для теста настоящие ресурсы и потом удаляет их.)
2. `variables` задаёт входные значения для этого запуска. Они важнее, чем `terraform.tfvars`.
3. `assert` проверяет условие на запланированных значениях. Провайдер хранит `replicas` как текст, поэтому мы сравниваем с `"4"`.
4. `expect_failures = [var.replicas]` говорит: «этот запуск *должен* упасть, и ошибка должна прийти из проверки `var.replicas`». Так тестируется, что плохой ввод отклоняется.

```bash
tofu test
```

```text
tests/web.tftest.hcl... pass
  run "uses_the_replicas_variable"... pass
  run "rejects_zero_replicas"... pass

Success! 2 passed, 0 failed.
```

Чтобы увидеть, что тест умеет падать, замените в `main.tf` строку `replicas = var.replicas` на `replicas = 2` и снова запустите `tofu test`:

```text
tests/web.tftest.hcl... fail
  run "uses_the_replicas_variable"... fail

Error: Test assertion failed
  on tests/web.tftest.hcl line 9, in run "uses_the_replicas_variable":
   9:     condition     = kubernetes_deployment_v1.web.spec[0].replicas == "4"
    │ kubernetes_deployment_v1.web.spec[0].replicas is "2"

the Deployment must use var.replicas
  run "rejects_zero_replicas"... pass

Failure! 1 passed, 1 failed.
```

(Сокращено: мы убрали строки рамки и разницу значений.)

Верните `var.replicas` на место.

> 🧪 **Тестирование:** в CI запускайте `tofu fmt -check -recursive`, `tofu validate` и `tofu test` для каждого pull request, до того как кто-то посмотрит на план. Тестам с `command = plan` всё равно нужно подключение провайдера (здесь — кластер kind); в конвейере сначала создают кластер kind, как в Уроке 21.7.

## 7. Удаляем всё

`tofu destroy` удаляет все ресурсы из состояния в обратном порядке зависимостей. Он тоже спрашивает `yes`. Когда закончите урок (после упражнений), запустите:

```bash
tofu destroy
```

```text
Plan: 0 to add, 0 to change, 3 to destroy.
...
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
...
kubernetes_namespace_v1.web: Destruction complete after 6s

Destroy complete! Resources: 3 destroyed.
```

(Если вы сделали упражнения, числа будут больше.) Затем удалите кластер: `kind delete cluster --name course-21-08`.

> 💡 **Совет:** когда одна и та же группа ресурсов нужна несколько раз (тестовое и продакшен-окружение), перенесите её в **модуль** (module): папку с файлами `.tf` и собственными переменными, которую используют как функцию: `module "test" { source = "./modules/web" ... }`. Тот же стиль конфигурации работает и для других систем. Провайдер `kreuzwerker/docker` управляет локальными контейнерами и сетями Docker; облачные провайдеры управляют настоящими кластерами. Меняются только типы ресурсов; `init`, `plan`, `apply`, состояние и дрейф работают везде одинаково.

## Частые ошибки

> ⚠️ **Частая ошибка:** запуск `tofu plan` в новой папке (или после добавления провайдера) без `tofu init`:

```text
Error: Inconsistent dependency lock file
...
  - provider registry.opentofu.org/hashicorp/kubernetes: required by this configuration but no version is selected
...
  tofu init
```

Исправление: запустите `tofu init`.

> ⚠️ **Частая ошибка:** неправильное имя контекста в блоке `provider` (здесь `kind-course-21.08` с точкой вместо дефиса):

```text
Error: Provider configuration: cannot load Kubernetes client config
...
context "kind-course-21.08" does not exist
```

Исправление: `kubectl config get-contexts` показывает настоящие имена.

> ⚠️ **Частая ошибка:** удалить `terraform.tfstate`, чтобы «начать с чистого листа». Объекты в кластере остаются, но OpenTofu о них забывает. Следующий `apply` пытается создать их снова и падает с ошибкой «already exists». Вместо этого используйте `tofu destroy`.

## Попробуйте сами

1. Замените текст в ConfigMap на `<h1>Changed by OpenTofu</h1>\n` и запустите `tofu plan`. Какой символ и какую итоговую строку вы видите? (`~`, `Plan: 0 to add, 1 to change, 0 to destroy.`) Примените, проверьте страницу через port-forward, затем верните прежний текст и примените снова.
2. Удалите пространство имён вручную: `kubectl delete namespace web`. Запустите `tofu plan`. OpenTofu замечает, что все три объекта исчезли, и планирует создать их заново (`3 to add`). Примените, чтобы всё починить.
3. Запустите `tofu output replicas`. Команда печатает только это одно выходное значение, что удобно в скриптах.

## Упражнения

### Упражнение 1 — Добавьте Service (легко)

Добавьте ресурс `kubernetes_service_v1` с именем `web` в то же пространство имён. Он выбирает поды с `app = "web"` и перенаправляет порт 80 на порт 80. `tofu apply` должен показать `Plan: 1 to add, 0 to change, 0 to destroy.`, а `kubectl -n web port-forward service/web 8088:80` вместе с `curl -s localhost:8088` должны напечатать страницу.

<details>
<summary>Подсказка</summary>

Структура та же, что у Service на YAML из Урока 21.2: `metadata` с именем и пространством имён и `spec` с `selector` и одним блоком `port` с полями `port` и `target_port`.

</details>

<details>
<summary>Ответ</summary>

Добавьте в `main.tf`:

```hcl
resource "kubernetes_service_v1" "web" {
  metadata {
    name      = "web"
    namespace = kubernetes_namespace_v1.web.metadata[0].name
  }

  spec {
    selector = { app = "web" }

    port {
      port        = 80
      target_port = 80
    }
  }
}
```

```text
Plan: 1 to add, 0 to change, 0 to destroy.
...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Почему это работает: ссылка на пространство имён заставляет OpenTofu создать Service после пространства имён. В плане появляется только новый ресурс, потому что остальные уже совпадают с описанием.

</details>

### Упражнение 2 — Переменная с тестом (средне)

Добавьте переменную `message` (строка, по умолчанию `"Created by OpenTofu"`, от 1 до 80 символов) и используйте её на странице: `"<h1>${var.message}</h1>\n"` (`${...}` вставляет значение в строку). Затем **напишите тест** — файл `tests/page.tftest.hcl` с двумя запусками: первый проверяет, что при `message = "Hello from a test"` запланированная страница равна ровно `<h1>Hello from a test</h1>\n`, второй — что пустое сообщение отклоняется. `tofu test` должен пройти все запуски, а `tofu plan` по-прежнему должен говорить «No changes».

<details>
<summary>Подсказка</summary>

В условии проверки используйте `length(var.message)`. В тесте читайте запланированную страницу как `kubernetes_config_map_v1.page.data["index.html"]`.

</details>

<details>
<summary>Ответ</summary>

В `variables.tf`:

```hcl
variable "message" {
  description = "The heading shown on the page."
  type        = string
  default     = "Created by OpenTofu"

  validation {
    condition     = length(var.message) > 0 && length(var.message) <= 80
    error_message = "message must have 1 to 80 characters."
  }
}
```

В `main.tf` данные ConfigMap становятся `"index.html" = "<h1>${var.message}</h1>\n"`. Файл теста:

```hcl
run "page_shows_the_message" {
  command = plan

  variables {
    message = "Hello from a test"
  }

  assert {
    condition     = kubernetes_config_map_v1.page.data["index.html"] == "<h1>Hello from a test</h1>\n"
    error_message = "the page must contain var.message inside <h1>"
  }
}

run "rejects_an_empty_message" {
  command = plan

  variables {
    message = ""
  }

  expect_failures = [var.message]
}
```

```text
tests/page.tftest.hcl... pass
  run "page_shows_the_message"... pass
  run "rejects_an_empty_message"... pass
tests/web.tftest.hcl... pass
  run "uses_the_replicas_variable"... pass
  run "rejects_zero_replicas"... pass

Success! 4 passed, 0 failed.
```

Почему это работает: значение по умолчанию даёт тот же текст, что и раньше, поэтому настоящему кластеру изменения не нужны. Тесты работают в режиме плана, поэтому проверяют конфигурацию, не трогая кластер.

</details>

### Упражнение 3 — Удалите ресурс (сложно)

Уберите ConfigMap из файлов: удалите его блок `resource`, а также блоки `volume_mount` и `volume` в Deployment, которые его используют. **До** запуска чего-либо запишите, что скажет план: какие ресурсы, какие символы, какая итоговая строка. Затем запустите `tofu plan` и сравните. Наконец, объясните в двух предложениях, какая часть OpenTofu делает этот план возможным. (Потом отмените правку или оставьте её и примените.)

<details>
<summary>Подсказка</summary>

Файлы больше вообще не упоминают ConfigMap. Откуда OpenTofu узнаёт, что он существует?

</details>

<details>
<summary>Ответ</summary>

Сокращённый вывод:

```text
  # kubernetes_config_map_v1.page will be destroyed
  # (because kubernetes_config_map_v1.page is not in configuration)
  - resource "kubernetes_config_map_v1" "page" {
  ...
  # kubernetes_deployment_v1.web will be updated in-place
  ~ resource "kubernetes_deployment_v1" "web" {
  ...
                      - volume_mount {
  ...
                  - volume {
  ...
Plan: 0 to add, 1 to change, 1 to destroy.
```

Почему это работает: **состояние** по-прежнему содержит `kubernetes_config_map_v1.page` с настоящим объектом `web/page`. OpenTofu сравнивает состояние с файлами, видит ресурс, который больше не описан, и планирует его удалить («not in configuration»); Deployment теряет только два блока, поэтому он меняется на месте, а не пересоздаётся.

</details>

## Вопросы для самопроверки

1. Что значит «декларативный» для инструмента IaC?

<details>
<summary>Ответ</summary>

Вы описываете желаемый результат, а не шаги. Инструмент сравнивает описание с реальностью и сам находит нужные изменения.

</details>

2. Зачем OpenTofu файл состояния, если он может спросить кластер?

<details>
<summary>Ответ</summary>

Чтобы знать, какие настоящие объекты принадлежат каким блокам ресурсов. Когда вы удаляете блок из файлов, только состояние помнит, что объект существует и его нужно удалить.

</details>

3. Почему состояние никогда не должно попадать в Git?

<details>
<summary>Ответ</summary>

Оно содержит все атрибуты всех ресурсов, включая секретные значения. Кроме того, оно меняется при каждом применении и должно быть заблокировано, пока кто-то применяет изменения, а Git этого не умеет.

</details>

4. Коллега вручную увеличил Deployment до 6 реплик, а в файлах написано 3. Что произойдёт при следующем `tofu apply`?

<details>
<summary>Ответ</summary>

План покажет дрейф (`6 -> 3`), и применение вернёт 3 реплики. Если правильно 6, меняйте файлы, а не кластер.

</details>

5. Что делает `command = plan` в тесте и что проверяет `expect_failures`?

<details>
<summary>Ответ</summary>

`command = plan` выполняет только план, поэтому ничего не создаётся. `expect_failures` говорит, что запуск должен упасть из-за названной проверки, например проверки переменной.

</details>

## Итоги

- IaC описывает инфраструктуру в файлах в Git, поэтому изменения проверяются, повторяются и видны до того, как произойдут.
- OpenTofu читает файлы `.tf` на HCL; провайдеры превращают ресурсы в вызовы API. Ссылки между ресурсами задают порядок.
- Цикл такой: `tofu init` → `tofu plan` → `tofu apply` → (в конце) `tofu destroy`. Всегда читайте итоговую строку плана.
- Состояние связывает ресурсы с настоящими объектами. Защищайте его как пароль, храните в бэкенде с блокировкой, никогда в Git.
- `tofu plan` находит дрейф. Меняйте управляемые вещи только через файлы и давайте каждому объекту одного владельца.
- Проверяйте конфигурации через `tofu fmt -check`, `tofu validate`, проверки переменных и `tofu test`.

## Источники и что почитать

- [OpenTofu — What is OpenTofu?](https://opentofu.org/docs/intro/) — официальное введение и основной рабочий процесс.
- [OpenTofu — Installing with Homebrew](https://opentofu.org/docs/intro/install/homebrew/) и [установка отдельным файлом](https://opentofu.org/docs/intro/install/standalone/) — как установить команду `tofu`.
- [OpenTofu — State](https://opentofu.org/docs/language/state/), [Backends](https://opentofu.org/docs/language/state/backends/) и [State and plan encryption](https://opentofu.org/docs/language/state/encryption/) — зачем нужно состояние, где его хранить и как шифровать.
- [OpenTofu — Input variables](https://opentofu.org/docs/language/values/variables/) и [Modules](https://opentofu.org/docs/language/modules/) — переменные, проверки и переиспользуемые модули.
- [OpenTofu — The `tofu test` command](https://opentofu.org/docs/cli/commands/test/) — файлы тестов, блоки `run`, `assert` и `expect_failures`.
- [Провайдер Kubernetes в реестре OpenTofu](https://search.opentofu.org/provider/hashicorp/kubernetes/latest) — все типы ресурсов и их поля.

---

[🏠 Начало](../START-HERE.md) · [← 21.7 CI/CD: от коммита до продакшена](07-ci-cd-from-commit-to-production.md) · [📋 Этап 21](README.md) · [21.9 Метрики с Prometheus и Grafana →](09-metrics-with-prometheus-and-grafana.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
