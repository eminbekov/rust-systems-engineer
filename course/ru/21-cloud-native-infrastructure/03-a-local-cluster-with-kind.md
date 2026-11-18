<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.2 Понятия Kubernetes: поды, Deployment и Service](02-kubernetes-concepts.md) · [📋 Этап 21](README.md) · [21.4 Развёртывание сервиса на Rust в Kubernetes →](04-deploying-a-rust-service-to-kubernetes.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) <!-- nav -->

# Урок 21.3 — Локальный кластер с kind и kubectl

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 3 из 14 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как установить kind и kubectl на Mac и создать настоящий кластер Kubernetes внутри Docker.
- Что такое файл kubeconfig и контекст и как kubectl понимает, с каким кластером говорить.
- Повседневные команды kubectl: `get`, `describe`, `logs`, `exec`, `delete`, `scale`, `expose`, `port-forward`, `run`.
- Как увидеть, что контроллеры из Урока 21.2 заменяют удалённый под.
- Как перейти от набранных команд к файлам манифестов с помощью `--dry-run=client -o yaml`, `apply` и `diff`.

## Перед началом

- Вы прошли [Урок 21.2 — Понятия Kubernetes](02-kubernetes-concepts.md).
- Docker Desktop установлен и запущен ([Урок 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)), и в *Settings → Resources* ему выделено не меньше 4 ГБ памяти.
- Вы знаете Homebrew — менеджер пакетов для macOS ([Урок 00.5](../00-getting-started/05-install-rust-and-rustrover.md)) — и строгие скрипты на bash ([Урок 11.9 — Надёжные скрипты на bash](../11-linux-shell-and-containers/09-robust-bash-scripts.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **kind** (Kubernetes IN Docker, Kubernetes в Docker) | Инструмент, который создаёт кластер Kubernetes, узлы которого — контейнеры Docker. Сделан проектом Kubernetes для тестирования. |
| **kubectl** | Клиент командной строки для Kubernetes. Отправляет ваши запросы API-серверу (API — Application Programming Interface, программный интерфейс). Произносят «куб-контрол» или «куб-си-ти-эл». |
| **kubeconfig** | Файл (по умолчанию `~/.kube/config`), в котором перечислены кластеры, пользователи с их учётными данными и контексты. |
| **контекст** (context) | Именованная пара «этот кластер + этот пользователь (+ при желании пространство имён)» в kubeconfig. *Текущий контекст* решает, куда kubectl отправляет команды. |
| **событие** (event) | Короткая запись, которую компонент пишет об объекте, например «Pulling image» или «Failed». Их показывает `kubectl describe`. |
| **проброс порта** (port-forward) | Возможность kubectl открыть порт на вашем компьютере и провести его через API-сервер к поду или Service. |
| **ImagePullBackOff** | Статус пода: узел не смог скачать образ и ждёт всё дольше и дольше перед новой попыткой. |
| **пробный запуск** (dry run) | Выполнение команды без внесения изменений, чтобы увидеть, что она сделала бы. |
| **дымовой тест** (smoke test) | Очень быстрый тест, который проверяет, что что-то вообще работает («отвечает ли?»), перед более глубокими тестами. |

## 1. Установка kind и kubectl

Установите оба инструмента через Homebrew:

```bash
brew install kind kubectl
```

Затем проверьте их. Ваши версии могут быть новее, чем те, с которыми писался этот урок:

```bash
kind version
kubectl version --client
```

```text
kind v0.33.0 go1.26.7 darwin/arm64
Client Version: v1.37.2
Kustomize Version: v5.8.1
```

`kubectl version --client` печатает только версию клиента; без `--client` он спросил бы версию и у кластера, а его у нас пока нет. Kustomize — инструмент, встроенный в kubectl; вы будете использовать его в Уроке 21.6.

> 💡 **Совет:** без Homebrew обе программы можно скачать с их официальных страниц (ссылки в конце): для kind — файл `kind-darwin-arm64` со страницы релизов; для kubectl — `https://dl.k8s.io/release/v1.37.2/bin/darwin/arm64/kubectl`. Рядом с каждым лежит файл `.sha256`. Сравните контрольную сумму (`shasum -a 256 <файл>`), прежде чем делать файл исполняемым и класть его в папку из вашего `PATH`.

## 2. Создание кластера

Убедитесь, что Docker Desktop запущен, затем:

```bash
kind create cluster --name course-21-03
```

```text
Creating cluster "course-21-03" ...
 ✓ Ensuring node image (kindest/node:v1.37.0) 🖼️
 ✓ Preparing nodes 📦 
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-course-21-03"
You can now use your cluster with:

kubectl cluster-info --context kind-course-21-03

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

(Каждый шаг сначала печатает строку с `•`, а потом заменяет её строкой с `✓`; мы показываем только итоговые строки. Последняя строка — случайное дружелюбное сообщение.) При первом запуске скачивается образ узла (на диске больше 1 ГБ), поэтому это может занять несколько минут. Следующие запуски занимают 15–30 секунд.

**Что происходит:**

1. kind запускает **один контейнер Docker** из образа `kindest/node`. Внутри работают systemd, containerd и kubelet; это целый узел Kubernetes.
2. Он запускает в этом контейнере плоскость управления (API-сервер, etcd, планировщик, менеджер контроллеров), устанавливает сетевой плагин (CNI, Container Network Interface, сетевой интерфейс контейнеров) и простой провайдер хранилища.
3. Он записывает адрес и учётные данные нового кластера в ваш **kubeconfig** и делает его текущим контекстом.

Узел — это просто контейнер. Docker его показывает (мы фильтруем по имени, чтобы вы видели только объекты курса):

```bash
docker ps --filter name=course-21-03 --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
```

```text
NAMES                        IMAGE                  PORTS
course-21-03-control-plane   kindest/node:v1.37.0   127.0.0.1:62668->6443/tcp
```

API-сервер слушает порт 6443 внутри контейнера, а Docker публикует его на случайном порту вашего Mac, только на `127.0.0.1`. Ваш номер порта будет другим.

## 3. kubeconfig и контексты

kubectl сам по себе ничего не знает о кластерах. При каждом запуске он читает файл **kubeconfig** (`~/.kube/config`, если переменная окружения `KUBECONFIG` не указывает на другой) и использует **текущий контекст**:

```bash
kubectl config current-context
kubectl config get-contexts
```

```text
kind-course-21-03
CURRENT   NAME                CLUSTER             AUTHINFO            NAMESPACE
*         kind-course-21-03   kind-course-21-03   kind-course-21-03   
```

Контекст соединяет **кластер** (адрес и сертификат API-сервера) с **AuthInfo** (пользователем и его учётными данными). Если позже вы будете работать с несколькими кластерами, здесь будет несколько строк, а переключаться можно командой `kubectl config use-context <имя>`. `*` отмечает текущий.

```bash
kubectl cluster-info
```

```text
Kubernetes control plane is running at https://127.0.0.1:62668
CoreDNS is running at https://127.0.0.1:62668/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

> 🔒 **Безопасность:** kubeconfig содержит учётные данные. Тот, что пишет kind, даёт **полные права администратора** в его кластере. Никогда не коммитьте kubeconfig в Git, не вставляйте его в чат и никогда не копируйте kubeconfig настоящего кластера на общий компьютер. Для kind это не так важно (кластер живёт только на вашем Mac и исчезает, когда вы его удаляете), но важна привычка.

После создания узлу нужно несколько секунд, чтобы стать готовым. Дождитесь этого и посмотрите на него:

```bash
kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes -o wide
```

```text
node/course-21-03-control-plane condition met
NAME                         STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION            CONTAINER-RUNTIME
course-21-03-control-plane   Ready    control-plane   21s   v1.37.0   172.30.0.3    <none>        Debian GNU/Linux 13 (trixie)   7.0.14-linuxkit (arm64)   containerd://2.3.4
```

`-o wide` добавляет столбцы. Узел работает на Debian 13 внутри Linux-виртуальной машины Docker Desktop (ядро `linuxkit`) на arm64, а средой выполнения контейнеров служит containerd: ровно картинка из Урока 21.2.

Каждая команда `kubectl get` устроена одинаково: `kubectl get <вид> [имя] [параметры]`. Попробуйте `kubectl get namespaces`:

```text
NAME                 STATUS   AGE
default              Active   22s
kube-node-lease      Active   22s
kube-public          Active   22s
kube-system          Active   22s
local-path-storage   Active   18s
```

## 4. Ваш первый Deployment

Для исследования у kubectl есть *императивные* сокращения, которые создают объекты без файла. Создайте Deployment с двумя копиями nginx:

```bash
kubectl create deployment web --image=nginx:1.30-alpine --replicas=2
kubectl rollout status deployment/web
```

```text
deployment.apps/web created
Waiting for deployment "web" rollout to finish: 0 of 2 updated replicas are available...
Waiting for deployment "web" rollout to finish: 1 of 2 updated replicas are available...
deployment "web" successfully rolled out
```

`kubectl rollout status` ждёт, пока Deployment не достигнет желаемого состояния, — это удобно в скриптах. Теперь выведите список подов:

```bash
kubectl get pods -o wide
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
web-6f5d6d9c94-wdj7t   1/1     Running   0          30s   10.244.0.6   course-21-03-control-plane   <none>           <none>
web-6f5d6d9c94-zxxt8   1/1     Running   0          30s   10.244.0.5   course-21-03-control-plane   <none>           <none>
```

`READY 1/1` значит, что готов один контейнер из одного. У каждого пода свой IP-адрес (IP — Internet Protocol, межсетевой протокол). Имена ваших подов будут другими (суффиксы случайные).

### Заглянуть внутрь: describe, logs, exec

`kubectl describe` печатает всё об одном объекте, включая **события**, которые компоненты о нём записали. Подставьте имя своего пода:

```bash
kubectl describe pod web-6f5d6d9c94-wdj7t
```

```text
Name:             web-6f5d6d9c94-wdj7t
Namespace:        default
Node:             course-21-03-control-plane/172.30.0.3
Labels:           app=web
                  pod-template-hash=6f5d6d9c94
Status:           Running
IP:               10.244.0.6
Controlled By:  ReplicaSet/web-6f5d6d9c94
Containers:
  nginx:
    Image:          nginx:1.30-alpine
    State:          Running
    Ready:          True
    Restart Count:  0
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  30s   default-scheduler  Successfully assigned default/web-6f5d6d9c94-wdj7t to course-21-03-control-plane
  Normal  Pulling    30s   kubelet            spec.containers{nginx}: Pulling image "nginx:1.30-alpine"
  Normal  Pulled     0s    kubelet            spec.containers{nginx}: Successfully pulled image "nginx:1.30-alpine" in 1.354s (29.467s including waiting). Image size: 25987805 bytes.
  Normal  Created    0s    kubelet            spec.containers{nginx}: Container created
  Normal  Started    0s    kubelet            spec.containers{nginx}: Container started
```

(Сокращено: в настоящем выводе есть ещё условия, тома и многое другое.) Прочитайте события сверху вниз, и вы увидите, как происходит Урок 21.2: **планировщик** назначил под на узел, затем **kubelet** скачал образ, создал контейнер и запустил его. `Controlled By: ReplicaSet/...` — это ссылка на владельца. Когда под не запускается, события — первое, куда стоит смотреть.

`kubectl logs` показывает, что контейнер написал в стандартный вывод. С `deployment/web` kubectl сам выбирает один под из Deployment:

```bash
kubectl logs deployment/web --tail=3
```

```text
Found 2 pods, using pod/web-6f5d6d9c94-zxxt8
2026/11/18 08:06:47 [notice] 1#1: start worker process 42
2026/11/18 08:06:47 [notice] 1#1: start worker process 43
2026/11/18 08:06:47 [notice] 1#1: start worker process 44
```

`kubectl exec` выполняет команду внутри работающего контейнера, как `docker exec`:

```bash
kubectl exec deployment/web -- nginx -v
```

```text
nginx version: nginx/1.30.6
```

Всё после `--` — команда для контейнера. Используйте `exec`, чтобы смотреть, а не менять: помните правило стада из Урока 21.1.

## 5. Посмотрите, как работают контроллеры

Теперь самый важный эксперимент урока. Удалите один под (подставьте имя одного из своих подов):

```bash
kubectl delete pod web-6f5d6d9c94-wdj7t
kubectl get pods
```

```text
pod "web-6f5d6d9c94-wdj7t" deleted from default namespace
NAME                   READY   STATUS    RESTARTS   AGE
web-6f5d6d9c94-jczm7   1/1     Running   0          2s
web-6f5d6d9c94-zxxt8   1/1     Running   0          32s
```

Подов по-прежнему два. Удалённого нет, а его место занял **новый** под (`jczm7`, ему 2 секунды). Контроллер ReplicaSet увидел «желаемо 2, фактически 1» и подействовал: цикл согласования из Урока 21.1 в настоящей работе.

> 💡 **Совет:** чтобы следить за изменениями вживую, запустите `kubectl get pods --watch` (или `-w`) во второй вкладке терминала до удаления пода. Каждое изменение печатает новую строку; остановите наблюдение через Ctrl+C.

Измените желаемое состояние командой `scale`:

```bash
kubectl scale deployment web --replicas=4
kubectl rollout status deployment/web
kubectl get deployment web
```

```text
deployment.apps/web scaled
Waiting for deployment "web" rollout to finish: 2 of 4 updated replicas are available...
Waiting for deployment "web" rollout to finish: 3 of 4 updated replicas are available...
deployment "web" successfully rolled out
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    4/4     4            4           33s
```

`READY 4/4`: готовы четыре пода из четырёх. `scale` поменял только число `spec.replicas` в Deployment; остальное сделали контроллеры.

## 6. Доступ к подам

У подов есть IP-адреса только внутри кластера. Чтобы до них достучаться, сначала дайте им Service. `kubectl expose` создаёт Service, который выбирает поды Deployment:

```bash
kubectl expose deployment web --port=80
kubectl get service web
```

```text
service/web exposed
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.96.27.137   <none>        80/TCP    0s
```

Адрес ClusterIP недоступен с вашего Mac. **Проброс порта** открывает локальный порт и проводит его через API-сервер к Service. Запустите его во **второй вкладке терминала**, потому что он работает, пока вы не нажмёте Ctrl+C:

```bash
kubectl port-forward service/web 8088:80
```

```text
Forwarding from 127.0.0.1:8088 -> 80
Forwarding from [::1]:8088 -> 80
```

`8088:80` означает «локальный порт 8088 к порту 80 Service». В первой вкладке:

```bash
curl -s http://localhost:8088 | head -4
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Проброс порта — инструмент разработчика. Он выбирает **один** под за Service и продолжает использовать его, так что это не балансировщик нагрузки, и он останавливается, когда этот под исчезает. Остановите его сейчас через Ctrl+C во второй вкладке.

Другой способ — проверять **изнутри** кластера временным подом. `kubectl run` создаёт один под; `--rm -i --restart=Never` заставляет его выполниться один раз, показать вывод и удалиться; `-q` скрывает собственные сообщения kubectl:

```bash
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://web | grep title
```

```text
<title>Welcome to nginx!</title>
```

`busybox` — небольшой официальный образ с базовыми утилитами Unix, включая `wget`. Внутри кластера имя `web` превращается в адрес Service (Урок 21.2).

## 7. От команд к файлам

Императивные команды хороши для исследования, но настоящий способ работы — декларативный: манифесты в Git, применяемые через `kubectl apply`. Первый манифест не обязательно писать по памяти. Удалите то, что создали, а затем пусть kubectl напишет YAML (YAML Ain't Markup Language — формат манифестов) за вас:

```bash
kubectl delete deployment,service web
kubectl create deployment web --image=nginx:1.30-alpine --replicas=2 --dry-run=client -o yaml > web.yaml
cat web.yaml
```

```text
deployment.apps "web" deleted from default namespace
service "web" deleted from default namespace
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  strategy: {}
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: nginx:1.30-alpine
        name: nginx
        resources: {}
status: {}
```

`--dry-run=client` означает «собери объект, но не отправляй»; `-o yaml` печатает его. Пустые `strategy: {}`, `resources: {}` и `status: {}` можно удалить; это заглушки. (kubectl пишет элементы списка без дополнительного отступа под ключом; оба стиля — корректный YAML.)

Теперь примените файл:

```bash
kubectl apply -f web.yaml
```

```text
deployment.apps/web created
```

Поменяйте `replicas: 2` на `replicas: 3` в `web.yaml`. Прежде чем применять изменение, спросите кластер, что *изменилось бы*:

```bash
kubectl diff -f web.yaml
```

```text
@@ -6,7 +6,7 @@
...
-  generation: 1
+  generation: 2
...
@@ -15,7 +15,7 @@
 spec:
   progressDeadlineSeconds: 600
-  replicas: 2
+  replicas: 3
   revisionHistoryLimit: 10
```

(Сокращено: настоящий вывод начинается с имён двух временных файлов и показывает несколько неизменённых строк вокруг каждого изменения.) `diff` сравнивает объект в кластере с объектом, который получился бы из вашего файла. Строки с `-` уходят, строки с `+` приходят. `generation` — счётчик изменений `spec`. Теперь примените, дважды:

```bash
kubectl apply -f web.yaml
kubectl apply -f web.yaml
```

```text
deployment.apps/web configured
deployment.apps/web unchanged
```

Второй `apply` ничего не меняет: `apply` **идемпотентен** — то самое свойство из Урока 21.1. Поэтому манифесты в Git плюс `kubectl apply` — безопасный способ работы: всю папку можно применить снова в любой момент.

> 🧪 **Тестирование:** `kubectl diff` завершается с кодом `1`, когда различия есть, и с `0`, когда их нет. Задание CI (Continuous Integration, непрерывная интеграция) может запускать его, чтобы проверять, что кластер всё ещё совпадает с Git (это называется *обнаружением дрейфа*, drift detection), а параметр `--dry-run=server` у `apply` просит API-сервер проверить манифест, не сохраняя его.

## 8. Уборка

Удалите объекты, а затем весь кластер:

```bash
kubectl delete -f web.yaml
kind delete cluster --name course-21-03
```

```text
deployment.apps "web" deleted from default namespace
Deleting cluster "course-21-03" ...
Deleted nodes: ["course-21-03-control-plane"]
```

`kind delete cluster` удаляет контейнер узла и запись кластера из вашего kubeconfig. Ваш `web.yaml` остаётся; вы можете за минуту создать новый кластер и снова его применить. Ради этого всё и хранят в файлах.

## Частые ошибки

**Нет кластера или не тот кластер.** Если у kubectl нет контекста (кластер не создавали или удалили), он обращается к `localhost:8080`, где никто не слушает:

```text
E1118 13:01:11.694393   15979 memcache.go:381] "Couldn't get current server API group list" err="Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

(Первая строка повторяется несколько раз.) Исправление: создайте кластер или проверьте `kubectl config current-context`. Обратная ошибка хуже: при нескольких контекстах команда, предназначенная тестовому кластеру, уходит в другой кластер. Всегда смотрите на текущий контекст перед `delete` или `apply`.

**Не то пространство имён.** `kubectl get pods` показывает только пространство имён текущего контекста (здесь `default`). В свежем кластере он печатает `No resources found in default namespace.`, хотя системные поды работают в `kube-system`. Используйте `kubectl get pods -n kube-system` или `-A` для всех пространств имён.

**Забыть, что проброс порта всё ещё работает.** Второй `kubectl port-forward` на тот же локальный порт падает, потому что порт занят. Остановите старый через Ctrl+C.

## Попробуйте сами

1. Снова создайте кластер `course-21-03` и примените свой `web.yaml`.
2. Откройте вторую вкладку с `kubectl get pods --watch`. В первой вкладке выполните `kubectl scale deployment web --replicas=1`, а затем `--replicas=3`. Следите за появляющимися строками.
3. Удалите под, пока работает наблюдение, и посчитайте, сколько секунд пройдёт, пока новый под не станет `Running`. Затем уберите за собой: `kind delete cluster --name course-21-03`.

## Упражнения

### Упражнение 1 — ImagePullBackOff (легко)

Создайте Deployment с опечаткой в теге: `kubectl create deployment broken --image=nginx:1.30-alpin`. С помощью kubectl выясните, *почему* под не запускается, а затем исправьте Deployment, не удаляя его.

<details>
<summary>Подсказка</summary>

`kubectl get pods -l app=broken` показывает статус; `kubectl describe pod <имя>` показывает события. `kubectl set image deployment/broken <контейнер>=<образ>` меняет образ; имя контейнера — `nginx` (kubectl взял его из имени образа).

</details>

<details>
<summary>Ответ</summary>

```bash
kubectl get pods -l app=broken
```

```text
NAME                      READY   STATUS             RESTARTS   AGE
broken-6f597897d6-r24v9   0/1     ImagePullBackOff   0          25s
```

События `kubectl describe pod broken-6f597897d6-r24v9` заканчиваются так:

```text
  Warning  Failed     11s (x2 over 23s)  kubelet            spec.containers{nginx}: Failed to pull image "nginx:1.30-alpin": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:1.30-alpin": failed to resolve reference "docker.io/library/nginx:1.30-alpin": docker.io/library/nginx:1.30-alpin: not found
  Warning  Failed     11s (x2 over 23s)  kubelet            spec.containers{nginx}: Error: ErrImagePull
```

Тега `1.30-alpin` не существует. Исправьте и проследите за выкаткой:

```bash
kubectl set image deployment/broken nginx=nginx:1.30-alpine
kubectl rollout status deployment/broken
kubectl delete deployment broken
```

```text
deployment.apps/broken image updated
Waiting for deployment "broken" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "broken" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "broken" rollout to finish: 1 old replicas are pending termination...
deployment "broken" successfully rolled out
deployment.apps "broken" deleted from default namespace
```

`ErrImagePull` — первая неудача; `ImagePullBackOff` значит, что kubelet теперь ждёт всё дольше (10 с, 20 с, 40 с, … до 5 минут) между попытками. Смена образа создала новый ReplicaSet с рабочим шаблоном.

</details>

### Упражнение 2 — Скрипт дымового теста (средне)

Напишите `smoke.sh`: строгий скрипт на bash, который проверяет, что Service `web` отвечает внутри кластера страницей, содержащей `Welcome to nginx`. Он должен завершаться с кодом `0`, когда всё в порядке, и с ненулевым кодом (и сообщением в стандартный поток ошибок), когда нет. Проверьте его дважды: один раз с Service, другой — после `kubectl delete service web`.

<details>
<summary>Подсказка</summary>

Используйте `set -euo pipefail` (Урок 11.9), запустите временный под через `kubectl run ... -q --rm -i --restart=Never --image=busybox:1.37 -- sh -c 'sleep 2; wget -qO- -T 5 http://web'` и проверьте текст через `grep -q`. `kubectl run` завершается с кодом возврата команды пода. `sleep 2` важен: `kubectl run -i` подключается к поду только после его запуска, и вывод, напечатанный до этого момента, может потеряться.

</details>

<details>
<summary>Ответ</summary>

```bash
#!/usr/bin/env bash
# smoke.sh: does the Service "web" answer inside the cluster?
set -euo pipefail

# "sleep 2" gives kubectl time to attach before wget prints anything.
if ! body=$(kubectl run smoke -q --rm -i --restart=Never --image=busybox:1.37 \
    -- sh -c 'sleep 2; wget -qO- -T 5 http://web' 2>/dev/null); then
  echo "smoke: the service did not answer" >&2
  exit 1
fi

if ! grep -q 'Welcome to nginx' <<<"$body"; then
  echo "smoke: unexpected page" >&2
  exit 1
fi
echo "smoke: ok"
```

С Service (после `kubectl expose deployment web --port=80`):

```text
smoke: ok
```

После `kubectl delete service web` имени `web` больше нет, `wget` падает, а скрипт печатает `smoke: the service did not answer` и завершается с кодом `1`.

Почему это работает: `kubectl run --rm -i` ждёт под и возвращает код возврата команды (`sh`, который возвращает код возврата `wget`); `if ! body=$(...)` ловит неудачу раньше, чем `set -e` остановит скрипт. Тайм-аут `-T 5` делает тест быстрым. В Уроке 21.7 такой тест запускается в CI после каждого развёртывания.

</details>

### Упражнение 3 — Подсчёт подов по фазам на Rust (сложно)

Напишите программу на Rust `code/p21-podstat`, которая запускает

```text
kubectl get pods --no-headers -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
```

и печатает, сколько подов находится в каждой фазе (`Running`, `Pending`, …), отсортировав по имени фазы. Вынесите разбор в функцию `count_phases(output: &str) -> BTreeMap<String, usize>` и напишите для неё модульный тест, которому не нужен кластер.

<details>
<summary>Подсказка</summary>

`std::process::Command::new("kubectl").args([...]).output()` запускает программу и собирает её вывод (вы использовали `Command` в Уроке 10.5; Урок 11.5 объясняет коды возврата). `output.status.success()` говорит, получилось ли. `*map.entry(key).or_insert(0) += 1` считает.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::BTreeMap;
use std::process::Command;

/// Counts pods per phase from the output of
/// `kubectl get pods --no-headers -o custom-columns=NAME:.metadata.name,PHASE:.status.phase`.
fn count_phases(output: &str) -> BTreeMap<String, usize> {
    let mut counts = BTreeMap::new();
    for line in output.lines() {
        let mut columns = line.split_whitespace();
        if let (Some(_name), Some(phase)) = (columns.next(), columns.next()) {
            *counts.entry(phase.to_string()).or_insert(0) += 1;
        }
    }
    counts
}

fn main() {
    let output = Command::new("kubectl")
        .args([
            "get",
            "pods",
            "--no-headers",
            "-o",
            "custom-columns=NAME:.metadata.name,PHASE:.status.phase",
        ])
        .output()
        .expect("could not start kubectl; is it installed?");
    if !output.status.success() {
        eprintln!("kubectl failed: {}", String::from_utf8_lossy(&output.stderr));
        std::process::exit(1);
    }
    let text = String::from_utf8_lossy(&output.stdout);
    for (phase, count) in count_phases(&text) {
        println!("{phase}: {count}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_each_phase() {
        let output = "web-1   Running\nweb-2   Running\nweb-3   Pending\n";
        let counts = count_phases(output);
        assert_eq!(counts["Running"], 2);
        assert_eq!(counts["Pending"], 1);
        assert_eq!(counts.len(), 2);
    }

    #[test]
    fn ignores_empty_lines() {
        assert!(count_phases("\n\n").is_empty());
    }
}
```

Пока `web` (3 реплики) работал, а сломанный Deployment из Упражнения 1 ещё падал, `cargo run` напечатал:

```text
Pending: 1
Running: 3
```

а `cargo test` напечатал `test result: ok. 2 passed`. У пода в `ImagePullBackOff` фаза `Pending`: фаза — грубая сводка, а подробная причина лежит в статусе контейнера.

Почему это работает: программа сама не говорит с Kubernetes; она переиспользует kubectl, который уже знает kubeconfig и контекст. Разбор — чистая функция, поэтому тесту не нужен кластер (разделение «решить и действовать» из Урока 21.1). Для настоящего инструмента вы бы использовали крейт `kube`, который обращается к API-серверу напрямую.

</details>

## Вопросы для самопроверки

1. Чем физически является «узел» kind на вашем Mac?

<details>
<summary>Ответ</summary>

Контейнером Docker (здесь `course-21-03-control-plane`), запущенным из образа `kindest/node`, в котором работают systemd, containerd, kubelet и компоненты плоскости управления.

</details>

2. Как kubectl узнаёт, с каким кластером говорить?

<details>
<summary>Ответ</summary>

Он читает kubeconfig (`~/.kube/config` или файл из `KUBECONFIG`) и использует текущий контекст, который называет кластер (адрес и сертификат CA — Certificate Authority, удостоверяющего центра, который подписал сертификат API-сервера) и пользователя (учётные данные).

</details>

3. Какая команда показывает, *почему* под не запускается, и какую часть её вывода читать первой?

<details>
<summary>Ответ</summary>

`kubectl describe pod <имя>`, и первым делом раздел `Events` в конце.

</details>

4. Вы удаляете под Deployment с 2 репликами. Что происходит и какой компонент это делает?

<details>
<summary>Ответ</summary>

За несколько секунд создаётся новый под с новым именем, потому что контроллер ReplicaSet видит 1 под там, где желаемо 2. Затем планировщик размещает его, а kubelet запускает.

</details>

5. Почему `kubectl port-forward` не годится, чтобы обслуживать настоящих пользователей?

<details>
<summary>Ответ</summary>

Это отладочный туннель через API-сервер с вашего компьютера: он использует только один под (без балансировки нагрузки), останавливается, когда этот под исчезает или когда вы его закрываете, и работает, только пока работает ваш терминал.

</details>

6. Вы дважды выполняете `kubectl apply -f web.yaml`, не меняя файл. Что печатает второй запуск и какое свойство это показывает?

<details>
<summary>Ответ</summary>

`deployment.apps/web unchanged`. Это показывает, что `apply` идемпотентен: повторное применение того же желаемого состояния ничего не меняет.

</details>

## Итоги

- kind запускает целый кластер Kubernetes в контейнерах Docker; `kind create cluster --name …` и `kind delete cluster --name …` занимают секунды.
- kubectl читает kubeconfig и использует текущий контекст. В kubeconfig хранятся учётные данные: держите его вне Git. Проверяйте контекст, прежде чем что-то менять.
- Ежедневные команды: `get` (с `-o wide`, `-n`, `-A`, `-l`), `describe` (читайте события), `logs`, `exec`, `delete`, `scale`, `rollout status`.
- Удалите под — и появится новый: контроллер ReplicaSet выполняет согласование.
- До подов можно достучаться через Service; с вашего Mac — через `port-forward`, изнутри кластера — временным подом (`kubectl run --rm -i`).
- Работайте декларативно: создайте манифест через `--dry-run=client -o yaml`, храните его в Git, проверяйте через `kubectl diff`, меняйте через `kubectl apply` (идемпотентно).

## Источники и что почитать

- [kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) — установка kind и создание кластеров, в том числе многоузловых.
- [kind — Releases](https://github.com/kubernetes-sigs/kind/releases) — загрузки с контрольными суммами.
- [Документация Kubernetes — Install and Set Up kubectl on macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) — официальные способы установить kubectl с проверкой контрольных сумм.
- [Документация Kubernetes — kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/) — команды этого урока и многие другие.
- [Документация Kubernetes — Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) — кластеры, пользователи и контексты.
- [Документация Kubernetes — Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) — kubectl должен отличаться от кластера не больше чем на одну минорную версию.

---

[🏠 Начало](../START-HERE.md) · [← 21.2 Понятия Kubernetes: поды, Deployment и Service](02-kubernetes-concepts.md) · [📋 Этап 21](README.md) · [21.4 Развёртывание сервиса на Rust в Kubernetes →](04-deploying-a-rust-service-to-kubernetes.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
