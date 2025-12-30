# Bynry-case-study
https://drive.google.com/file/d/1oXy-mRSd9sR9ihYgdb0n0I0dPcfXT_VO/view?usp=sharing


Backend Engineering Intern – Case Study (StockFlow)

Aishwary jain, jainaishwary07@gmail.com
LinkedIn
Tech Stack Assumed: Django, Django REST Framework, PostgreSQL



PART 1: Code Review & Debugging (Django)
Issues Identified:

 1. No request validation:
 The API directly trusts request.data fields. Missing or invalid fields will raise runtime errors.

 2. SKU uniqueness not enforced:
 Business rule requires SKU to be globally unique, but no check or DB constraint exists.

 3. Incorrect Product–Warehouse coupling:
 Product model stores warehouse_id directly, which breaks multi-warehouse support.

 4. No atomic transaction:
 Product and Inventory are saved separately. Partial failures cause data inconsistency.

 5. Price precision:
 Price should be handled using DecimalField, not float.

 Production Impact:
 - API crashes
 - Duplicate products
 - Inconsistent inventory
 - Poor scalability

 Corrected Django-Based Approach:
 - Use serializers for validation
 - Enforce unique constraints
 - Separate Product and Inventory
 - Wrap operations in transaction.atomic()

from decimal import Decimal
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json or {}

    # Basic validation
    if not data.get('name') or not data.get('sku'):
        return jsonify({"error": "name and sku are required"}), 400

    try:
        # Create product
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=Decimal(str(data.get('price', 0)))
        )

        db.session.add(product)
        db.session.flush()  # get product.id without commit

        # Create inventory only if provided
        if data.get('warehouse_id') and data.get('initial_quantity') is not None:
            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data['warehouse_id'],
                quantity=data['initial_quantity']
            )
            db.session.add(inventory)

        db.session.commit()

        return jsonify({
            "message": "Product created",
            "product_id": product.id
        }), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "SKU already exists"}), 409

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": "Something went wrong"}), 500







PART 2: Database Design
I am just writing the entities, table names etc. that I believe are essential
Core Models:

 Company
 - id
 - name

 Warehouse
 - id
 - company (FK)
 - name

 Product
 - id
 - name
 - sku (unique)
 - price (DecimalField)
 - product_type

 Inventory
 - id
 - product (FK)
 - warehouse (FK)
 - quantity
 - updated_at

 InventoryLog
 - id
 - inventory (FK)
 - change
 - reason
 - created_at

 Supplier
 - id
 - name
 - contact_email

 ProductSupplier (M2M)
 - product (FK)
 - supplier (FK)

 Bundle
 - bundle (FK to Product)
 - child_product (FK to Product)
 - quantity

 Design Justification:
 - Supports multi-warehouse products
 - Tracks stock history
 - Enables supplier reordering
 - Supports bundled products

Questions to be asked:
Do we track purchase price vs selling price?
Should bundles have dynamic pricing or fixed?
Do we need soft delete (active / inactive)?
Should inventory history track user / system changes?
Can warehouses transfer stock between each other?



Part 3 – Low Stock Alerts API (Django)
Assumptions (Say this first – interviewers like this)
Because requirements are incomplete, I assume:


Each Product has a product_type
Each product_type defines a low_stock_threshold
Inventory is tracked per (product, warehouse)
“Recent sales” means sales in the last 30 days
A Sale table exists for sales activity
Supplier is linked to Product


One alert = one product in one warehouse
Basic Models (Assumed)
Product(id, name, sku, product_type, supplier)
ProductType(id, name, low_stock_threshold)
Warehouse(id, name, company)
Inventory(product, warehouse, quantity)
Sale(product, warehouse, quantity, created_at)
Supplier(id, name, contact_email)
Django API Implementation (Simple Function-Based View)


from datetime import timedelta
from django.http import JsonResponse
from django.utils.timezone import now
from django.views.decorators.http import require_GET
from .models import Inventory, Sale
@require_GET
def low_stock_alerts(request, company_id):
    alerts = []
    thirty_days_ago = now() - timedelta(days=30)
    inventories = Inventory.objects.filter(
        warehouse__company_id=company_id
    ).select_related(
        'product',
        'warehouse',
        'product__supplier',
        'product__product_type'
    )
    for inventory in inventories:
        product = inventory.product
        warehouse = inventory.warehouse
        recent_sales = Sale.objects.filter(
            product=product,
            warehouse=warehouse,
            created_at__gte=thirty_days_ago
        )
        if not recent_sales.exists():
            continue 
        threshold = product.product_type.low_stock_threshold
        if inventory.quantity >= threshold:
            continue
        total_sold = sum(s.quantity for s in recent_sales)
        days = max((now() - thirty_days_ago).days, 1)
        avg_daily_sale = max(total_sold // days, 1)
        days_until_stockout = inventory.quantity // avg_daily_sale
        alerts.append({
            "product_id": product.id,
            "product_name": product.name,
            "sku": product.sku,
            "warehouse_id": warehouse.id,
            "warehouse_name": warehouse.name,
            "current_stock": inventory.quantity,
            "threshold": threshold,
            "days_until_stockout": days_until_stockout,
            "supplier": {
                "id": product.supplier.id,
                "name": product.supplier.name,
                "contact_email": product.supplier.contact_email
            }
        })
    return JsonResponse({
        "alerts": alerts,
        "total_alerts": len(alerts)
    })

Edge cases I handled
If a product has no recent sales, I skip it because alerts are only meaningful for actively selling products.
If a company has multiple warehouses, I iterate over inventory per warehouse so each warehouse is checked independently.
If sales quantity is zero or very low, I default the average daily sale to 1 to avoid division-by-zero errors.
If the low-stock threshold is not directly on the product, I fetch it from the product type, keeping business rules centralized.
For large datasets, I use select_related to reduce database queries and avoid N+1 query problems.
If no products qualify for alerts, the API returns an empty alerts list with total_alerts = 0, not an error.
