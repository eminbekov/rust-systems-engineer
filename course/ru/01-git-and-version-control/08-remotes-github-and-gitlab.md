<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.7 Конфликты слияния](07-merge-conflicts.md) · [📋 Этап 01](README.md) · [01.9 Командная работа: pull request, код-ревью и соглашения →](09-teamwork-pull-requests-and-conventions.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/08-remotes-github-and-gitlab.md) <!-- nav -->

# Урок 01.8 — Удалённые репозитории: GitHub и GitLab

> **Этап 01 — Git и контроль версий** · Урок 8 из 10 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое удалённый репозиторий и как `clone`, `push`, `fetch` и `pull` переносят коммиты между копиями.
- Как отработать всё это **без интернета** с локальной папкой-«сервером».
- Что GitHub и GitLab добавляют к Git и как создать аккаунт и репозиторий.
- Как безопасно входить с помощью SSH-ключа (и почему не по паролю).
- Как читать и исправлять самые частые ошибки: отклонённые push и «permission denied».
- Основные правила безопасности для секретов и ключей.

## Перед началом

- Вы прошли [Урок 01.7 — Конфликты слияния](07-merge-conflicts.md).
- Вы умеете сливать ветки и читать `git log --graph` ([Урок 01.4](04-viewing-history-and-changes.md), [Урок 01.6](06-branches-and-merging.md)).
- Для второй половины нужен бесплатный аккаунт GitHub (и, по желанию, аккаунт GitLab). Если пока не хотите создавать аккаунт, выполните первую половину; часть с аккаунтом не нужна для упражнений следующего урока, но нужна для проекта в [Уроке 01.10](10-project-your-learning-repo-and-checkpoint.md).

## Новые слова

| Слово | Значение |
|---|---|
| **удалённый репозиторий** (remote) | Другая копия вашего репозитория, обычно на сервере. |
| **`origin`** | Имя по умолчанию для удалённого репозитория, из которого вы клонировали или который первым добавили. |
| **clone** (клонировать) | Сделать на своём компьютере полную копию удалённого репозитория со всей историей. |
| **push** (отправить) | Отправить ваши новые коммиты в удалённый репозиторий. |
| **fetch** (получить) | Скачать новые коммиты из удалённого репозитория **без** изменения ваших веток и файлов. |
| **pull** (подтянуть) | `fetch`, за которым следует слияние в текущую ветку. |
| **ветка удалённого отслеживания** (remote-tracking branch) | Ваша локальная пометка о том, где была ветка в удалённом репозитории при последнем контакте, например `origin/main`. Коммитить в неё нельзя. |
| **upstream** (отслеживаемая ветка) | Ветка удалённого репозитория, с которой связана локальная ветка. `git push` и `git pull` используют её. |
| **bare-репозиторий** (bare repository, «голый») | Репозиторий без рабочих файлов, только история. Серверы используют bare-репозитории. |
| **non-fast-forward** | Ситуация, когда в удалённом репозитории есть коммиты, которых у вас нет, поэтому ваш push отклоняется. |
| **URL** (Uniform Resource Locator, унифицированный указатель ресурса) | Адрес вроде `https://github.com/user/repo.git`. |
| **HTTPS** (HyperText Transfer Protocol Secure, защищённый протокол передачи гипертекста) | Протокол веб-страниц; Git может использовать его с токеном для входа. |
| **SSH** (Secure Shell, защищённая оболочка) | Протокол защищённого соединения с другим компьютером. Git может использовать его с ключом для входа. |
| **пара ключей** (key pair) | Два связанных файла: **закрытый ключ** (секретный, остаётся на вашем компьютере) и **открытый ключ** (им можно делиться). |
| **PAT** (Personal Access Token, персональный токен доступа) | Длинная случайная строка, работающая как пароль для одной цели; создаётся в настройках GitHub/GitLab. |
| **2FA** (Two-Factor Authentication, двухфакторная аутентификация) | Второе подтверждение личности (например, код из приложения) в дополнение к паролю. |
| **`gh`** | Официальный инструмент командной строки GitHub (GitHub CLI). |
| **CLI** (Command-Line Interface, интерфейс командной строки) | Способ работать с программой, вводя команды. |
| **README** | Текстовый файл (обычно `README.md`) в корне репозитория, объясняющий, что это за проект. |
| **лицензия** (licence) | Текстовый файл, который говорит, что другие люди могут делать с вашим кодом. |

## 1. Что такое удалённый репозиторий?

До сих пор вы работали в одной копии репозитория. **Удалённый репозиторий** (remote) — это другая копия на другом компьютере. С ним можно:

- хранить **резервную копию** своей работы вне ноутбука;
- **делиться** кодом с командой, которая делает push и pull в один и тот же удалённый репозиторий;
- запускать **автоматические проверки** при каждом push (следующий урок).

```text
   your computer                         server (GitHub, GitLab, a folder…)
 ┌────────────────┐    git push  ───▶   ┌─────────────────────┐
 │  your repo     │                      │  remote repository  │
 │  main          │    git fetch  ◀───   │  main               │
 │  origin/main   │    git pull   ◀───   │                     │
 └────────────────┘                      └─────────────────────┘
```

Поскольку Git распределённый, удалённый репозиторий ничем не особенный. Это просто ещё один репозиторий, о котором все договорились. Всё, что мы делаем ниже, работает с любым удалённым репозиторием.

## 2. Локальная лаборатория: «сервер» в папке

Прежде чем трогать GitHub, потренируйтесь с удалённым репозиторием, который живёт в папке на вашем компьютере. Он ведёт себя как настоящий сервер, и сломать вы ничего не можете.

Сначала создайте «сервер». Сервер хранит только историю, поэтому это **bare**-репозиторий. По соглашению имя его папки заканчивается на `.git`.

```bash
cd ~/projects/git-playground
mkdir remote-lab
cd remote-lab
git init --bare shared.git
```

```text
Initialized empty Git repository in /Users/ada/projects/git-playground/remote-lab/shared.git/
```

Затем создайте репозиторий Ады и подключите его к «серверу»:

```bash
git init ada
cd ada
echo "# Team notes" > README.md
git add README.md
git commit -m "Start team notes"
git remote add origin ../shared.git
git remote -v
git push -u origin main
```

```text
origin	../shared.git (fetch)
origin	../shared.git (push)
To ../shared.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

**Что происходит:**

1. `git remote add origin ../shared.git` даёт удалённому репозиторию короткое имя `origin` и адрес (здесь — относительный путь; потом это будет URL).
2. `git remote -v` перечисляет удалённые репозитории. У каждого есть адрес для получения и адрес для отправки.
3. `git push -u origin main` отправляет ветку `main` в `origin`. `-u` (`--set-upstream`) заодно связывает вашу `main` с `origin/main`, чтобы позже простые `git push` или `git pull` знали, куда идти. (Ваша настройка `push.autoSetupRemote` из [Урока 01.2](02-install-and-configure-git.md) делает эту связь автоматически при первом push *новой* ветки.)
4. `* [new branch] main -> main` значит: в удалённом репозитории `main` не было; теперь есть.

Проверьте состояние:

```bash
git branch -vv
git status
git log --oneline --decorate
```

```text
* main ebcd10d [origin/main] Start team notes
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
ebcd10d (HEAD -> main, origin/main) Start team notes
```

**Что происходит:** `[origin/main]` в `git branch -vv` показывает upstream ветки `main`. `origin/main` в логе — это **ветка удалённого отслеживания**: ваша пометка «`main` в `origin` находится на этом коммите». `up to date` значит, что ваша `main` и ваша пометка — один и тот же коммит.

## 3. Второй человек: `git clone`

Теперь подключается Боб. Он делает полную копию «сервера» командой `clone`:

```bash
cd ..
git clone shared.git bob
cd bob
git config user.name "Bob Builder"
git config user.email "bob@example.com"
git remote -v
git branch -a
```

```text
Cloning into 'bob'...
done.
origin	/Users/ada/projects/git-playground/remote-lab/shared.git (fetch)
origin	/Users/ada/projects/git-playground/remote-lab/shared.git (push)
* main
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
```

**Что происходит:**

1. `git clone <адрес> <папка>` создаёт папку, копирует **всю историю**, автоматически добавляет удалённый репозиторий `origin` и переключается на главную ветку.
2. `git config user.name …` без `--global` задаёт Бобу собственную личность только внутри этого репозитория (на вашем компьютере это воображаемый второй человек).
3. `git branch -a` перечисляет локальные ветки и ветки удалённого отслеживания (`remotes/origin/…`).

Боб добавляет коммит и отправляет его:

```bash
echo "Bob was here" >> README.md
git commit -am "Bob adds a line"
git push
```

```text
[main 3e99a64] Bob adds a line
 1 file changed, 1 insertion(+)
To /Users/ada/projects/git-playground/remote-lab/shared.git
   ebcd10d..3e99a64  main -> main
```

`ebcd10d..3e99a64` означает: `main` в удалённом репозитории передвинулась с первого хеша на второй.

## 4. В удалённом репозитории новости: `fetch`, `pull` и отклонённый push

Ада тем временем не знала о работе Боба. Она делает собственный коммит:

```bash
cd ../ada
echo "Ada notes" > ada.txt
git add ada.txt
git commit -m "Ada adds a file"
git push
```

```text
To ../shared.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to '../shared.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

**Что происходит:** это самая частая ошибка Git в командах. В удалённой `main` есть коммит (Боба), которого нет в `main` у Ады. Если бы Git принял её push, коммит Боба потерялся бы. Поэтому push **отклонён**. Ничего не повреждено. Правило: **сначала нужно принести новости удалённого репозитория к себе.** Сделайте это в два шага, чтобы посмотреть, прежде чем решать:

```bash
git fetch
git status
git log --oneline --graph --all --decorate
```

```text
From ../shared
   ebcd10d..3e99a64  main       -> origin/main
On branch main
Your branch and 'origin/main' have diverged,
and have 1 and 1 different commits each, respectively.
  (use "git pull" if you want to integrate the remote branch with yours)

nothing to commit, working tree clean
* 604011a (HEAD -> main) Ada adds a file
| * 3e99a64 (origin/main, origin/HEAD) Bob adds a line
|/
* ebcd10d Start team notes
```

**Что происходит:**

1. `git fetch` скачал коммит Боба и передвинул `origin/main`. Ваша собственная `main` и ваши файлы **не затронуты**. Поэтому `fetch` всегда безопасен.
2. `git status` сообщает, что ветки **разошлись** (diverged): по одному коммиту с каждой стороны. Граф показывает развилку.

Теперь объединяйте. `git pull` — сокращение для `git fetch` + `git merge`. С безопасной настройкой `pull.ff only` из Урока 01.2 pull, которому потребовалось бы слияние, **останавливается**:

```bash
git pull
```

```text
hint: Diverging branches can't be fast-forwarded, you need to either:
hint:
hint: 	git merge --no-ff
hint:
hint: or:
hint:
hint: 	git rebase
hint:
hint: Disable this message with "git config set advice.diverging false"
fatal: Not possible to fast-forward, aborting.
```

**Что происходит:** Git отказывается создавать коммит слияния у вас за спиной и называет два способа: merge или rebase. Пока выберите merge, явно:

```bash
git merge origin/main --no-edit
git log --oneline --graph --all --decorate
git push
git status
```

```text
Merge made by the 'ort' strategy.
 README.md | 1 +
 1 file changed, 1 insertion(+)
*   d87e070 (HEAD -> main) Merge remote-tracking branch 'origin/main'
|\
| * 3e99a64 (origin/main, origin/HEAD) Bob adds a line
* | 604011a Ada adds a file
|/
* ebcd10d Start team notes
To ../shared.git
   3e99a64..d87e070  main -> main
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

**Что происходит:** `git merge origin/main` слил ветку удалённого отслеживания в `main` (обычное слияние из [Урока 01.6](06-branches-and-merging.md); если бы менялись одни и те же строки, вы решали бы конфликт, как в [Уроке 01.7](07-merge-conflicts.md)). Теперь `main` у Ады содержит оба коммита, поэтому `git push` для сервера — fast-forward, и он принят.

Боб забирает результат:

```bash
cd ../bob
git pull
git log --oneline
```

```text
From /Users/ada/projects/git-playground/remote-lab/shared
   3e99a64..d87e070  main       -> origin/main
Updating 3e99a64..d87e070
Fast-forward
 ada.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 ada.txt
d87e070 Merge remote-tracking branch 'origin/main'
604011a Ada adds a file
3e99a64 Bob adds a line
ebcd10d Start team notes
```

Для Боба это обычный fast-forward: у него не было ничего нового, поэтому Git просто сдвинул его `main` вперёд.

### Ежедневная рутина

| Команда | Что делает | Меняет мою ветку? |
|---|---|---|
| `git clone <url>` | копирует весь репозиторий | создаёт его |
| `git remote -v` | показывает удалённые репозитории и их адреса | нет |
| `git fetch` | скачивает новости, двигает `origin/*` | **нет** |
| `git pull` | `fetch` + слияние (с нашей настройкой: только fast-forward) | да |
| `git push` | отправляет мои коммиты в upstream | нет (меняет удалённый репозиторий) |
| `git push -u origin <branch>` | первый push ветки, задаёт upstream | нет |

> 💡 **Совет:** привычка хорошего участника команды: **`git pull` перед началом работы, `git pull` перед push.** Небольшая и частая интеграция означает небольшие конфликты.

> ⚠️ **Частая ошибка:** «исправлять» отклонённый push командой `git push --force`. Она перезаписывает историю удалённого репозитория и уничтожает чужие коммиты. Никогда не делайте force-push в общую ветку вроде `main`. (Осторожный вариант, `--force-with-lease`, для **ваших собственных** веток вы увидите в [Уроке 01.9](09-teamwork-pull-requests-and-conventions.md).)

## 5. GitHub и GitLab

Сервис хостинга даёт вам «сервер», чтобы вы не запускали его сами, и добавляет:

- **веб-страницу** вашего репозитория (файлы, история, diff);
- **управление доступом** (приватные репозитории, команды);
- **pull request** (GitHub) / **merge request** (GitLab) для ревью ([Урок 01.9](09-teamwork-pull-requests-and-conventions.md));
- **задачи** (issues, отслеживание работы);
- **CI** (Continuous Integration, непрерывная интеграция): автоматические сборки и тесты при каждом push.

| | GitHub | GitLab |
|---|---|---|
| Адрес | `github.com` | `gitlab.com` (или собственный сервер вашей компании) |
| Средство ревью | pull request (PR) | merge request (MR) |
| Автоматические проверки | GitHub Actions (`.github/workflows/*.yml`) | GitLab CI (`.gitlab-ci.yml`) |
| Инструмент командной строки | `gh` | `glab` |

Они очень похожи. Хорошо выучите один, и на второй уйдёт час. Сам проект Rust и большинство крейтов живут на GitHub; многие компании используют GitLab, часто на собственных серверах.

### Создайте аккаунт

1. Откройте github.com и выберите *Sign up*. Используйте надёжный уникальный пароль в менеджере паролей.
2. Включите **2FA** (*Settings → Password and authentication*). Аккаунты без неё легче всего украсть. Храните коды восстановления в безопасности.
3. В *Settings → Emails* выберите «Keep my email addresses private» и используйте адрес `…@users.noreply.github.com` как `user.email` в Git ([Урок 01.2](02-install-and-configure-git.md)), если хотите не светить свой адрес в публичных коммитах.
4. По желанию: то же самое на gitlab.com.

Создайте аккаунт сами и никогда не сообщайте пароль или токены никому, включая ИИ-ассистента (ИИ — искусственный интеллект).

## 6. Вход: SSH-ключи

Чтобы отправлять в приватный репозиторий, сервер должен знать, кто вы. Сегодня GitHub **не** принимает пароль от аккаунта для Git по HTTPS. Есть два хороших способа:

| Способ | Как работает | Подходит для |
|---|---|---|
| **SSH-ключ** | У вас закрытый ключ; сервер знает ваш открытый ключ. | ноутбук, которым вы пользуетесь каждый день (рекомендуется здесь) |
| **HTTPS + токен (PAT)** | Вы вставляете токен вместо пароля; помощник его хранит. | места, где SSH заблокирован; скрипты и CI |

### Как работает пара ключей

Вы создаёте **два** файла. **Закрытый ключ** доказывает, что вы — это вы. Он никогда не покидает ваш компьютер, и никто другой не должен его видеть. **Открытый ключ** — подходящий к нему замок; вы отдаёте его GitHub. При подключении сервер с помощью открытого ключа проверяет, что у вас есть соответствующий закрытый, при этом закрытый ключ никогда не передаётся.

### Создайте ключ

```bash
ssh-keygen -t ed25519 -C "ada@example.com"
```

`ssh-keygen` задаёт три вопроса. Нажмите `Enter`, чтобы принять файл по умолчанию `~/.ssh/id_ed25519`, затем дважды введите **парольную фразу** (passphrase, пароль, защищающий файл ключа). Используйте её: если кто-то скопирует диск вашего ноутбука, парольная фраза всё равно защитит ключ.

```text
Generating public/private ed25519 key pair.
Your identification has been saved in /Users/ada/.ssh/id_ed25519
Your public key has been saved in /Users/ada/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:g15nmXfDJKWmpV/+1ncMgfDW00MB/CZXbSSrnCRvFKo ada@example.com
```

**Что происходит:**

1. `-t ed25519` выбирает современный, короткий и стойкий тип ключа (Ed25519 — название алгоритма; прежний популярный тип — RSA, названный по первым буквам фамилий изобретателей Rivest, Shamir и Adleman).
2. `-C "…"` добавляет комментарий, метку, чтобы вы помнили, для чего ключ.
3. В `~/.ssh/` появляются два файла: `id_ed25519` (**закрытый**, не показывать никому) и `id_ed25519.pub` (**открытый**, им можно делиться).
4. Отпечаток (fingerprint) — короткий хеш открытого ключа; по нему можно опознать ключ.

> 🔒 **Безопасность:** файл закрытого ключа — `id_ed25519` (без `.pub`). Никогда не отправляйте его по почте, не вставляйте на сайт или в чат, не коммитьте в Git. Вставляйте только файл `.pub`.

Если ключ в `~/.ssh/` у вас уже есть, не перезаписывайте его: `ssh-keygen` спросит перед перезаписью. Ответьте `n` и выберите другое имя файла.

### Пусть Mac запомнит парольную фразу

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

```text
Identity added: /Users/ada/.ssh/id_ed25519 (ada@example.com)
```

Это сохраняет ключ в `ssh-agent` (вспомогательной программе, которая хранит разблокированные ключи), а парольную фразу — в связке ключей macOS (Keychain), так что вводить парольную фразу придётся только один раз. Чтобы это происходило автоматически после каждой перезагрузки, создайте или отредактируйте файл `~/.ssh/config` и добавьте:

```text
Host github.com gitlab.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

**Что происходит:** для подключений к этим двум хостам SSH добавляет ключ в агент, использует Keychain и берёт этот файл ключа.

### Дайте GitHub открытый ключ

1. Скопируйте **открытый** ключ в буфер обмена: `pbcopy < ~/.ssh/id_ed25519.pub` (`pbcopy` — программа macOS, копирующая свой ввод в буфер обмена).
2. На GitHub откройте *Settings → SSH and GPG keys → New SSH key* (GPG, GNU Privacy Guard, — другой вид ключей, который сейчас не нужен), задайте название (например, «MacBook»), выберите тип «Authentication key» и вставьте ключ.
3. То же самое сделайте на GitLab (*Preferences → SSH Keys*), если вы им пользуетесь.

Открытый ключ — одна строка:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBVfFM8Sx0U7SRhFYumqY73KRYDx0ihD26lO5+PHowNU ada@example.com
```

### Проверьте соединение

```bash
ssh -T git@github.com
```

В первый раз SSH спросит `Are you sure you want to continue connecting (yes/no/[fingerprint])?` Сравните отпечаток с опубликованным в документации GitHub и введите `yes`. После этого ответ должен выглядеть так:

```text
Hi your-username! You've successfully authenticated, but GitHub does not provide shell access.
```

(GitLab отвечает `Welcome to GitLab, @your-username!`.) Если вы не добавили ключ (или ключ не предлагается), вы получите самую частую ошибку SSH:

```text
git@github.com: Permission denied (publickey).
```

Это значит: сервер не узнал ни один из ваших ключей. Проверьте, что вы вставили **открытый** ключ в **свой** аккаунт, что ключ загружен (`ssh-add -l`) и что адрес удалённого репозитория начинается с `git@github.com:`, а не с `https://`.

## 7. Подключите свой репозиторий к GitHub

1. На GitHub выберите *+ → New repository*. Задайте имя (например, `git-practice`). Выберите **Private**. **Не** отмечайте «Add a README», «.gitignore» или «license», если будете отправлять существующий локальный репозиторий, потому что новый репозиторий должен быть пустым.
2. GitHub покажет адреса. Скопируйте **SSH**-адрес. Он выглядит как `git@github.com:your-username/git-practice.git`.
3. В локальном репозитории выполните:

```bash
git remote add origin git@github.com:your-username/git-practice.git
git push -u origin main
```

(Замените `your-username` и имя репозитория своими.) Обновите веб-страницу: ваши файлы и история уже онлайн. Чтобы получить репозиторий с GitHub на новый компьютер:

```bash
git clone git@github.com:your-username/git-practice.git
```

Два стиля адреса:

| Стиль | Пример | Вход |
|---|---|---|
| SSH | `git@github.com:user/repo.git` | ваш SSH-ключ |
| HTTPS | `https://github.com/user/repo.git` | токен (PAT) |

Адрес можно изменить позже: `git remote set-url origin <новый-адрес>`.

### GitHub CLI (по желанию)

`gh` выполняет шаги с сайта из терминала:

```bash
brew install gh
gh auth login
gh repo create git-practice --private --source=. --remote=origin --push
```

`gh auth login` выполняет вход через браузер. `gh repo create … --source=. --push` создаёт приватный репозиторий из текущей папки, добавляет `origin` и отправляет ваши коммиты. `gh` тоже безопасно хранит свой токен в системной связке ключей. Это удобство: всё, что он делает, вы можете сделать через `git` и сайт.

## 8. README, лицензия и файл игнорирования

Репозиторий на GitHub должен открываться с **README**. Напишите в нём: что за проект, как его собрать и запустить (`cargo run`, `cargo test`) и как участвовать. Файл **лицензии** (`LICENSE`) говорит другим, что они могут делать с вашим кодом; без лицензии они юридически не вправе использовать его повторно. Проекты на Rust часто используют двойную лицензию «MIT OR Apache-2.0» (MIT, названная по Массачусетскому технологическому институту, — короткая разрешающая лицензия; Apache 2.0 — другая). Для приватного учебного репозитория она не нужна.

## 9. Не пускайте секреты в удалённый репозиторий

Удалённый репозиторий — общий, а публичные репозитории читают роботы, которые ищут секреты уже через минуты.

- **Никогда не коммитьте** пароли, ключи API (Application Programming Interface, программный интерфейс приложения), токены, файлы `.env` или закрытые ключи. Добавляйте их в `.gitignore` до первого коммита.
- Если секрет **был** отправлен, считайте его украденным: **отзовите его** в сервисе (удалите токен, замените ключ) и создайте новый. Удаление файла в новом коммите не помогает ([Урок 01.5](05-undoing-changes-safely.md)). GitHub также сканирует push на известные форматы токенов и может предупредить вас.
- Для всего незаконченного предпочитайте **приватный** репозиторий.
- Включите 2FA и пользуйтесь менеджером паролей.
- Парольная фраза на SSH-ключе — дешёвая страховка.

> 🔒 **Безопасность:** персональный токен доступа так же могуществен, как пароль, в пределах выданных ему прав. Давайте ему минимальные права и короткий срок действия. Никогда не вставляйте его в чат и не коммитьте.

## Частые ошибки

> ⚠️ **Частая ошибка:** `error: remote origin already exists.` Вы выполнили `git remote add origin …` дважды. Измените адрес: `git remote set-url origin <адрес>`, или удалите его: `git remote remove origin`.

> ⚠️ **Частая ошибка:** `fatal: 'nowhere' does not appear to be a git repository`, а следом `Could not read from remote repository`. Имя или адрес удалённого репозитория неверны. Проверьте `git remote -v`.

> ⚠️ **Частая ошибка:** `! [rejected] main -> main (fetch first)` или `(non-fast-forward)`. В удалённом репозитории есть коммиты, которых у вас нет. `git fetch`, слияние (или `git pull`), затем снова push. Никогда `--force` для общей ветки.

> ⚠️ **Частая ошибка:** `Permission denied (publickey)`. Ключа нет в вашем аккаунте, он не загружен или вы использовали неверный адрес. Выполните `ssh -T git@github.com`, чтобы проверить один только ключ.

> ⚠️ **Частая ошибка:** HTTPS запрашивает пароль, а затем завершается ошибкой аутентификации. GitHub не принимает пароли от аккаунтов для Git. Используйте SSH или токен в качестве пароля.

> ⚠️ **Частая ошибка:** создать репозиторий на GitHub с README и затем пытаться отправить в него свою историю. Две истории не связаны, и Git отказывается (`refusing to merge unrelated histories`). Создавайте удалённый репозиторий **пустым**.

> ⚠️ **Частая ошибка:** коммитить под одной личностью, а входить под другой. Автор коммита берётся из `user.email`, а не из вашего логина. Задайте `user.email` как адрес (или noreply-адрес) вашего аккаунта.

## Попробуйте сами

Часть A — работает без сети:

1. Повторите лабораторную из разделов 2–4 с начала в новой папке `remote-lab2`.
2. В `ada` сделайте второй коммит и отправьте его; в `bob` выполните `git fetch`, а затем `git status`: что он говорит?
3. В `bob` выполните `git pull`. Что говорит вывод? Почему это fast-forward?
4. Выполните `git branch -vv` в обеих папках.

Часть B — нужен аккаунт GitHub:

5. Создайте аккаунт, включите 2FA.
6. Создайте SSH-ключ (с парольной фразой), добавьте его в `ssh-agent`, добавьте открытый ключ в GitHub и проверьте командой `ssh -T git@github.com`.
7. Создайте **пустой приватный** репозиторий `git-practice`, подключите к нему `hello-rust` или `branch-demo` и отправьте.
8. Посмотрите страницу репозитория: коммиты, файлы, число веток.

## Упражнения

### Упражнение 1 — Bare-репозиторий и клон (легко)

Создайте bare-репозиторий `team.git`, склонируйте его как `me`, сделайте коммит и отправьте. Докажите, что коммит есть в bare-репозитории, не клонируя заново.

<details>
<summary>Подсказка</summary>

`git init --bare`, `git clone`, затем `git -C team.git log --oneline main` показывает лог другой папки.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/git-playground/remote-lab
git init --bare team.git
git clone team.git me
cd me
echo "hello" > hello.txt
git add hello.txt
git commit -m "Add hello"
git push -u origin main
git -C ../team.git log --oneline main
```

Git предупреждает `You appear to have cloned an empty repository.`, а последняя команда печатает коммит `Add hello` (с вашим хешем).

Почему это работает: клонирование пустого репозитория даёт пустой локальный репозиторий с уже заданным `origin` и `main` в качестве первой ветки (благодаря `init.defaultBranch`); первый push создаёт `main` в удалённом репозитории. `git -C <папка>` запускает Git так, будто вы находитесь в этой папке.

</details>

### Упражнение 2 — Прочитайте строку статуса (легко)

`git status` печатает `Your branch is ahead of 'origin/main' by 2 commits.` Что произошло и какая команда следующая? А если написано `behind … by 1 commit`?

<details>
<summary>Подсказка</summary>

Ahead («впереди») = у вас есть коммиты, которых нет в удалённом репозитории. Behind («позади») = в удалённом репозитории есть коммиты, которых нет у вас.

</details>

<details>
<summary>Ответ</summary>

*Ahead by 2:* вы сделали два коммита, которых ещё нет в удалённом репозитории. Выполните `git push`.

*Behind by 1:* кто-то отправил один коммит, которого у вас нет. Выполните `git pull` (он сделает fast-forward).

Если написано *diverged*, у обеих сторон есть новые коммиты: fetch, слияние `origin/main`, затем push.

Почему это работает: Git сравнивает вашу ветку с веткой удалённого отслеживания `origin/main` (состояние на момент последнего fetch). Сначала выполните `git fetch`, чтобы эти сведения были свежими.

</details>

### Упражнение 3 — Двое, один файл (средне)

Используя `shared.git`, `ada` и `bob` из лаборатории: Боб меняет `README.md` и отправляет. Ада меняет **другой** файл и пытается отправить. Покажите отклонение, затем объедините через `fetch` + `merge` и успешно отправьте.

<details>
<summary>Подсказка</summary>

Это в точности раздел 4. После `git fetch` слейте `origin/main`.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/git-playground/remote-lab/bob
echo "Bob again" >> README.md
git commit -am "Bob edits README"
git push
cd ../ada
echo "more notes" >> ada.txt
git commit -am "Ada edits ada.txt"
git push
git fetch
git merge origin/main --no-edit
git push
```

Первый `git push` Ады печатает `! [rejected] main -> main (fetch first)`; последний проходит успешно.

Почему это работает: push Ады отбросил бы коммит Боба, поэтому сервер его отклоняет. После слияния ветка Ады содержит коммит Боба, поэтому push лишь добавляет к истории удалённого репозитория.

</details>

### Упражнение 4 — Сделайте одноразовый SSH-ключ (средне)

Сгенерируйте ключ Ed25519 во **временном** месте (не в `~/.ssh`) с пустой парольной фразой — только для этого упражнения. Выведите его отпечаток. Затем удалите файлы ключа.

<details>
<summary>Подсказка</summary>

Используйте `-f` для пути к файлу и `-N ""` для пустой парольной фразы. `ssh-keygen -lf <file>.pub` печатает отпечаток.

</details>

<details>
<summary>Ответ</summary>

```bash
ssh-keygen -t ed25519 -C "exercise" -f ~/projects/git-playground/test_key -N ""
ssh-keygen -lf ~/projects/git-playground/test_key.pub
rm ~/projects/git-playground/test_key ~/projects/git-playground/test_key.pub
```

```text
256 SHA256:7ygxdFjIz5wiG+h4R5nVMLglbfHnwX10S2l8ZsYq90E exercise (ED25519)
```

(Ваш отпечаток будет другим: ключ случайный.)

Почему это работает: `-f` выбирает выходной файл, а `-N ""` задаёт пустую парольную фразу, что нормально для ключа, который вы сразу выбросите. Для настоящего ключа всегда используйте парольную фразу и храните его в `~/.ssh`.

</details>

### Упражнение 5 — Исправьте ошибочный удалённый репозиторий (сложно)

В папке `ada` лаборатории сначала создайте ошибку: `git remote set-url origin git@github.com:wrong/repo.git`. Теперь покажите, как увидеть неверный адрес, заменить его на `../shared.git` и доказать, что `git push` работает. Также скажите, что делать, если вы хотели полностью удалить удалённый репозиторий.

<details>
<summary>Подсказка</summary>

`git remote -v`, `git remote set-url`, `git remote remove`.

</details>

<details>
<summary>Ответ</summary>

```bash
git remote -v
git remote set-url origin ../shared.git
git remote -v
git push
```

```text
origin	git@github.com:wrong/repo.git (fetch)
origin	git@github.com:wrong/repo.git (push)
origin	../shared.git (fetch)
origin	../shared.git (push)
```

Чтобы полностью удалить удалённый репозиторий: `git remote remove origin` (удаляется имя и пометки `origin/*`, но не сам удалённый репозиторий).

Почему это работает: удалённый репозиторий — это только имя и адрес в `.git/config`. `set-url` меняет адрес; ваши коммиты не затрагиваются.

</details>

## Вопросы для самопроверки

1. Чем отличаются `git fetch` и `git pull`?

<details>
<summary>Ответ</summary>

`fetch` только скачивает новости и обновляет `origin/*`; он не трогает ваши ветки и файлы. `pull` — это `fetch` с последующим слиянием (с нашей настройкой — только fast-forward) в текущую ветку.

</details>

2. Что означает `origin/main`?

<details>
<summary>Ответ</summary>

Ветка удалённого отслеживания: ваша локальная пометка о том, где была `main` в `origin` при последнем fetch, push или pull.

</details>

3. Почему первый push Ады был отклонён?

<details>
<summary>Ответ</summary>

В удалённой `main` был коммит (Боба), которого у Ады не было. Принятие её push потеряло бы его. Сначала ей нужно получить и объединить.

</details>

4. Почему никогда нельзя делать `git push --force` в `main`?

<details>
<summary>Ответ</summary>

Она перезаписывает историю удалённого репозитория и уничтожает чужие коммиты.

</details>

5. Какой ключ вы отдаёте GitHub, а какой не должен покидать ваш компьютер?

<details>
<summary>Ответ</summary>

Отдаёте **открытый** ключ (`id_ed25519.pub`). **Закрытый** ключ (`id_ed25519`) не должен покидать ваш компьютер.

</details>

6. Что обычно означает `Permission denied (publickey)`?

<details>
<summary>Ответ</summary>

Сервер не принял ни один из ваших ключей: открытый ключ не добавлен в ваш аккаунт, ключ не загружен в агент или вы используете неверный адрес или пользователя.

</details>

7. Вы закоммитили токен и отправили его. Что делать первым делом?

<details>
<summary>Ответ</summary>

Отозвать токен в сервисе и создать новый. Затем убрать его из кода и добавить правила в `.gitignore`. Удаления в более позднем коммите недостаточно.

</details>

## Итоги

- Удалённый репозиторий — другая копия репозитория; `origin` — имя по умолчанию. `git clone` делает полную копию.
- `git push` отправляет ваши коммиты, `git fetch` скачивает, не трогая вашу работу, `git pull` — это fetch + слияние.
- `origin/main` — ветка удалённого отслеживания: ваша пометка об удалённом репозитории.
- Отклонённый push означает, что в удалённом репозитории есть коммиты, которых у вас нет: fetch, слияние, push. Никогда не делайте force-push в общую ветку.
- Всё можно отработать с bare-репозиторием в папке.
- GitHub и GitLab хостят удалённые репозитории и добавляют ревью, задачи и CI.
- Входите с помощью SSH-ключа (открытый ключ — сервису, закрытый остаётся у вас) и используйте 2FA.
- Никогда не коммитьте секреты; если секрет утёк, отзовите его.

## Источники и что почитать

- [Pro Git — 2.5 Working with Remotes](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes) — официальные команды для удалённых репозиториев.
- [Pro Git — 3.5 Remote Branches](https://git-scm.com/book/en/v2/Git-Branching-Remote-Branches) — ветки удалённого отслеживания и upstream.
- [GitHub Docs — Connecting to GitHub with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh) — создание ключей, добавление, проверка.
- [GitHub Docs — Securing your account with two-factor authentication](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa) — включение 2FA.
- [GitLab Docs — Use SSH keys to communicate with GitLab](https://docs.gitlab.com/user/ssh/) — то же самое для GitLab.
- [GitHub CLI manual](https://cli.github.com/manual/) — все команды `gh`.

---

[🏠 Начало](../START-HERE.md) · [← 01.7 Конфликты слияния](07-merge-conflicts.md) · [📋 Этап 01](README.md) · [01.9 Командная работа: pull request, код-ревью и соглашения →](09-teamwork-pull-requests-and-conventions.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
