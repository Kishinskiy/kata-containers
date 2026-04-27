# Конфигурация Helm

## Параметры

Helm chart предоставляет всеобъемлющий набор опций конфигурации. Вы можете просмотреть параметры и их описания, перейдя в [исходный код на GitHub](https://github.com/kata-containers/kata-containers/blob/main/tools/packaging/kata-deploy/helm-chart/kata-deploy/values.yaml) или с помощью helm:

```sh
# Список доступных версий chart kata-deploy:
#   helm search repo kata-deploy-charts/kata-deploy --versions
#
# Затем замените X.Y.Z ниже на желаемую версию chart:
helm show values --version X.Y.Z oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy
```

### shims

Kata поставляется с рядом предварительно собранных артефактов и сред выполнения. Вы можете выборочно включать или отключать конкретные shims. Например:

```yaml title="values.yaml"
shims:
  disableAll: true
  qemu:
    enabled: true
  qemu-nvidia-gpu:
    enabled: true
  qemu-nvidia-gpu-snp:
    enabled: false

```

Shims также могут иметь специфичные для них опции конфигурации:

```yaml
  qemu-nvidia-gpu:
    enabled: ~
    supportedArches:
      - amd64
    allowedHypervisorAnnotations: []
    containerd:
      snapshotter: ""
    runtimeClass:
      # Эта метка автоматически добавляется gpu-operator. Переопределите её,
      # если хотите использовать другую метку.
      # Раскомментируйте после выхода GPU Operator v26.3
      # nodeSelector:
        # nvidia.com/cc.ready.state: "false"
```

Лучше всего обратиться к файлу `values.yaml` по умолчанию (указанному выше) для получения более подробной информации.

### Пользовательские среды выполнения (Custom Runtimes)

Kata позволяет создавать пользовательские конфигурации сред выполнения. Это делается путем наложения пользовательских конфигураций поверх одной из существующих конфигураций runtime. Например, мы можем использовать `qemu-nvidia-gpu` в качестве базовой конфигурации и добавить к ней наши собственные параметры:

```yaml
customRuntimes:
  enabled: false
  runtimes:
    my-gpu-runtime:
      baseConfig: "qemu-nvidia-gpu"  # Обязательно: существующая конфигурация для использования в качестве основы
      dropIn: |                      # Опционально: переопределения через механизм config.d
        [hypervisor.qemu]
        default_memory = 1024
        default_vcpus = 4
      runtimeClass: |
        kind: RuntimeClass
        apiVersion: node.k8s.io/v1
        metadata:
          name: kata-my-gpu-runtime
          labels:
            app.kubernetes.io/managed-by: kata-deploy
        handler: kata-my-gpu-runtime
        overhead:
          podFixed:
            memory: "640Mi"
            cpu: "500m"
        scheduling:
          nodeSelector:
            katacontainers.io/kata-runtime: "true"
      # Опционально: специфичная для CRI конфигурация
      containerd:
        snapshotter: "nydus"  # Настройка снапшоттера containerd (nydus, erofs, и т.д.)
      crio:
        pullType: "guest-pull"  # Настройка CRI-O runtime_pull_image = true
```

Снова обратитесь к файлу [`values.yaml`](#parameters) по умолчанию для получения более подробной информации.

## Примеры

Мы предоставляем несколько примеров, которые вы можете передать в helm с помощью флага `-f`/`--values`.

### [`try-kata-tee.values.yaml`](https://github.com/kata-containers/kata-containers/blob/main/tools/packaging/kata-deploy/helm-chart/kata-deploy/try-kata-tee.values.yaml)

Этот файл включает только shims TEE (Trusted Execution Environment) для конфиденциальных вычислений:

```sh
helm install kata-deploy oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy \
  --version VERSION \
  -f try-kata-tee.values.yaml
```

Включает:

- `qemu-snp` - AMD SEV-SNP (amd64)
- `qemu-tdx` - Intel TDX (amd64)
- `qemu-se` - IBM Secure Execution for Linux (SEL) (s390x)
- `qemu-se-runtime-rs` - IBM Secure Execution for Linux (SEL) Rust runtime (s390x)
- `qemu-coco-dev` - Confidential Containers development (amd64, s390x)
- `qemu-coco-dev-runtime-rs` - Confidential Containers development Rust runtime (amd64, arm64, s390x)

### [`try-kata-nvidia-gpu.values.yaml`](https://github.com/kata-containers/kata-containers/blob/main/tools/packaging/kata-deploy/helm-chart/kata-deploy/try-kata-nvidia-gpu.values.yaml)

Этот файл включает только shims с поддержкой NVIDIA GPU:

```sh
helm install kata-deploy oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy \
  --version VERSION \
  -f try-kata-nvidia-gpu.values.yaml
```

Включает:

- `qemu-nvidia-gpu` - Стандартная поддержка NVIDIA GPU (amd64)
- `qemu-nvidia-gpu-snp` - NVIDIA GPU с AMD SEV-SNP (amd64)
- `qemu-nvidia-gpu-tdx` - NVIDIA GPU с Intel TDX (amd64)

### `nodeSelector`

Мы можем развернуть Kata только на определенных узлах, используя `nodeSelector`

```sh
# Сначала промаркируйте узлы, где вы хотите установить kata-containers
$ kubectl label nodes worker-node-1 kata-containers=enabled
$ kubectl label nodes worker-node-2 kata-containers=enabled

# Затем установите chart с `nodeSelector`
$ helm install kata-deploy \
  --set nodeSelector.kata-containers="enabled" \
  "${CHART}" --version  "${VERSION}"
```

Вы также можете использовать файл values:

```yaml title="values.yaml"
nodeSelector:
  kata-containers: "enabled"
  node-type: "worker"
```

```sh
$ helm install kata-deploy -f values.yaml "${CHART}" --version "${VERSION}"
```

### Множественные установки Kata на одном узле

Для отладки, тестирования и других сценариев использования возможно развернуть несколько версий Kata на одном и том же узле. Все необходимые артефакты получают суффикс `multiInstallSuffix`, чтобы различать каждую установку. **ВНИМАНИЕ**: для этого требуется как минимум **containerd-2.0**, поскольку эта версия имеет поддержку drop-in конфигураций, что является обязательным условием для корректной работы `multiInstallSuffix`.

```sh
$ helm install kata-deploy-cicd       \
  -n kata-deploy-cicd                 \
  --set env.multiInstallSuffix=cicd   \
  --set env.debug=true                \
  "${CHART}" --version  "${VERSION}"
```

Примечание: `runtimeClasses` создаются автоматически Helm (через
      `runtimeClasses.enabled=true`, что является значением по умолчанию).

Теперь проверьте установку, изучив `runtimeClasses`:

```sh
$ kubectl get runtimeClasses
NAME                            HANDLER                         AGE
kata-clh-cicd                   kata-clh-cicd                   77s
kata-cloud-hypervisor-cicd      kata-cloud-hypervisor-cicd      77s
kata-dragonball-cicd            kata-dragonball-cicd            77s
kata-fc-cicd                    kata-fc-cicd                    77s
kata-qemu-cicd                  kata-qemu-cicd                  77s
kata-qemu-coco-dev-cicd         kata-qemu-coco-dev-cicd         77s
kata-qemu-nvidia-gpu-cicd       kata-qemu-nvidia-gpu-cicd       77s
kata-qemu-nvidia-gpu-snp-cicd   kata-qemu-nvidia-gpu-snp-cicd   77s
kata-qemu-nvidia-gpu-tdx-cicd   kata-qemu-nvidia-gpu-tdx-cicd   76s
kata-qemu-runtime-rs-cicd       kata-qemu-runtime-rs-cicd       77s
kata-qemu-se-runtime-rs-cicd    kata-qemu-se-runtime-rs-cicd    77s
kata-qemu-snp-cicd              kata-qemu-snp-cicd              77s
kata-qemu-tdx-cicd              kata-qemu-tdx-cicd              77s
kata-stratovirt-cicd            kata-stratovirt-cicd            77s
```

## Селекторы узлов RuntimeClass для TEE Shims

**Ручная конфигурация:** Любой `nodeSelector`, который вы устанавливаете под `shims.<shim>.runtimeClass.nodeSelector`, **всегда применяется** к RuntimeClass этого shim, независимо от наличия NFD. Используйте это, когда вы хотите привязать рабочие нагрузки TEE к определенным узлам (например, без NFD или с пользовательскими метками).

**Автоматическое добавление при наличии NFD:** Если вы *не* устанавливаете `runtimeClass.nodeSelector` для TEE shim, chart может **автоматически добавлять** метки на основе NFD, когда NFD обнаружен в кластере (развернут этим chart с `node-feature-discovery.enabled=true` или найден внешний):

- AMD SEV-SNP shims: `amd.feature.node.kubernetes.io/snp: "true"`
- Intel TDX shims: `intel.feature.node.kubernetes.io/tdx: "true"`
- IBM Secure Execution for Linux (SEL) shims (s390x): `feature.node.kubernetes.io/cpu-security.se.enabled: "true"`

Chart использует функцию `lookup` Helm для обнаружения NFD (путем поиска DaemonSet `node-feature-discovery-worker`). Автоматическое добавление выполняется только при обнаружении NFD и если для этого shim не установлен ручной `runtimeClass.nodeSelector`.

**Примечание**: Обнаружение NFD требует доступа к кластеру. Во время `helm template` (пробный запуск без кластера) внешний NFD не виден, поэтому автоматически добавляемые метки не добавляются. Ручные значения `runtimeClass.nodeSelector` все равно применяются во всех случаях.

## Настройка конфигурации с помощью Drop-in файлов

Когда kata-deploy устанавливает Kata Containers, базовые файлы конфигурации не должны изменяться напрямую. Вместо этого используйте drop-in конфигурационные файлы для настройки параметров. Этот подход гарантирует, что ваши пользовательские настройки сохранятся после обновлений kata-deploy.

### Как работают Drop-in файлы

Среда выполнения Kata читает базовый файл конфигурации, а затем применяет любые файлы `.toml`, найденные в каталоге `config.d/` рядом с ним. Файлы обрабатываются в алфавитном порядке, причем более поздние файлы переопределяют предыдущие настройки.

### Создание пользовательских Drop-in файлов

Чтобы добавить пользовательские настройки, создайте файл `.toml` в соответствующем каталоге `config.d/`. Используйте числовой префикс для управления порядком применения.

**Зарезервированные префиксы** (используемые kata-deploy):

- `10-*`: Основные настройки kata-deploy
- `20-*`: Настройки отладки
- `30-*`: Параметры ядра

**Рекомендуемые префиксы для пользовательских настроек**: `50-89`

### Примеры Drop-In конфигураций

#### Добавление пользовательских параметров ядра

```bash
# SSH на узел или используйте kubectl exec
sudo mkdir -p /opt/kata/share/defaults/kata-containers/runtimes/qemu/config.d/
sudo cat > /opt/kata/share/defaults/kata-containers/runtimes/qemu/config.d/50-custom.toml << 'EOF'
[hypervisor.qemu]
kernel_params = "my_param=value"
EOF
```

#### Изменение размера памяти по умолчанию

```bash
sudo cat > /opt/kata/share/defaults/kata-containers/runtimes/qemu/config.d/50-memory.toml << 'EOF'
[hypervisor.qemu]
default_memory = 4096
EOF
```