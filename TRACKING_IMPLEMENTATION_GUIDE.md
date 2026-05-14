# Live Tracking System — Implementation Guide

This guide details every code change required to add the multi-bus live tracking system to the existing Django project. 

## 1. Database Models (`myapp/models.py`)
Add the `BusLocation` model to store the GPS coordinates. Add this at the very bottom of the file.

```python
class BusLocation(models.Model):
    """
    Real-time tracking coordinates for a bus.
    """
    bus = models.ForeignKey(Bus, on_delete=models.CASCADE, related_name='locations')
    latitude = models.FloatField()
    longitude = models.FloatField()
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-updated_at']

    def __str__(self):
        return f"{self.bus.bus_number} - {self.latitude}, {self.longitude}"
```

## 2. API & View Routing (`myapp/urls.py`)
Add these URL paths to your `urlpatterns` list:

```python
    # Live Tracking Map Page
    path('track-bus-api/', views.track_bus_api, name='track_bus_api'),
    
    # Live Tracking Backend APIs
    path('api/bus/<int:bus_id>/update/', views.update_bus_location, name='api_update_location'),
    path('api/buses/locations/', views.get_all_buses_location, name='api_all_bus_locations'),
```

## 3. Backend Logic & APIs (`myapp/views.py`)
Add the following imports at the top of your `views.py` if they are not already there:
```python
from django.views.decorators.csrf import csrf_exempt
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from .models import BusLocation, Bus
```

Then, add these three new functions to handle the tracking map and the APIs:

```python
@login_required(login_url='/login/')
def track_bus_api(request):
    """Renders the HTML page containing the map"""
    return render(request, 'app1/track_bus_api.html')

@csrf_exempt
@api_view(['POST'])
@permission_classes([AllowAny])
def update_bus_location(request, bus_id):
    """API for the Flutter mobile app to push GPS coordinates to"""
    try:
        bus = Bus.objects.get(id=bus_id)
        lat = request.data.get('lat') or request.data.get('latitude')
        lng = request.data.get('lng') or request.data.get('longitude')
        if lat is None or lng is None:
            return Response({"error": "Latitude and longitude required"}, status=400)
        BusLocation.objects.create(bus=bus, latitude=lat, longitude=lng)
        return Response({"message": "Location updated", "bus_id": bus_id, "lat": lat, "lng": lng})
    except Bus.DoesNotExist:
        return Response({"error": "Bus not found"}, status=404)

@api_view(['GET'])
def get_all_buses_location(request):
    """API for the Web Dashboard to fetch all bus locations"""
    buses = Bus.objects.filter(is_active=True).prefetch_related('locations', 'assigned_drivers')
    bus_data = []
    for bus in buses:
        latest_location = bus.locations.first()

        route_code = 'N/A'
        try:
            driver = bus.assigned_drivers.filter(is_active=True).first()
            if driver and driver.assigned_route:
                route_code = driver.assigned_route.code
        except Exception:
            pass

        bus_data.append({
            'id': bus.id,
            'bus_number': bus.bus_number,
            'latitude': latest_location.latitude if latest_location else None,
            'longitude': latest_location.longitude if latest_location else None,
            'driver_name': bus.driver_name or 'Unassigned',
            'route_code': route_code,
            'last_updated': latest_location.updated_at.isoformat() if latest_location else None,
        })
    return Response({'success': True, 'buses': bus_data})
```

**Important Auth Fix for Mobile App:**
Ensure your `login_user` view and `register_user` view in `views.py` have the `@csrf_exempt` decorator above them. This allows the mobile app to log in without dealing with complex browser cookie management.

## 4. Frontend Map HTML (`myapp/templates/app1/track_bus_api.html`)
Create this new file inside your templates folder. This HTML file holds the Leaflet map and the side-panel UI. Copy the exact contents of `track_bus_api.html` from this project.

## 5. Frontend Javascript (`myapp/static/app1/js/live_tracking.js`)
Create this new Javascript file. This contains the polling mechanism that requests data from `/api/buses/locations/` every 5 seconds and moves the markers on the Leaflet map. Copy the exact contents of `live_tracking.js` from this project.

## 6. Sidebar Navigation & SPA Bypass (`myapp/templates/app1/base.html`)
Add the link to the new map page in the sidebar navigation:
```html
<a href="{% url 'track_bus_api' %}" data-full-load="true">
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M19 17h2c.6 0 1-.4 1-1v-3c0-.9-.7-1.7-1.5-1.9L18 10l-2-6H8l-2 6-2.5.1C2.7 10.3 2 11.1 2 12v3c0 .6.4 1 1 1h2"/>
        <circle cx="7" cy="17" r="2"/>
        <circle cx="17" cy="17" r="2"/>
    </svg>
    Track Bus
</a>
```

**CRITICAL FIX for `base.html` Javascript:**
Find the Javascript at the bottom of `base.html` that handles the `loadPage` AJAX navigation. You must update it so that the map page triggers a full page reload, otherwise the map scripts will not execute. Update the click listener like this:
```javascript
navLinks.forEach(link => {
    link.addEventListener('click', function(e) {
        // Full page load for tracking page
        if (this.dataset.fullLoad === 'true') {
            window.location.href = this.href;
            return;
        }
        e.preventDefault();
        const url = this.href;
        loadPage(url, true);
    });
});
```

## 7. Middleware Fix (`myapp/middleware.py`)
Ensure superusers aren't blocked from the admin panel by the custom middleware. Update `AdminRequiredMiddleware`:
```python
        if request.path.startswith('/admin/'):
            if not request.user.is_authenticated:
                return redirect('login_page')
            
            # Allow Django superusers and staff to access the admin panel freely
            if not (request.user.is_superuser or request.user.is_staff):
                try:
                    if request.user.userprofile.user_type != 'admin':
                        return redirect('dashboard')
                except:
                    return redirect('dashboard')
```

## 8. Template Naming Case Sensitivity
If hosting on Linux (like PythonAnywhere), file systems are case-sensitive. Ensure your `views.py` calls `render(request, 'app1/Login.html')` with a capital **L**, matching the exact uppercase/lowercase name of the `.html` file in your templates folder.
