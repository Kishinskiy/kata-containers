# Установка

## Helm Chart

[Helm](https://helm.sh/docs/intro/install/) можно использовать для установки шаблонов манифестов Kubernetes.

### Предварительные требования

- **Kubernetes ≥ v1.22** – v1.22 является первым релизом, где CRI v1 API стал стандартным, а `RuntimeClass` вышел из альфа-версии. Chart зависит от этих стабильных интерфейсов; более ранним кластерам требуются `feature‑gates` или CRI-адаптеры, которые выходят за рамки данной документации.

- **Kata Release 3.12** - в v3.12.0 началась публикация helm-chart на странице релизов для упрощения использования. Начиная с v3.8.0, мы поставляем helm-chart через исходный код в репозитории `GitHub` kata-containers.

- CRI-совместимая среда выполнения (containerd или CRI-O). Для использования функции `multiInstallSuffix` требуется как минимум **containerd-2.0**, который поддерживает drop-in конфигурационные файлы.

- Узлы должны разрешать загрузку модулей ядра и установку артефактов Kata (chart запускает привилегированные контейнеры для этого).

### `helm install`

```sh
# Установка напрямую из официального OCI-реестра ghcr.io
# обновите VERSION X.YY.Z по необходимости или используйте последнюю версию

export VERSION=$(curl -sSL https://api.github.com/repos/kata-containers/kata-containers/releases/latest | jq .tag_name | tr -d '"')
export CHART="oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy"

$ helm install kata-deploy "${CHART}" --version "${VERSION}"

# Посмотрите все доступные настройки
$ helm show values "${CHART}" --version "${VERSION}"
```

Это устанавливает DaemonSet `kata-deploy` и ресурсы `RuntimeClass` Kata по умолчанию в вашем кластере.

Чтобы посмотреть, какие версии chart доступны:

```sh
$ helm show chart oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy
```

### `helm uninstall`

```sh
$ helm uninstall kata-deploy -n kube-system
```

Во время удаления Helm сообщит, что некоторые ресурсы были сохранены из-за политики ресурсов (`ServiceAccount`, `ClusterRole`, `ClusterRoleBinding`). Это **нормально**. Задача (Job) post-delete hook запускается после удаления и убирает эти ресурсы, чтобы в кластере не осталось общесистемных правил `RBAC`.

## Предварительно собранные релизы

Kata также можно установить с использованием предварительно собранных релизов: https://github.com/kata-containers/kata-containers/releases

Этот метод не предоставляет никаких средств для управления жизненным циклом артефактов.