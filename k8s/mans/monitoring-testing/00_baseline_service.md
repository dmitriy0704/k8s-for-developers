# Baseline: tasks-service + PostgreSQL

## Условия
- tasks-service: Spring Boot 4.1.1, HikariCP 30, Tomcat 400
- PostgreSQL: 16-alpine, Docker
- k6: ramp-up 30s на каждый stage

## Результаты
| VU | RPS | p(95) | failed | max |
|----|-----|-------|--------|-----|
| 100 | 42 | 10.43ms | 0% | 40ms |
| 300 | 165 | 11.63ms | 0% | 67ms |
| 500 | 278 | 14.59ms | 0% | 83ms |
| 800 | 448 | 11.01ms | 0% | 122ms |

## Метрики при 800 VU
- hikaricp_acquire_max: 38ms (нет перегрузки)
- hikaricp_pending: 0 (нет очереди)
- jvm_gc_pause_max: 7ms (GC не влияет)
- pg_stat_activity: 30 idle, 1 active (PG не напряжён)

## Вывод
Capacity > 800 VU. Точка деградации не достигнута.
Это baseline для сравнения с Gateway, Istio, Rate Limit.