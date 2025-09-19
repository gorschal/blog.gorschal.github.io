---
layout: post
title: The Four Golden Signals
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - prometheus
---

Это абсолютный минимум того, что должно интересовать `Prometheus` в вашем приложении.

<!--more-->

### Трафик (Traffic)

**Что измеряет**: Насколько востребован сервис?  
**Метрики**:

- HTTP: http_requests_total (RPS - Requests Per Second).
- gRPC: grpc_server_requests_total.
- Message Queue: messages_consumed_total, messages_produced_total.

**Важно**: Разбивать по лейблам service_name, method, path, endpoint.

### Ошибки (Errors)

**Что измеряет**: Частота неудачных запросов/обработок.  
**Метрики**:

- HTTP: http_requests_total{status_code=~"5..|4.."} или отдельный http_errors_total.
- gRPC: grpc_server_requests_total{grpc_code!="OK"}.
- Общие: exceptions_thrown_total (по типам исключений).

**Важно**: Считать rate ошибок, а не абсолютные значения. rate(errors[5m]) / rate(total_requests[5m]) \* 100.

### Задержка (Latency)

**Что измеряет**: Как быстро сервис отвечает?  
**Метрики**:

- Гистограмма! http_request_duration_seconds_bucket (и её аналоги для gRPC, DB и т.д.).

**Важно**: Смотреть на перцентили (95й, 99й), а не на среднее. histogram_quantile(0.95, rate(...[5m])). 99-й перцентиль покажет худший опыт ваших пользователей.

### Насыщение (Saturation)

**Что измеряет**: Насколько "занят" сервис? Насколько он близок к своему лимиту?  
**Метрики**:

- Инфраструктура: CPU usage, Memory usage, Disk I/O.
- Приложение: Размер очереди задач (queue_size_current), использование пулов соединений (db_connections_active), заполненность кэша.
