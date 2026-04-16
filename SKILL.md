---
name: coffestore-directus
description: >
  Schema completo de CoffeStore en Directus: pedidos, productos, clientes,
  variedades, orígenes y catálogo de café de especialidad colombiano.
  Usar cuando se consulten, analicen o modifiquen datos del e-commerce CoffeStore.
  Trigger: pedidos, orders, productos, clientes, customers, inventario, ventas,
  pagos crypto, USDC, USDT, facturación, envíos, newsletter, blog, banners.
---

# CoffeStore — Schema Directus

## Contexto del negocio

CoffeStore es un e-commerce de café de especialidad colombiano con pagos en
criptomonedas (USDC/USDT en BSC, Solana, Ethereum y Polygon).
Base URL de la API: `{{DIRECTUS_URL}}`
Autenticación: Bearer token via variable `{{DIRECTUS_TOKEN}}`

---

## Colecciones principales

### `orders` — Pedidos
| Campo | Tipo | Descripción |
|---|---|---|
| id | uuid | PK |
| order_number | string | Número legible del pedido |
| customer_id | uuid | FK → customers.id |
| customer_email | string | Email del cliente (desnormalizado) |
| customer_first_name | string | Nombre |
| customer_last_name | string | Apellido |
| customer_phone | string | Teléfono |
| shipping_address_line1 | text | Dirección línea 1 |
| shipping_address_line2 | text | Dirección línea 2 |
| shipping_city | string | Ciudad de envío |
| shipping_state | string | Departamento/Estado |
| shipping_postal_code | string | Código postal |
| shipping_country | string | País |
| subtotal | decimal | Subtotal sin envío ni impuestos |
| shipping_cost | decimal | Costo de envío |
| tax | decimal | Impuestos |
| total | decimal | Total final |
| payment_method | string | Método de pago (crypto, etc.) |
| payment_network | string | Red blockchain (BSC, Solana, Ethereum, Polygon) |
| payment_status | string | Estado del pago: `pending` `confirmed` `failed` |
| payment_transaction_hash | string | Hash de transacción blockchain |
| status | string | Estado del pedido (ver valores abajo) |
| notes | text | Notas internas |
| date_created | dateTime | Fecha de creación |
| date_updated | dateTime | Última actualización |

**Estados de pedido (`status`):**
```
pending → confirmed → processing → shipped → delivered
                                            → cancelled
                                            → refunded
```

**Filtros comunes de pedidos:**
```
Pendientes:         filter[status][_eq]=pending
Confirmados hoy:    filter[status][_eq]=confirmed&filter[date_created][_gte]=$NOW(-1d)
Por cliente:        filter[customer_id][_eq]={uuid}
Por email:          filter[customer_email][_eq]={email}
Pagos pendientes:   filter[payment_status][_eq]=pending
Por red crypto:     filter[payment_network][_eq]=BSC
```

---

### `order_items` — Ítems de pedido
Relación M2M entre orders y products.
| Campo | Tipo | Descripción |
|---|---|---|
| id | uuid | PK |
| order_id | uuid | FK → orders.id |
| product_id | uuid | FK → products.id |
| quantity | integer | Cantidad |
| unit_price | decimal | Precio unitario al momento de compra |
| subtotal | decimal | quantity × unit_price |

---

### `products` — Catálogo de café
| Campo | Tipo | Descripción |
|---|---|---|
| id | uuid | PK |
| status | string | `published` `draft` `archived` |
| name | string | Nombre del producto |
| slug | string | URL amigable |
| short_description | text | Descripción corta |
| description | text | Descripción completa |
| variety_id | uuid | FK → coffee_varieties.id |
| origin_id | uuid | FK → origins.id |
| process_id | uuid | FK → processes.id |
| roast_level | string | Nivel de tueste: `light` `medium` `medium-dark` `dark` |
| altitude | integer | Altitud en metros sobre el nivel del mar |
| harvest_year | integer | Año de cosecha |
| score_sca | integer | Puntuación SCA (Specialty Coffee Association) |
| tasting_notes | json | Array de notas de cata: `["chocolate", "caramelo", "frutas"]` |
| price | decimal | Precio de venta |
| compare_at_price | decimal | Precio anterior (tachado) |
| stock | integer | Unidades disponibles |
| sku | string | Código de producto |
| weight_grams | integer | Peso en gramos |
| image_main | uuid | FK → directus_files.id |
| featured | boolean | Producto destacado en home |
| bestseller | boolean | Más vendido |
| date_created | timestamp | Fecha de creación |

**Filtros comunes de productos:**
```
Publicados:         filter[status][_eq]=published
Destacados:         filter[featured][_eq]=true&filter[status][_eq]=published
Sin stock:          filter[stock][_lte]=0
Por variedad:       filter[variety_id][_eq]={uuid}
Por origen:         filter[origin_id][_eq]={uuid}
Score > 85:         filter[score_sca][_gte]=85
```

---

### `customers` — Clientes
| Campo | Tipo | Descripción |
|---|---|---|
| id | uuid | PK |
| email | string | Email único |
| first_name | string | Nombre |
| last_name | string | Apellido |
| phone | string | Teléfono |
| default_address_line1 | text | Dirección principal |
| default_city | string | Ciudad |
| default_state | string | Departamento |
| default_country | string | País |
| accepts_marketing | boolean | Acepta emails de marketing |
| status | string | `active` `inactive` `blocked` |
| date_created | dateTime | Fecha de registro |

---

### `coffee_varieties` — Variedades de café
Catálogo de variedades: Caturra, Castillo, Colombia, Geisha, etc.

### `origins` — Orígenes
Regiones cafeteras: Huila, Nariño, Cauca, Antioquia, Sierra Nevada, etc.
Incluye campo `country` para orígenes internacionales.

### `processes` — Procesos de beneficio
Lavado, Natural, Honey, Anaeróbico, etc.

---

## Colecciones de contenido

### `posts` — Blog editorial
Artículos sobre café, cultura cafetera, tutoriales de preparación.
Relaciones: `autores` (M2O), `post_categories` (M2M), `etiquetas` (M2M)

### `home_banners` — Banners del home
Carrusel de la página principal con campos: `title`, `subtitle`, `image`,
`cta_text`, `cta_url`, `status`, `active`, `start_date`, `end_date`, `priority`.

### `testimonials` — Testimonios de clientes
Reseñas y testimonios para mostrar en la tienda.

### `newsletter_subscribers` — Suscriptores
Lista de emails suscritos al newsletter con campo `status`: `active` `unsubscribed`.

### `site_settings` — Configuración global
Singleton con logo, colores, metadatos, redes sociales. Solo existe 1 registro.

### `products_images` — Galería de imágenes
Tabla junction para múltiples imágenes por producto.

---

## Patrones de query frecuentes

### Resumen de ventas del día
```
GET /items/orders
  ?filter[date_created][_gte]=$NOW(-1d)
  &filter[payment_status][_eq]=confirmed
  &aggregate[sum]=total
  &aggregate[count]=id
```

### Pedidos pendientes de despacho
```
GET /items/orders
  ?filter[status][_eq]=confirmed
  &filter[payment_status][_eq]=confirmed
  &sort=-date_created
  &fields=id,order_number,customer_first_name,customer_last_name,
          customer_email,total,payment_network,date_created
```

### Productos con stock bajo
```
GET /items/products
  ?filter[status][_eq]=published
  &filter[stock][_lte]=5
  &sort=stock
  &fields=id,name,sku,stock,price
```

### Top productos más vendidos
```
GET /items/order_items
  ?aggregate[sum]=quantity
  &groupBy[]=product_id
  &sort=-quantity_sum
  &limit=10
  &fields=product_id.name,product_id.sku,quantity
```

### Clientes que han comprado más de una vez
```
GET /items/orders
  ?aggregate[count]=id
  &groupBy[]=customer_id
  &filter[count][_gt]=1
  &sort=-count
  &fields=customer_id.email,customer_id.first_name,count
```

---

## Notas importantes para el agente

1. **Pagos crypto**: CoffeStore usa USDC/USDT. El campo `payment_transaction_hash`
   contiene el hash de la transacción en blockchain. `payment_network` indica la red.
   Un pago puede estar `pending` aunque el pedido esté `confirmed` si aún no se verificó
   la transacción en blockchain.

2. **Desnormalización de cliente en orders**: Los campos `customer_email`,
   `customer_first_name`, etc. están duplicados en `orders` intencionalmente para
   preservar el snapshot del cliente al momento de la compra.

3. **Productos de café de especialidad**: El campo `score_sca` es la puntuación
   de la Specialty Coffee Association. Puntajes ≥ 80 son considerados "specialty".
   Puntajes ≥ 90 son excepcionales y de alta rareza.

4. **tasting_notes**: Es un campo JSON con array de strings en español.
   Ejemplo: `["chocolate negro", "caramelo", "naranja", "acidez cítrica"]`

5. **Orígenes colombianos principales**: Huila (dulce, caramelo), Nariño (acidez brillante),
   Cauca (floral, frutal), Antioquia (equilibrado), Sierra Nevada (limpio, achocolatado).
