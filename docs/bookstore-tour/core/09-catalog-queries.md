# Core 09 — Catalog queries, paging, and explicit SQL

## Business context

Customers need author/title filters, price bounds, stable sorting, and pagination. Loading every book into memory is no longer acceptable, and aggregate-oriented repositories are not always the best query tool.

## Request

Support:

```text
GET /api/v1/books?page=0&size=20&sort=title,asc&author=tolkien&maxPrice=30
```

## Application contract

Define framework-neutral query types such as `BookSearchQuery`, `PageQuery`, and `PagedResult<T>`. The HTTP adapter converts `Pageable`; application ports do not expose Spring Data types.

## Persistence choices

- Use a derived Spring Data query once to learn the mechanism.
- Use `JdbcClient` for an explicit projection query with selected columns and predicates.
- Keep query DTO/projection types outside the domain aggregate if they represent a read model rather than business behavior.
- Whitelist sortable fields; never concatenate arbitrary client input into SQL.

## Performance analysis

- Cap page size to prevent pagination abuse.
- `Page` usually requires a count query; `Slice` only determines whether more rows exist.
- Offset paging becomes expensive deep into large datasets; teach keyset/cursor pagination as the later alternative.
- Add indexes only from observed query plans and workload, not by decorating every column.
- Avoid N+1-style repeated lookups even without an ORM.

## Tests and proof

Prove stable sorting, combined filters, maximum page size, invalid sort rejection, empty results, and query-plan/index assumptions where performance claims are made.

## When not to use a repository abstraction

Reporting, search-heavy projections, and vendor-specific SQL often fit a dedicated query adapter better than forcing every query through aggregate repositories.

Continue to [orders and concurrency](10-orders-transactions-and-locking.md).

