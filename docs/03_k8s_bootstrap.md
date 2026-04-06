# Kubernetes Bootstrap

## Overview

Инструкция по развёртыванию полного DevOps стека с нуля на чистой VM.

## Prerequisites

- Debian 12 (Bookworm)
- Минимум 4GB RAM, 20GB disk
- Доступ по SSH

## 1. Подготовка системы
```bash
# Обновляем список пакетов и устанавливаем свежие версии
# Это важно чтобы не было конфликтов при установке Kubernetes
sudo apt-get update && sudo apt-get upgrade -y

# Отключаем swap — Kubernetes сам управляет памятью
# и swap мешает его расчётам
sudo swapoff -a

# Убираем swap из автозапуска чтобы не включился после перезагрузки
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Включаем обработку сетевого трафика между контейнерами через iptables
# Без этого поды не смогут общаться друг с другом
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Применяем настройки ядра сразу без перезагрузки
sudo sysctl --system
```

## 2. Установка containerd

containerd — это среда выполнения контейнеров. Kubernetes использует её
для запуска подов. Мы ставим containerd вместо Docker потому что
Kubernetes не нуждается в полном Docker — только в runtime.
```bash
# Устанавливаем утилиты для добавления репозитория Docker
sudo apt-get install -y ca-certificates curl gnupg

# Создаём папку для хранения GPG ключей репозиториев
sudo install -m 0755 -d /etc/apt/keyrings

# Скачиваем GPG ключ Docker для проверки подлинности пакетов
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Добавляем репозиторий Docker в список источников пакетов
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Обновляем список пакетов чтобы apt увидел новый репозиторий
sudo apt-get update && sudo apt-get install -y containerd.io

# Генерируем стандартный конфиг containerd
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml

# Включаем SystemdCgroup — без этого Kubernetes и containerd
# будут конфликтовать при управлении ресурсами контейнеров
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Перезапускаем containerd с новым конфигом и добавляем в автозапуск
sudo systemctl restart containerd && sudo systemctl enable containerd
```

## 3. Установка kubeadm, kubelet, kubectl

Три главных компонента Kubernetes:
- **kubelet** — агент на ноде, запускает и следит за подами
- **kubeadm** — инструмент для инициализации кластера
- **kubectl** — CLI для управления кластером
```bash
# Скачиваем бинарники напрямую с dl.k8s.io
# (pkgs.k8s.io недоступен из-за сетевых ограничений)
curl -L --max-time 60 -O https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubectl
curl -L --max-time 60 -O https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubeadm
curl -L --max-time 60 -O https://dl.k8s.io/release/v1.29.15/bin/linux/amd64/kubelet

# Устанавливаем бинарники в системную папку и даём права на выполнение
sudo install -o root -g root -m 0755 kubectl kubeadm kubelet /usr/local/bin/

# Создаём systemd unit файл для kubelet
# systemd будет знать как запускать kubelet при старте системы
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Создаём drop-in конфиг — дополнительные параметры запуска kubelet
# (какой runtime использовать, где искать конфиг кластера)
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/local/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
EOF

# Перезагружаем systemd чтобы увидел новые файлы и запускаем kubelet
sudo systemctl daemon-reload && sudo systemctl enable kubelet && sudo systemctl start kubelet
```

## 4. Инициализация кластера
```bash
# Инициализируем кластер
# --pod-network-cidr — диапазон IP адресов для подов
# этот диапазон нужен Calico для настройки сети
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Копируем конфиг кластера в домашнюю папку пользователя
# kubectl будет использовать этот файл для подключения к кластеру
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Убираем taint с мастер ноды — по умолчанию на мастере
# не запускаются обычные поды, это разрешает их запуск
# (нужно так как у нас одна нода)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 5. Установка Calico (CNI)

CNI (Container Network Interface) — это плагин который создаёт
виртуальную сеть между подами. Без CNI поды не видят друг друга.
Calico — один из самых популярных CNI плагинов.
```bash
# Применяем манифест Calico — он создаст все нужные компоненты
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Ждём пока все поды Calico запустятся (может занять 1-2 минуты)
kubectl get pods -n kube-system -w
```

## 6. Установка Helm

Helm — пакетный менеджер для Kubernetes. Как apt для Linux,
только для Kubernetes. Позволяет устанавливать сложные приложения
одной командой с настраиваемыми параметрами.
```bash
# Скачиваем и запускаем официальный скрипт установки Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## 7. Установка FluxCD

FluxCD — GitOps инструмент. Он следит за репозиторием infra.git
и автоматически применяет изменения в кластер. Мы не деплоим руками —
Flux делает это сам когда видит изменения в Git.
```bash
# Подключаем FluxCD к GitHub репозиторию infra
# Flux создаст Deploy Key в репозитории и начнёт следить за ним
flux bootstrap github \
  --owner=bogdanzemly-va \
  --repository=infra \
  --branch=main \
  --path=clusters/vm
```

## 8. Установка Prometheus + Grafana

kube-prometheus-stack — это один Helm чарт который устанавливает сразу:
Prometheus (сбор метрик), Grafana (визуализация), Alertmanager (алерты),
Node Exporter (метрики самой машины).
```bash
# Добавляем репозиторий с чартом
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Создаём отдельный namespace для мониторинга
kubectl create namespace monitoring

# Устанавливаем стек мониторинга
# NodePort 32000 — порт на котором будет доступна Grafana
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=32000
```

## 9. Установка Vault

HashiCorp Vault — система управления секретами. Хранит пароли,
токены и ключи в зашифрованном виде. Поды получают секреты
через Vault Agent вместо того чтобы хранить их в манифестах.
```bash
# Скачиваем Helm чарт Vault с GitHub (прямые серверы HashiCorp заблокированы)
curl -L -O https://github.com/hashicorp/vault-helm/archive/refs/tags/v0.27.0.tar.gz
tar -xzf v0.27.0.tar.gz

# Создаём namespace для Vault
kubectl create namespace vault

# Устанавливаем Vault в dev режиме (для продакшена нужен HA режим)
# dev режим — данные в памяти, не нужна инициализация и unseal
helm install vault ./vault-helm-0.27.0 \
  --namespace vault \
  --set server.dev.enabled=true \
  --set server.dev.devRootToken="root" \
  --set injector.enabled=true

# Настраиваем секреты для сервисов
kubectl exec -it vault-0 -n vault -- sh -c '
export VAULT_TOKEN=root
vault secrets enable -path=microservices kv-v2
vault kv put microservices/service-py/dev \
  POSTGRES_URL="postgresql://postgres:postgres@postgres.infra.svc.cluster.local:5432/orders" \
  REDIS_URL="redis://redis.infra.svc.cluster.local:6379" \
  KAFKA_BROKER="kafka.infra.svc.cluster.local:9092"
vault kv put microservices/service-go/dev \
  MONGO_URL="mongodb://mongo.infra.svc.cluster.local:27017/shipments" \
  REDIS_URL="redis://redis.infra.svc.cluster.local:6379" \
  KAFKA_BROKER="kafka.infra.svc.cluster.local:9092" \
  RABBITMQ_URL="amqp://guest:guest@rabbitmq.infra.svc.cluster.local:5672"
vault auth enable kubernetes
vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc.cluster.local:443"
'
```

## 10. Проверка
```bash
# Все поды должны быть в статусе Running
kubectl get pods -A

# Нода должна быть в статусе Ready
kubectl get nodes

# Проверяем сервисы в namespace infra
kubectl get svc -n infra
```