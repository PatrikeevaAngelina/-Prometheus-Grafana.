# 🚀 Мониторинг инфраструктуры с Prometheus и Grafana

Проект представляет собой полный стек мониторинга для сбора, хранения и визуализации метрик вашей инфраструктуры.

---

## 🏗️ Архитектура мониторинга

### Компоненты системы

- **Prometheus** - сервер для сбора и хранения временных рядов данных (порт 9090)
- **Grafana** - платформа для визуализации метрик и создания дашбордов (порт 3000)
- **Node Exporter** - агент для сбора метрик операционной системы (порт 9100)
- **cAdvisor** - сбор метрик контейнеров и Docker (порт 8080)
- **Alertmanager** - обработка и отправка уведомлений (порт 9093)

### Схема работы

1. **Сбор метрик**: Exporters (node_exporter, cadvisor) собирают метрики с систем и приложений
2. **Хранение**: Prometheus периодически забирает метрики по HTTP и сохраняет их в базу данных
3. **Визуализация**: Grafana запрашивает данные из Prometheus через API для отображения в дашбордах
4. **Алертинг**: Prometheus проверяет правила алертинга и отправляет уведомления в Alertmanager
5. **Уведомления**: Alertmanager обрабатывает и отправляет алерты по email, Slack, Telegram и другим каналам

---

## 📊 Основные метрики и пояснения

В таблице ниже представлены ключевые метрики, которые собираются системой мониторинга:

| Группа метрик | Метрика | Тип | Описание |
|---------------|---------|-----|----------|
| **🖥️ Системные** | `node_cpu_seconds_total` | Counter | Суммарное время работы CPU в разных режимах (user, system, idle) |
| | `node_memory_MemAvailable_bytes` | Gauge | Доступная оперативная память в байтах |
| | `node_filesystem_avail_bytes` | Gauge | Свободное место на файловых системах в байтах |
| | `node_load1` | Gauge | Средняя загрузка системы за 1 минуту |
| | `node_network_receive_bytes_total` | Counter | Общее количество полученных сетевых байт |
| **📦 Контейнеры** | `container_cpu_usage_seconds_total` | Counter | Суммарное использование CPU контейнерами в секундах |
| | `container_memory_usage_bytes` | Gauge | Текущее использование памяти контейнерами в байтах |
| | `container_network_receive_bytes_total` | Counter | Входящий сетевой трафик контейнеров |
| | `container_fs_usage_bytes` | Gauge | Использование дискового пространства контейнерами |
| **🌐 Веб-сервисы** | `http_requests_total` | Counter | Общее количество HTTP запросов |
| | `http_request_duration_seconds` | Histogram | Время обработки HTTP запросов (распределение) |
| | `up` | Gauge | Статус доступности сервиса (1 = доступен, 0 = недоступен) |
| | `process_resident_memory_bytes` | Gauge | Память, используемая процессом |
| **🗄️ Базы данных** | `pg_database_size_bytes` | Gauge | Размер базы данных PostgreSQL в байтах |
| | `pg_stat_activity_count` | Gauge | Количество активных подключений к БД |
| | `pg_postmaster_uptime_seconds` | Gauge | Время работы сервера PostgreSQL |

> **Пояснения по типам метрик:**
> - **Gauge** - текущее значение, которое может увеличиваться и уменьшаться (память, температура, загрузка)
> - **Counter** - монотонно возрастающий счетчик (запросы, ошибки, трафик)
> - **Histogram** - для измерения распределений и квантилей (время ответа, размеры запросов)

---

## ⚙️ Пример конфигурационного файла Prometheus

```yaml
# prometheus.yml

global:
  scrape_interval: 15s      # Интервал сбора метрик
  evaluation_interval: 15s  # Интервал оценки правил алертинга
  external_labels:
    environment: 'production'
    cluster: 'main'

# Правила алертинга (вынесены в отдельные файлы)
rule_files:
  - "/etc/prometheus/alerts/*.yml"

# Конфигурация Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
      scheme: http
      timeout: 10s

# Конфигурации сбора метрик
scrape_configs:
  # Мониторинг самого Prometheus
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 30s
    metrics_path: '/metrics'
    honor_labels: true

  # Мониторинг Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    scrape_interval: 15s
    labels:
      environment: 'production'
      job: 'node-exporter'
      group: 'infrastructure'

  # Мониторинг cAdvisor
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    scrape_interval: 15s
    labels:
      environment: 'production'
      job: 'cadvisor'
      group: 'containers'

  # Мониторинг произвольного приложения
  - job_name: 'web-app'
    static_configs:
      - targets: ['web-app:8080']
    scrape_interval: 10s
    metrics_path: '/metrics'
    labels:
      environment: 'production'
      job: 'application'
      group: 'services'
# alerts/system.yml
groups:
  - name: system
    rules:
      # Alert при высокой загрузке CPU
      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
          team: devops
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for more than 5 minutes. Current value: {{ $value | humanize }}%"
          
      # Alert при нехватке памяти
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: devops
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85% for more than 5 minutes. Current value: {{ $value | humanize }}%"
          
      # Alert при нехватке дискового пространства
      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 90
        for: 2m
        labels:
          severity: critical
          team: devops
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk usage is above 90% on {{ $labels.device }}. Current value: {{ $value | humanize }}%"
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts:/etc/prometheus/alerts
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
    networks:
      - monitoring
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring
    restart: unless-stopped

volumes:
  prometheus_data:
    driver: local
  grafana_data:
    driver: local

networks:
  monitoring:
    driver: bridge
