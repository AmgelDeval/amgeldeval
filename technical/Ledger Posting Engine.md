# Ledger Posting Engine



## Outbox Table

```sql
CREATE TABLE outbox_event (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type VARCHAR(50) NOT NULL,  -- VOUCHER
    aggregate_id VARCHAR(50) NOT NULL,    -- voucherId
    event_type VARCHAR(50) NOT NULL,      -- LEDGER_UPDATE
    payload JSONB NOT NULL,

    status VARCHAR(20) NOT NULL DEFAULT 'NEW',
    retry_count INT DEFAULT 0,

    created_at TIMESTAMP NOT NULL DEFAULT now(),
    processed_at TIMESTAMP NULL
);

CREATE INDEX idx_outbox_new
ON outbox_event(status, created_at);
```

## Writing to Outbox (Producer)

- If transaction commits → both rows exist
- If rollback → neither exists

  
```java

@Transactional
public void createVoucher(Voucher voucher) {

    voucherRepository.save(voucher);

    OutboxEvent event = OutboxEvent.builder()
        .aggregateType("VOUCHER")
        .aggregateId(voucher.getId())
        .eventType("LEDGER_UPDATE")
        .payload(toJson(voucher))
        .build();

    outboxRepository.save(event);
}
```


## Outbox Processor (Cluster-Safe)

```java

@Scheduled(fixedDelay = 1000)
@Transactional
public void processOutbox() {

    Optional<OutboxEvent> event =
        outboxRepository.fetchNextUnprocessed();

    if (event.isEmpty()) return;

    OutboxEvent e = event.get();

    try {
        ledgerService.updateAllLedgers(
            e.getAggregateId(),
            fromJson(e.getPayload())
        );

        e.markProcessed();

    } catch (Exception ex) {
        e.incrementRetry();
        throw ex;
    }
}



```


## Repository (Critical Part)

```java

@Query(value = """
    SELECT *
    FROM outbox_event
    WHERE status = 'NEW'
    ORDER BY created_at
    FOR UPDATE SKIP LOCKED
    LIMIT 1
    """, nativeQuery = true)
Optional<OutboxEvent> fetchNextUnprocessed();


```

- Multiple pods safe
- Exactly one processes a row
