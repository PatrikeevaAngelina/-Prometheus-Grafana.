# 🚀 Мониторинг инфраструктуры с Prometheus и Grafana

Проект представляет собой полный стек мониторинга для сбора, хранения и визуализации метрик вашей инфраструктуры.

---

## 🏗️ Архитектура мониторинга

### Общая схема работы
+--------------+ Сбор метрик +-------------+ Запросы данных +----------+
| | ----------------> | | -------------------> | |
| Мониторируемые | | Prometheus | | Grafana |
| сервисы | | (9090) | | (3000) |
| (приложения, | | | | |
| базы данных, | +-------------+ +----------+
| системы) | |
+--------------+ | Отправка алертов
^ |
| v
| +---------------+
| | Alertmanager |
| | (9093) |
| +---------------+
| |
| | Уведомления
| v
| (Email, Slack, Telegram)
|
+--------------+
| Exporters |
| (node_exporter,|
| cadvisor) |
+--------------+

text

### Компоненты системы

- **Prometheus** - сервер для сбора и хранения временных рядов данных
- **Grafana** - платформа для визуализации метрик и создания дашбордов  
- **Node Exporter** - агент для сбора метрик операционной системы
- **cAdvisor** - сбор метрик контейнеров и Docker
- **Alertmanager** - обработка и отправка уведомлений

---

## 📊 Таблица основных метрик

В таблице ниже представлены ключевые метрики, которые собираются системой мониторинга:

| Группа метрик | Метрика | Тип | Описание |
|---------------|---------|-----|----------|
| **🖥️ Системные** | `node_cpu_seconds_total` | Counter | Суммарное время работы CPU по режимам |
| | `node_memory_MemAvailable_bytes` | Gauge | Доступная оперативная память |
| | `node_filesystem_avail_bytes` | Gauge | Свободное место на файловых системах |
| | `node_load1` | Gauge | Средняя загрузка системы за 1 минуту |
| **📦 Контейнеры** | `container_cpu_usage_seconds_total` | Counter | Использование CPU контейнерами |
| | `container_memory_usage_bytes` | Gauge | Использование памяти контейнерами |
| | `container_network_receive_bytes_total` | Counter | Входящий сетевой трафик |
| **🌐 Веб-сервисы** | `http_requests_total` | Counter | Общее количество HTTP запросов |
| | `http_request_duration_seconds` | Histogram | Время обработки HTTP запросов |
| | `up` | Gauge | Статус доступности сервиса (1=up, 0=down) |
| **🗄️ Базы данных** | `pg_database_size_bytes` | Gauge | Размер базы данных |
| | `pg_stat_activity_count` | Gauge | Количество активных подключений |

> **Пояснения по типам метрик:**
> - **Gauge** - текущее значение, которое может увеличиваться и уменьшаться
> - **Counter** - монотонно возрастающий счетчик
> - **Histogram** - для измерения распределений и квантилей

---

## ⚙️ Пример конфигурационного файла Prometheus

Ниже представлен базовый конфигурационный файл Prometheus:

```yaml
# prometheus.yml

global:
  scrape_interval: 15s      # Интервал сбора метрик
  evaluation_interval: 15s  # Интервал оценки правил алертинга

# Правила алертинга
rule_files:
  - "alerts/*.yml"

# Конфигурация Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

# Конфигурации сбора метрик
scrape_configs:
  # Мониторинг самого Prometheus
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 30s

  # Мониторинг Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    labels:
      environment: 'production'
      job: 'node-exporter'

  # Мониторинг cAdvisor
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    labels:
      environment: 'production'

  # Мониторинг произвольного приложения
  - job_name: 'web-app'
    static_configs:
      - targets: ['web-app:8080']
    metrics_path: '/metrics'
    scrape_interval: 10s
Пример правила алертинга
yaml
# alerts/system.yml
groups:
  - name: system
    rules:
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85% for more than 5 minutes"
