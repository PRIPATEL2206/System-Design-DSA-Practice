# Django Admin Customization — Senior Level

## 1. Custom Admin for Production Use

### Model Registration (Beyond the Basics)
```python
from django.contrib import admin
from django.utils.html import format_html
from django.urls import reverse
from django.db.models import Count, Sum
from .models import Order, OrderItem, Product

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    # Display
    list_display = ['name', 'category_link', 'price', 'stock_status', 'status', 'created_at']
    list_display_links = ['name']
    list_filter = ['status', 'category', 'created_at']
    search_fields = ['name', 'description', 'sku']
    list_editable = ['status']  # inline editing in list view
    list_per_page = 50
    date_hierarchy = 'created_at'
    ordering = ['-created_at']
    
    # Form
    readonly_fields = ['id', 'created_at', 'updated_at', 'slug']
    prepopulated_fields = {'slug': ('name',)}
    autocomplete_fields = ['category']  # searchable dropdown
    fieldsets = (
        ('Basic Info', {
            'fields': ('name', 'slug', 'description', 'category')
        }),
        ('Pricing & Inventory', {
            'fields': ('price', 'sale_price', 'stock', 'sku')
        }),
        ('Status', {
            'fields': ('status',),
            'classes': ('collapse',),  # collapsible section
        }),
        ('Metadata', {
            'fields': ('id', 'created_at', 'updated_at'),
            'classes': ('collapse',),
        }),
    )
    
    # Performance: avoid N+1 in admin list
    def get_queryset(self, request):
        return super().get_queryset(request).select_related(
            'category'
        ).annotate(
            order_count=Count('orderitem')
        )
    
    # Custom columns with HTML
    @admin.display(description='Stock', ordering='stock')
    def stock_status(self, obj):
        if obj.stock == 0:
            color = 'red'
            text = 'Out of Stock'
        elif obj.stock < 10:
            color = 'orange'
            text = f'Low ({obj.stock})'
        else:
            color = 'green'
            text = f'In Stock ({obj.stock})'
        return format_html('<span style="color: {};">{}</span>', color, text)
    
    # Clickable FK link
    @admin.display(description='Category')
    def category_link(self, obj):
        url = reverse('admin:products_category_change', args=[obj.category.id])
        return format_html('<a href="{}">{}</a>', url, obj.category.name)
```

### Inline Models (Edit Related Objects Together)
```python
class OrderItemInline(admin.TabularInline):
    model = OrderItem
    extra = 0  # don't show empty forms
    readonly_fields = ['product', 'quantity', 'price', 'subtotal']
    can_delete = False
    
    @admin.display(description='Subtotal')
    def subtotal(self, obj):
        return f'₹{obj.price * obj.quantity:.2f}'
    
    def has_add_permission(self, request, obj=None):
        return False  # can't add items from admin


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'user_link', 'total', 'status', 'item_count', 'created_at']
    list_filter = ['status', 'created_at']
    search_fields = ['id', 'user__email', 'user__username']
    readonly_fields = ['id', 'user', 'total', 'created_at']
    inlines = [OrderItemInline]
    
    def get_queryset(self, request):
        return super().get_queryset(request).select_related(
            'user'
        ).annotate(
            item_count=Count('items')
        )
    
    @admin.display(description='Items', ordering='item_count')
    def item_count(self, obj):
        return obj.item_count
    
    @admin.display(description='User')
    def user_link(self, obj):
        url = reverse('admin:auth_user_change', args=[obj.user.id])
        return format_html('<a href="{}">{}</a>', url, obj.user.email)
```

### Custom Admin Actions
```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    actions = ['mark_as_shipped', 'export_to_csv', 'cancel_orders']
    
    @admin.action(description='Mark selected orders as shipped')
    def mark_as_shipped(self, request, queryset):
        updated = queryset.filter(status='paid').update(status='shipped')
        self.message_user(request, f'{updated} orders marked as shipped.')
    
    @admin.action(description='Export selected to CSV')
    def export_to_csv(self, request, queryset):
        import csv
        from django.http import HttpResponse
        
        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="orders.csv"'
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'User', 'Total', 'Status', 'Created'])
        
        for order in queryset.select_related('user'):
            writer.writerow([
                order.id, order.user.email, order.total,
                order.status, order.created_at.strftime('%Y-%m-%d')
            ])
        
        return response
    
    @admin.action(description='Cancel selected orders')
    def cancel_orders(self, request, queryset):
        # Only cancel pending orders
        pending = queryset.filter(status='pending')
        cancelled = pending.update(status='cancelled')
        skipped = queryset.exclude(status='pending').count()
        
        if skipped:
            self.message_user(
                request,
                f'{cancelled} cancelled. {skipped} skipped (not pending).',
                level='warning'
            )
        else:
            self.message_user(request, f'{cancelled} orders cancelled.')
```

---

## 2. Custom Admin Views & Dashboard

```python
from django.contrib import admin
from django.urls import path
from django.template.response import TemplateResponse
from django.db.models import Sum, Count
from django.db.models.functions import TruncDate

class CustomAdminSite(admin.AdminSite):
    site_header = 'MyApp Admin'
    site_title = 'MyApp'
    index_title = 'Dashboard'
    
    def get_urls(self):
        urls = super().get_urls()
        custom_urls = [
            path('dashboard/', self.admin_view(self.dashboard_view), name='dashboard'),
            path('reports/revenue/', self.admin_view(self.revenue_report), name='revenue_report'),
        ]
        return custom_urls + urls
    
    def dashboard_view(self, request):
        from apps.orders.models import Order
        from django.utils import timezone
        from datetime import timedelta
        
        today = timezone.now().date()
        last_30_days = today - timedelta(days=30)
        
        context = {
            **self.each_context(request),
            'title': 'Dashboard',
            'total_orders_today': Order.objects.filter(
                created_at__date=today
            ).count(),
            'revenue_today': Order.objects.filter(
                created_at__date=today, status='paid'
            ).aggregate(total=Sum('total'))['total'] or 0,
            'daily_revenue': Order.objects.filter(
                created_at__date__gte=last_30_days, status='paid'
            ).annotate(
                date=TruncDate('created_at')
            ).values('date').annotate(
                revenue=Sum('total'),
                count=Count('id')
            ).order_by('date'),
        }
        
        return TemplateResponse(request, 'admin/dashboard.html', context)

# Use custom admin site
admin_site = CustomAdminSite(name='myadmin')
```

---

## 3. Admin Security & Permissions

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    
    def has_delete_permission(self, request, obj=None):
        # Only superusers can delete orders
        return request.user.is_superuser
    
    def has_change_permission(self, request, obj=None):
        if obj and obj.status == 'shipped':
            return False  # can't edit shipped orders
        return super().has_change_permission(request, obj)
    
    def get_readonly_fields(self, request, obj=None):
        if obj and obj.status != 'pending':
            # Once order is processed, most fields become read-only
            return self.readonly_fields + ['user', 'total', 'status']
        return self.readonly_fields
    
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if not request.user.is_superuser:
            # Staff sees only their region's orders
            qs = qs.filter(region=request.user.profile.region)
        return qs

# Change admin URL (security: don't use /admin/)
# urls.py
urlpatterns = [
    path('management-portal-7x9k/', admin.site.urls),  # obscure URL
]
```

---

## 4. Admin Performance Tips

```python
# Problem: Admin list view with 1M records is slow
@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    # Disable count query on paginator (slow on large tables)
    show_full_result_count = False
    
    # Use select_related/prefetch_related
    def get_queryset(self, request):
        return super().get_queryset(request).select_related(
            'category', 'brand'
        ).prefetch_related('tags')
    
    # Raw ID widget for FK (dropdown is slow with 100K categories)
    raw_id_fields = ['category']  # shows ID input + lookup popup
    # Or: autocomplete_fields = ['category']  # AJAX search (requires search_fields on related admin)
    
    # Limit search to indexed fields
    search_fields = ['name', 'sku']  # NOT description (full scan)
    
    # Avoid expensive annotations in list view
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.resolver_match.url_name == 'products_product_changelist':
            return qs.only('id', 'name', 'price', 'status', 'category_id')
        return qs
```

---

## 5. Interview Questions

### Q: How do you customize Django Admin for a production app?
```
1. Custom list_display with formatted columns (HTML, colors)
2. Inline models for related objects (OrderItem inside Order)
3. Custom actions (bulk operations: export, status change)
4. Performance: select_related in get_queryset, show_full_result_count=False
5. Security: custom has_*_permission methods, obscure URL, IP whitelist
6. Custom dashboard views with business metrics
7. Audit logging (django-auditlog or django-simple-history)
```

### Q: When should you NOT use Django Admin?
```
- As a customer-facing UI (it's for internal staff only)
- For complex workflows (use custom views + forms instead)
- For high-traffic operations (admin isn't optimized for scale)
- When you need fine-grained RBAC (admin permissions are coarse)

Admin is great for: CRUD operations, data inspection, one-off admin tasks
Build custom UI for: user-facing features, complex business logic
```
