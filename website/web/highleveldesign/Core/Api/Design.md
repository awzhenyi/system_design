# Designing APIs

## Good Practices

### 1. Clear naming
- Use plural names to indicate a group of resources
- keep it clear and simple

### 2. Ensure reliability through idempotency
- calling the same requests multiple times return the same response (GET, PUT, DELETE)
- Usually POST are not idempotent (can design to require some unique id per api)
- PATCH usually are not idempotent

### 3. add versioning
- api/v1/..., api/v2/...
- allows for update of api while supporting backward compatibility

### 4. Add pagination
- `page + offset` (page size, page num `<->` Sql LIMIT + OFFSET) -> could be slow for large dataset 
- cursor based

### 5. Clear query strings for sorting and filtering of data
- add filter, sort_by, query params field in api
- eg. (GET v1/products/?filter=size:10&sort_by=data_added&size=15)

### 6. Security
- do not include sensitive info in request url, due to chance of it getting logged. instead use headers to store sensitive information
- enforce tls encryption

### 7. Direct paths
- api/v1/carts/{cart_id}/item/{item_id} ✅
- api/v1/items?item_id={item_id}&cart_id={cart_id} ❌