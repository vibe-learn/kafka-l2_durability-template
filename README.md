        # kafka — Тройка надёжности: acks, ISR, replication factor

        Homework-шаблон для урока **l2_durability** (Тройка надёжности: acks, ISR, replication factor) на платформе Vibe Learn.

        ## Что делать

        Chaos-тест треугольника надёжности на Go. Запускаешь кластер из 3 брокеров через docker-compose.
Прокатываешься по 9 конфигурациям (комбинации acks=0/1/all × RF=1/2/3 × MIR=1/2/3 где MIR≤RF),
для каждой: пишешь 100 сообщений, в середине записи убиваешь один брокер через docker kill,
читаешь партицию и считаешь message_loss_count.

Результат — таблица: config | message_loss_count.

CI-проверки (автоматические):
- config=(acks=all, RF=3, MIR=2) → message_loss_count == 0
- config=(acks=1, RF=3, MIR=2) при правильном тайминге (kill во время записи лидера до репликации) → message_loss_count > 0

## Контекст (из transfer-задачи урока)

Команда X жалуется: «мы поставили acks=all и потеряли событие при failover». После разбора
выясняется конфигурация: replication.factor=2, min.insync.replicas=1.

**Вопрос:** объясни, что именно произошло технически — почему acks=all не защитил? Предложи
исправление конфигурации. Учти стоимость исправления: если поставить min.insync.replicas=2
при replication.factor=2, что произойдёт при потере любого брокера? Приемлемо ли это?
Какую конфигурацию ты порекомендуешь в итоге и почему?

## Recap из урока

- **Треугольник надёжности** — это три параметра, которые работают только вместе: `acks=all` (продьюсер ждёт ISR), `min.insync.replicas=2` (брокер требует минимум 2 живых реплики), `replication.factor=3` (есть из чего составить ISR при потере брокера).
- **`acks=all` + `min.insync.replicas=1` — иллюзия durability**: ISR может быть {лидер}, тогда ack от одной реплики, при failover данные теряются. Это эквивалентно acks=1.
- **Callback `producer.send()` при `acks=all`** означает: сообщение реплицировано на все реплики текущего ISR (≥ min.insync.replicas). Это не «доставлено консьюмеру» и не «fsync на диск».
- **End-to-end durability** требует кооперации: треугольник покрывает broker-side, но консьюмер должен коммитить offset после обработки, а downstream-система — поддерживать идемпотентность.
- **Цена треугольника**: latency (продьюсер блокируется на ожидание ISR) и availability (при ISR < MIR брокер бросает NotEnoughReplicasException). RF=3+MIR=2 — приемлемый баланс для production.

        ## Как работать

        1. Платформа Vibe Learn создаёт копию этого репо в твоём GitHub-аккаунте по клику «Начать домашку» на странице урока (через GitHub `/generate`, codecrafters-pattern).
        2. Склонируй копию локально, реализуй TODO в `main.go`, прогони тесты, запушь.
        3. CI (`.github/workflows/ci.yml`) запускает `go vet` + `go test ./...` на каждый push. Платформа слушает результат через webhook от GitHub Actions и обновляет статус домашки на странице урока.

        ## Локальное окружение

        - Go 1.22+
        - Docker + docker-compose — `docker compose -f docker-compose.yml up -d` поднимает 3-нодовый Kafka cluster на портах 9092/9093/9094, использовать в тестах через bootstrap `localhost:9092,localhost:9093,localhost:9094`.

        ## Запуск

        ```bash
        # Поднять локальный Kafka
        docker compose up -d

        # Прогнать тесты (часть из них стартует свой ephemeral testcontainers cluster, часть использует docker-compose выше)
        go test ./...

        # Запустить main (печатает marker; замени stub на реализацию)
        go run .
        ```

        ## Заметка автора

        Это baseline-шаблон, сгенерированный платформой. Бизнес-сущность задачи (что конкретно реализовать в `main.go`, какие тесты сделать строгими) расширяется по ходу итераций — параллельно с углублением теории урока.
