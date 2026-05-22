# Django Interview Coding Rounds — Senior Level

## Common Patterns Asked in Django Machine Coding Rounds

These are actual problems given at companies like Flipkart, Swiggy, Razorpay, and Zerodha
in their Django/Python backend rounds. You typically get 60-90 minutes.

---

## Problem 1: Rate Limiter API

**Task:** Build a rate limiter that allows N requests per user per minute.
If limit exceeded, return 429. Store state in Redis.

```python
# models.py — no model needed (Redis only)

# services.py
from django.core.cache import cache
import time

class RateLimiter:
    """Sliding window rate limiter using Redis sorted sets."""
    
    def __init__(self, limit=100, window=60):
        self.limit = limit
        self.window = window  # seconds
    
    def is_allowed(self, user_id):
        key = f'ratelimit:{user_id}'
        now = time.time()
        window_start = now - self.window
        
        pipe = cache.client.get_client().pipeline()
        
        # Remove expired entries
        pipe.zremrangebyscore(key, 0, window_start)
        # Count current window
        pipe.zcard(key)
        # Add current request
        pipe.zadd(key, {str(now): now})
        # Set expiry on key
        pipe.expire(key, self.window)
        
        results = pipe.execute()
        current_count = results[1]
        
        if current_count >= self.limit:
            return False, self.limit - current_count, self._retry_after(key)
        return True, self.limit - current_count - 1, 0
    
    def _retry_after(self, key):
        """Seconds until oldest request expires."""
        oldest = cache.client.get_client().zrange(key, 0, 0, withscores=True)
        if oldest:
            return int(self.window - (time.time() - oldest[0][1]))
        return self.window


# views.py
from rest_framework.views import APIView
from rest_framework.response import Response

class RateLimitedView(APIView):
    rate_limiter = RateLimiter(limit=100, window=60)
    
    def dispatch(self, request, *args, **kwargs):
        user_id = request.user.id if request.user.is_authenticated else request.META['REMOTE_ADDR']
        
        allowed, remaining, retry_after = self.rate_limiter.is_allowed(user_id)
        
        if not allowed:
            return Response(
                {'error': 'Rate limit exceeded'},
                status=429,
                headers={
                    'X-RateLimit-Limit': str(self.rate_limiter.limit),
                    'X-RateLimit-Remaining': '0',
                    'Retry-After': str(retry_after),
                }
            )
        
        response = super().dispatch(request, *args, **kwargs)
        response['X-RateLimit-Limit'] = str(self.rate_limiter.limit)
        response['X-RateLimit-Remaining'] = str(remaining)
        return response
```

---

## Problem 2: URL Shortener Service

**Task:** Build a URL shortener with analytics. POST creates short URL,
GET redirects, GET /stats/{code} returns click count.

```python
# models.py
import string
import random
from django.db import models

class ShortURL(models.Model):
    original_url = models.URLField(max_length=2048)
    code = models.CharField(max_length=10, unique=True, db_index=True)
    created_by = models.ForeignKey('auth.User', on_delete=models.CASCADE, null=True)
    click_count = models.PositiveIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField(null=True, blank=True)
    
    @classmethod
    def generate_code(cls, length=7):
        chars = string.ascii_letters + string.digits
        while True:
            code = ''.join(random.choices(chars, k=length))
            if not cls.objects.filter(code=code).exists():
                return code


class ClickEvent(models.Model):
    short_url = models.ForeignKey(ShortURL, on_delete=models.CASCADE, related_name='clicks')
    ip_address = models.GenericIPAddressField()
    user_agent = models.TextField(blank=True)
    referer = models.URLField(blank=True, null=True)
    clicked_at = models.DateTimeField(auto_now_add=True)
    country = models.CharField(max_length=2, blank=True)


# serializers.py
from rest_framework import serializers

class CreateShortURLSerializer(serializers.Serializer):
    url = serializers.URLField()
    custom_code = serializers.CharField(max_length=10, required=False)
    expires_in_hours = serializers.IntegerField(required=False, min_value=1)
    
    def validate_custom_code(self, value):
        if ShortURL.objects.filter(code=value).exists():
            raise serializers.ValidationError('This code is already taken.')
        return value

class ShortURLStatsSerializer(serializers.ModelSerializer):
    class Meta:
        model = ShortURL
        fields = ['code', 'original_url', 'click_count', 'created_at', 'expires_at']


# views.py
from django.shortcuts import redirect
from django.utils import timezone
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.db.models import F

class CreateShortURLView(APIView):
    def post(self, request):
        serializer = CreateShortURLSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        data = serializer.validated_data
        code = data.get('custom_code') or ShortURL.generate_code()
        
        expires_at = None
        if data.get('expires_in_hours'):
            from datetime import timedelta
            expires_at = timezone.now() + timedelta(hours=data['expires_in_hours'])
        
        short_url = ShortURL.objects.create(
            original_url=data['url'],
            code=code,
            created_by=request.user if request.user.is_authenticated else None,
            expires_at=expires_at,
        )
        
        return Response({
            'short_url': f'https://short.ly/{short_url.code}',
            'code': short_url.code,
            'expires_at': short_url.expires_at,
        }, status=status.HTTP_201_CREATED)


class RedirectView(APIView):
    authentication_classes = []
    permission_classes = []
    
    def get(self, request, code):
        try:
            short_url = ShortURL.objects.get(code=code)
        except ShortURL.DoesNotExist:
            return Response({'error': 'Not found'}, status=404)
        
        if short_url.expires_at and short_url.expires_at < timezone.now():
            return Response({'error': 'Link expired'}, status=410)
        
        # Atomic increment (no race condition)
        ShortURL.objects.filter(code=code).update(click_count=F('click_count') + 1)
        
        # Log click asynchronously
        log_click.delay(short_url.id, request.META.get('REMOTE_ADDR'),
                       request.META.get('HTTP_USER_AGENT', ''))
        
        return redirect(short_url.original_url, permanent=False)


class StatsView(APIView):
    def get(self, request, code):
        short_url = ShortURL.objects.get(code=code)
        
        from django.db.models.functions import TruncDate
        from django.db.models import Count
        
        daily_clicks = ClickEvent.objects.filter(
            short_url=short_url
        ).annotate(
            date=TruncDate('clicked_at')
        ).values('date').annotate(count=Count('id')).order_by('-date')[:30]
        
        return Response({
            'url': ShortURLStatsSerializer(short_url).data,
            'daily_clicks': list(daily_clicks),
        })
```

---

## Problem 3: Task Queue (Simple Celery Clone)

**Task:** Design a simple async task queue using Django + Redis.
Tasks can be submitted, polled for status, and retried on failure.

```python
# models.py
import uuid
from django.db import models

class Task(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending'
        RUNNING = 'running'
        COMPLETED = 'completed'
        FAILED = 'failed'
        RETRYING = 'retrying'
    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    task_name = models.CharField(max_length=255)
    payload = models.JSONField(default=dict)
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING)
    result = models.JSONField(null=True, blank=True)
    error = models.TextField(blank=True)
    retries = models.IntegerField(default=0)
    max_retries = models.IntegerField(default=3)
    created_at = models.DateTimeField(auto_now_add=True)
    started_at = models.DateTimeField(null=True)
    completed_at = models.DateTimeField(null=True)


# services.py
import importlib
from django.utils import timezone
from django.db.models import F

class TaskQueue:
    
    @staticmethod
    def submit(task_name, payload=None, max_retries=3):
        """Submit a task to the queue."""
        task = Task.objects.create(
            task_name=task_name,
            payload=payload or {},
            max_retries=max_retries,
        )
        # Push to Redis queue
        from django.core.cache import cache
        cache.rpush('task_queue', str(task.id))
        return task
    
    @staticmethod
    def process_next():
        """Worker picks next task from queue and executes."""
        from django.core.cache import cache
        
        task_id = cache.lpop('task_queue')
        if not task_id:
            return None
        
        task = Task.objects.get(id=task_id)
        task.status = Task.Status.RUNNING
        task.started_at = timezone.now()
        task.save(update_fields=['status', 'started_at'])
        
        try:
            # Dynamically call the task function
            module_path, func_name = task.task_name.rsplit('.', 1)
            module = importlib.import_module(module_path)
            func = getattr(module, func_name)
            
            result = func(**task.payload)
            
            task.status = Task.Status.COMPLETED
            task.result = result
            task.completed_at = timezone.now()
            task.save(update_fields=['status', 'result', 'completed_at'])
            
        except Exception as e:
            task.error = str(e)
            task.retries += 1
            
            if task.retries < task.max_retries:
                task.status = Task.Status.RETRYING
                task.save(update_fields=['status', 'error', 'retries'])
                # Re-queue with delay
                cache.rpush('task_queue', str(task.id))
            else:
                task.status = Task.Status.FAILED
                task.completed_at = timezone.now()
                task.save(update_fields=['status', 'error', 'retries', 'completed_at'])
        
        return task


# views.py
class SubmitTaskView(APIView):
    def post(self, request):
        task = TaskQueue.submit(
            task_name=request.data['task_name'],
            payload=request.data.get('payload', {}),
        )
        return Response({'task_id': str(task.id), 'status': task.status}, status=201)

class TaskStatusView(APIView):
    def get(self, request, task_id):
        task = Task.objects.get(id=task_id)
        return Response({
            'task_id': str(task.id),
            'status': task.status,
            'result': task.result,
            'error': task.error,
            'retries': task.retries,
        })
```

---

## Problem 4: Leaderboard API

**Task:** Build a real-time leaderboard. Users submit scores,
API returns top N players and a user's rank.

```python
# Redis Sorted Set approach (O(log N) operations)
from django.core.cache import cache

class LeaderboardService:
    KEY = 'leaderboard:global'
    
    def submit_score(self, user_id, score):
        """Update user's score (keeps highest)."""
        redis = cache.client.get_client()
        current = redis.zscore(self.KEY, user_id)
        if current is None or score > current:
            redis.zadd(self.KEY, {str(user_id): score})
    
    def get_top(self, n=10):
        """Get top N players with scores."""
        redis = cache.client.get_client()
        # ZREVRANGE: highest scores first
        results = redis.zrevrange(self.KEY, 0, n - 1, withscores=True)
        return [
            {'user_id': user_id.decode(), 'score': int(score), 'rank': i + 1}
            for i, (user_id, score) in enumerate(results)
        ]
    
    def get_rank(self, user_id):
        """Get a user's rank (1-indexed)."""
        redis = cache.client.get_client()
        # ZREVRANK: rank by highest score (0-indexed)
        rank = redis.zrevrank(self.KEY, str(user_id))
        if rank is None:
            return None
        score = redis.zscore(self.KEY, str(user_id))
        return {'user_id': user_id, 'rank': rank + 1, 'score': int(score)}
    
    def get_around_user(self, user_id, n=5):
        """Get n players above and below a user."""
        redis = cache.client.get_client()
        rank = redis.zrevrank(self.KEY, str(user_id))
        if rank is None:
            return []
        
        start = max(0, rank - n)
        end = rank + n
        
        results = redis.zrevrange(self.KEY, start, end, withscores=True)
        return [
            {'user_id': uid.decode(), 'score': int(score), 'rank': start + i + 1}
            for i, (uid, score) in enumerate(results)
        ]


# views.py
class LeaderboardView(APIView):
    service = LeaderboardService()
    
    def get(self, request):
        """GET /leaderboard/?top=10"""
        n = int(request.query_params.get('top', 10))
        return Response({'leaderboard': self.service.get_top(n)})
    
    def post(self, request):
        """POST /leaderboard/ {score: 1500}"""
        self.service.submit_score(request.user.id, request.data['score'])
        rank = self.service.get_rank(request.user.id)
        return Response(rank)


class MyRankView(APIView):
    service = LeaderboardService()
    
    def get(self, request):
        """GET /leaderboard/me/"""
        rank = self.service.get_rank(request.user.id)
        nearby = self.service.get_around_user(request.user.id, n=3)
        return Response({'my_rank': rank, 'nearby': nearby})
```

---

## Problem 5: Event Booking System (Concurrency)

**Task:** Build an event ticket booking API that handles concurrent requests
without double-booking. Limited seats per event.

```python
# models.py
class Event(models.Model):
    name = models.CharField(max_length=255)
    total_seats = models.PositiveIntegerField()
    available_seats = models.PositiveIntegerField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    event_date = models.DateTimeField()

class Booking(models.Model):
    class Status(models.TextChoices):
        CONFIRMED = 'confirmed'
        CANCELLED = 'cancelled'
    
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='bookings')
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    seats = models.PositiveIntegerField()
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.CONFIRMED)
    booked_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['event', 'user'],
                condition=models.Q(status='confirmed'),
                name='one_active_booking_per_user_event'
            )
        ]


# services.py
from django.db import transaction
from django.db.models import F

class BookingService:
    
    @staticmethod
    @transaction.atomic
    def book_seats(user, event_id, num_seats):
        """
        Atomically book seats with race-condition prevention.
        Uses select_for_update to lock the event row.
        """
        # Lock event row — other requests wait
        event = Event.objects.select_for_update().get(id=event_id)
        
        if event.available_seats < num_seats:
            raise ValueError(
                f'Only {event.available_seats} seats available, requested {num_seats}'
            )
        
        # Check duplicate booking
        if Booking.objects.filter(event=event, user=user, status='confirmed').exists():
            raise ValueError('You already have a booking for this event')
        
        # Decrease available seats
        event.available_seats -= num_seats
        event.save(update_fields=['available_seats'])
        
        # Create booking
        booking = Booking.objects.create(
            event=event,
            user=user,
            seats=num_seats,
            total_amount=event.price * num_seats,
        )
        
        return booking
    
    @staticmethod
    @transaction.atomic
    def cancel_booking(booking_id, user):
        """Cancel and release seats."""
        booking = Booking.objects.select_for_update().get(
            id=booking_id, user=user, status='confirmed'
        )
        
        booking.status = 'cancelled'
        booking.save(update_fields=['status'])
        
        # Release seats
        Event.objects.filter(id=booking.event_id).update(
            available_seats=F('available_seats') + booking.seats
        )
        
        return booking


# Alternative: Optimistic locking (higher throughput)
class BookingServiceOptimistic:
    
    @staticmethod
    def book_seats(user, event_id, num_seats, max_attempts=3):
        """
        Optimistic approach: try to update, retry on conflict.
        Better throughput than pessimistic locking for high traffic.
        """
        for attempt in range(max_attempts):
            event = Event.objects.get(id=event_id)
            
            if event.available_seats < num_seats:
                raise ValueError('Not enough seats')
            
            # Conditional update (only succeeds if seats haven't changed)
            updated = Event.objects.filter(
                id=event_id,
                available_seats=event.available_seats  # optimistic check
            ).update(
                available_seats=F('available_seats') - num_seats
            )
            
            if updated:
                # Success — create booking
                booking = Booking.objects.create(
                    event=event, user=user, seats=num_seats,
                    total_amount=event.price * num_seats,
                )
                return booking
            
            # Conflict — another request modified seats, retry
            continue
        
        raise ValueError('High demand. Please try again.')
```

---

## Problem 6: Notification System

**Task:** Build a notification system that supports multiple channels
(email, push, in-app) with user preferences.

```python
# models.py
class NotificationPreference(models.Model):
    user = models.OneToOneField('auth.User', on_delete=models.CASCADE)
    email_enabled = models.BooleanField(default=True)
    push_enabled = models.BooleanField(default=True)
    in_app_enabled = models.BooleanField(default=True)
    quiet_hours_start = models.TimeField(null=True)
    quiet_hours_end = models.TimeField(null=True)

class Notification(models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE, related_name='notifications')
    title = models.CharField(max_length=255)
    body = models.TextField()
    notification_type = models.CharField(max_length=50)  # order_update, promo, alert
    is_read = models.BooleanField(default=False)
    data = models.JSONField(default=dict)  # extra payload
    created_at = models.DateTimeField(auto_now_add=True)
    read_at = models.DateTimeField(null=True)


# services.py
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    @abstractmethod
    def send(self, user, title, body, data=None):
        pass

class EmailChannel(NotificationChannel):
    def send(self, user, title, body, data=None):
        send_email.delay(user.email, title, body)

class PushChannel(NotificationChannel):
    def send(self, user, title, body, data=None):
        send_push_notification.delay(user.id, title, body, data)

class InAppChannel(NotificationChannel):
    def send(self, user, title, body, data=None):
        Notification.objects.create(
            user=user, title=title, body=body,
            notification_type=data.get('type', 'general'), data=data or {}
        )


class NotificationService:
    channels = {
        'email': EmailChannel(),
        'push': PushChannel(),
        'in_app': InAppChannel(),
    }
    
    @classmethod
    def notify(cls, user, title, body, channels=None, data=None):
        """Send notification respecting user preferences."""
        prefs = NotificationPreference.objects.get_or_create(user=user)[0]
        
        if cls._is_quiet_hours(prefs):
            channels = ['in_app']  # only in-app during quiet hours
        
        active_channels = channels or ['email', 'push', 'in_app']
        
        for channel_name in active_channels:
            if channel_name == 'email' and not prefs.email_enabled:
                continue
            if channel_name == 'push' and not prefs.push_enabled:
                continue
            if channel_name == 'in_app' and not prefs.in_app_enabled:
                continue
            
            cls.channels[channel_name].send(user, title, body, data)
    
    @staticmethod
    def _is_quiet_hours(prefs):
        if not prefs.quiet_hours_start or not prefs.quiet_hours_end:
            return False
        from django.utils import timezone
        now = timezone.localtime().time()
        return prefs.quiet_hours_start <= now <= prefs.quiet_hours_end


# views.py
class NotificationListView(APIView):
    def get(self, request):
        notifications = Notification.objects.filter(
            user=request.user
        ).order_by('-created_at')[:50]
        
        unread_count = Notification.objects.filter(
            user=request.user, is_read=False
        ).count()
        
        return Response({
            'unread_count': unread_count,
            'notifications': NotificationSerializer(notifications, many=True).data,
        })

class MarkReadView(APIView):
    def post(self, request):
        ids = request.data.get('notification_ids', [])
        Notification.objects.filter(
            id__in=ids, user=request.user
        ).update(is_read=True, read_at=timezone.now())
        return Response({'status': 'ok'})
```

---

## Interview Tips for Coding Rounds

```
1. ASK CLARIFYING QUESTIONS first (2-3 min):
   - Expected scale? (affects DB choice, caching)
   - Auth required? (JWT vs session)
   - Real-time needed? (WebSocket vs polling)
   - Idempotency needed? (retry-safe?)

2. START with models + API contract (5 min):
   - Draw the data model
   - Define endpoints: POST /api/X, GET /api/X/{id}
   - Mention serializer briefly

3. IMPLEMENT core logic first (30-40 min):
   - Service layer with the main algorithm
   - Handle concurrency (select_for_update or F())
   - Write the view connecting everything

4. ADD edge cases (10-15 min):
   - Validation
   - Error responses
   - Race conditions
   - Pagination

5. MENTION but don't implement (5 min):
   - "I'd add caching here with Redis"
   - "This would be a Celery task in production"
   - "I'd add rate limiting middleware"
   - Show you think about production concerns
```
