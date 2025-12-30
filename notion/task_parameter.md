# Task Parameter API

## Endpoints

### 1. Retrieve Products
**`GET /tenant/{idtenant}/products`**

### 2. Add Products to Tasks by merge_key
**`POST /tasks/products`**

### 3. Edit Products Linked to Tasks by merge_key
**`PUT /tasks/products`**

### 4. Updated API for action-tasks-by-date
**`GET /api/v2/action-tasks-by-date/{idtenant}/{date}`**

---

## Payload Example

```json
{
  "merge_key": "...",
  "products": [
    { "idproduct": "uuid", "quantity": 10.0 }
  ]
}
```

---

(Archived from Notion on 2025-12-22)
