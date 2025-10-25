# 🚀 Мониторинг инфраструктуры с Prometheus и Grafana

<h3 style="color: #4CAF50;">✅ Проект представляет собой полный стек мониторинга для сбора, хранения и визуализации метрик вашей инфраструктуры.</h3>

---

## 🏗️ Архитектура мониторинга

### Компоненты системы

- <span style="color: #2196F3;">**Prometheus**</span> - сервер для сбора и хранения временных рядов данных (порт 9090)
- <span style="color: #FF9800;">**Grafana**</span> - платформа для визуализации метрик и создания дашбордов (порт 3000)
- <span style="color: #4CAF50;">**Node Exporter**</span> - агент для сбора метрик операционной системы (порт 9100)
- <span style="color: #9C27B0;">**cAdvisor**</span> - сбор метрик контейнеров и Docker (порт 8080)
- <span style="color: #F44336;">**Alertmanager**</span> - обработка и отправка уведомлений (порт 9093)

### Схема работы

1. <span style="color: #4CAF50;">**Сбор метрик**</span>: Exporters (node_exporter, cadvisor) собирают метрики с систем и приложений
2. <span style="color: #2196F3;">**Хранение**</span>: Prometheus периодически забирает метрики по HTTP и сохраняет их в базу данных
3. <span style="color: #FF9800;">**Визуализация**</span>: Grafana запрашивает данные из Prometheus через API для отображения в дашбордах
4. <span style="color: #F44336;">**Алертинг**</span>: Prometheus проверяет правила алертинга и отправляет уведомления в Alertmanager
5. <span style="color: #9C27B0;">**Уведомления**</span>: Alertmanager обрабатывает и отправляет алерты по email, Slack, Telegram и другим каналам

---

## 📊 Основные метрики и пояснения

<p style="color: #2196F3; font-weight: bold;">📈 В таблице ниже представлены ключевые метрики, которые собираются системой мониторинга:</p>

| Группа метрик | Метрика | Тип | Описание |
|---------------|---------|-----|----------|
| <span style="color: #4CAF50;">**🖥️ Системные**</span> | `node_cpu_seconds_total` | <span style="color: #FF9800;">Counter</span> | <span style="color: #2196F3;">Суммарное время работы CPU в разных режимах</span> |
| | `node_memory_MemAvailable_bytes` | <span style="color: #4CAF50;">Gauge</span> | <span style="color: #F44336;">Доступная оперативная память в байтах</span> |
| | `node_filesystem_avail_bytes` | <span style="color: #4CAF50;">Gauge</span> | <span style="color: #9C27B0;">Свободное место на файловых системах</span> |
| | `node_load1` | <span style="color: #4CAF50;">Gauge</span> | <span style="color: #2196F3;">Средняя загрузка системы за 1 минуту</span> |
| <span style="color: #9C27B0;">**📦 Контейнеры**</span> | `container_cpu_usage_seconds_total` | <span style="color: #FF9800;">Counter</span> | <span style="color: #4CAF50;">Использование CPU контейнерами</span> |
| | `container_memory_usage_bytes` | <span style="color: #4CAF50;">Gauge</span> | <span style="color: #F44336;">Использование памяти контейнерами</span> |
| | `container_network_receive_bytes_total` | <span style="color: #FF9800;">Counter</span> | <span style="color: #2196F3;">Входящий сетевой трафик</span> |
| <span style="color: #2196F3;">**🌐 Веб-сервисы**</span> | `http_requests_total` | <span style="color: #FF9800;">Counter</span> | <span style="color: #4CAF50;">Общее количество HTTP запросов</span> |
| | `up` | <span style="color: #4CAF50;">Gauge</span> | <span style="color: #F44336;">Статус доступности сервиса (1=up, 0=down)</span> |

> <span style="color: #FF9800;">**Пояснения по типам метрик:**</span>
> - <span style="color: #4CAF50;">**Gauge**</span> - <span style="color: #4CAF50;">текущее значение, которое может увеличиваться и уменьшаться</span>
> - <span style="color: #FF9800;">**Counter**</span> - <span style="color: #FF9800;">монотонно возрастающий счетчик</span>
> - <span style="color: #9C27B0;">**Histogram**</span> - <span style="color: #9C27B0;">для измерения распределений и квантилей</span>

---

## ⚙️ Пример конфигурационного файла Prometheus

<p style="color: #4CAF50; font-weight: bold;">🛠️ Ниже представлен базовый конфигурационный файл Prometheus:</p>

```yaml
# prometheus.yml

global:
  scrape_interval: 15s      # <span style="color: #4CAF50;">Интервал сбора метрик</span>
  evaluation_interval: 15s  # <span style="color: #FF9800;">Интервал оценки правил алертинга</span>
  external_labels:
    environment: 'production'
    cluster: 'main'

# <span style="color: #F44336;">Правила алертинга (вынесены в отдельные файлы)</span>
rule_files:
  - "/etc/prometheus/alerts/*.yml"

# <span style="color: #9C27B0;">Конфигурация Alertmanager</span>
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

# <span style="color: #2196F3;">Конфигурации сбора метрик</span>
scrape_configs:
  # <span style="color: #4CAF50;">Мониторинг самого Prometheus</span>
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # <span style="color: #FF9800;">Мониторинг Node Exporter</span>
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    labels:
      environment: 'production'

  # <span style="color: #9C27B0;">Мониторинг cAdvisor</span>
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
# alerts/system.yml
groups:
  - name: system
    rules:
      # <span style="color: #F44336;">Alert при высокой загрузке CPU</span>
      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: <span style="color: #FF9800;">warning</span>
        annotations:
          summary: "<span style="color: #F44336;">High CPU usage on {{ $labels.instance }}</span>"
          description: "<span style="color: #FF9800;">CPU usage is above 80% for more than 5 minutes</span>"
          
      # <span style="color: #F44336;">Alert при нехватке памяти</span>
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: <span style="color: #F44336;">critical</span>
        annotations:
          summary: "<span style="color: #F44336;">High memory usage on {{ $labels.instance }}</span>"
          description: "<span style="color: #9C27B0;">Memory usage is above 85% for more than 5 minutes</span>"
<p style="color: #2196F3; font-weight: bold;">🎯 Рекомендуем ознакомиться с <span style="color: #FF9800;">[подробным руководством](https://habr.com/ru/articles/709204/)</span> для более глубокого понимания работы стека мониторинга.</p>
