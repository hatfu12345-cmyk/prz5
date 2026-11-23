# Практика №5: syspulse → .deb → Nexus (APT) → apt install

## Структура репозитория

```
.
├── .github/workflows/publish-deb.yml   # пайплайн build → package → publish → verify
├── ci/syspulse-archive-keyring.gpg     # ПУБЛИЧНЫЙ GPG-ключ репозитория (создать, см. шаг 2)
├── go.mod
└── main.go
```

## 1. Запуск Nexus

```bash
docker network create ci
docker run -d --name nexus --network ci -p 8081:8081 -v nexus-data:/nexus-data sonatype/nexus3:3
# ждать 1–3 мин, пока не начнёт открываться http://<host>:8081
docker exec nexus cat /nexus-data/admin.password     # стартовый пароль admin
```

В мастере первого входа задайте новый пароль и **включите Anonymous access**.
Без анонимного чтения `apt-get update` в контейнере получит 401.

## 2. GPG-ключ для подписи репозитория

Без ключа Nexus не даст создать APT-репозиторий: поле Signing Key обязательное.

```bash
export GPG_PASS='СложныйПароль'
gpg --batch --pinentry-mode loopback --passphrase "$GPG_PASS" \
    --quick-gen-key "syspulse repo <devops@example.com>" rsa4096 sign never

# приватный ключ: вставить в Nexus, НЕ коммитить
gpg --armor --pinentry-mode loopback --passphrase "$GPG_PASS" \
    --export-secret-keys devops@example.com > nexus-private.asc

# публичный ключ: коммитится в репозиторий, нужен apt в контейнере
mkdir -p ci && gpg --export devops@example.com > ci/syspulse-archive-keyring.gpg
```

## 3. Hosted APT-репозиторий

Administration → Repositories → Create repository → **apt (hosted)**:

| Поле | Значение |
|---|---|
| Name | `syspulse` |
| Distribution | `stable` |
| Signing Key | всё содержимое `nexus-private.asc` |
| Passphrase | `$GPG_PASS` |
| Deployment policy | **Allow redeploy** (чтобы повторные запуски проходили) |

URL для apt: `http://<host>:8081/repository/syspulse/`

## 4. Пользователь для деплоя

1. Security → Roles → Create role: `syspulse-deployer`, привилегия `nx-repository-view-apt-syspulse-*`.
2. Security → Users → Create local user: `deployer`, роль `syspulse-deployer`.

## 5. Секреты GitHub

Settings → Secrets and variables → Actions:
`NEXUS_HOST` (`localhost`, если Nexus и раннер на одной машине), `NEXUS_USER`, `NEXUS_PASSWORD`.

## 6. Требования к self-hosted раннеру

На машине с раннером должны быть `dpkg-deb`, `curl`, `file`, `docker`,
а пользователь раннера должен состоять в группе `docker`.

## 7. Запуск

```bash
git checkout -b release
git add . && git commit -m "Practice 5: deb pipeline"
git push -u origin release
```

Для проверки идемпотентности нажмите Actions → Re-run all jobs, второй запуск тоже должен стать зелёным.

## Что приложить к отчёту

- лог шага Build со строкой `file`: `statically linked`
- лог Package: `dpkg-deb --info` и `--contents`
- лог Publish: HTTP 204, строка `Package: syspulse` из Packages, начало InRelease (`-----BEGIN PGP SIGNED MESSAGE-----`)
- лог Verify: `apt-get update` без ошибок, `syspulse 1.2.0`
- скриншот Nexus: Browse → syspulse
- два успешных запуска подряд
- артефакт `syspulse-deb`
