```python
import json

from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

from .models import CarListing


def serialize_car(car):
    return {"_id": f"car-{car.id}", "ownerId": car.owner_id, "make": car.make, "model": car.model, "dailyPrice": car.daily_price, "availableFrom": car.available_from, "availableUntil": car.available_until, "active": car.active, "published": car.published}


def request_owner(request):
    owner_id = request.headers.get("X-User-Id", "")
    authorization = request.headers.get("Authorization", "")
    return owner_id or (authorization.removeprefix("Bearer ") if authorization.startswith("Bearer ") else "")


@csrf_exempt
def create_car(request):
    if request.method != "POST":
        return JsonResponse({"detail": "method not allowed"}, status=405)
    try:
        payload = json.loads(request.body or b"{}")
    except (TypeError, ValueError, json.JSONDecodeError):
        return JsonResponse({"detail": "invalid JSON"}, status=400)
    owner_id = request_owner(request)
    required = (payload.get("make"), payload.get("model"), payload.get("dailyPrice"), payload.get("availableFrom"), payload.get("availableUntil"))
    if not owner_id or any(value is None for value in required) or float(payload["dailyPrice"]) <= 0 or str(payload["availableFrom"]) > str(payload["availableUntil"]):
        return JsonResponse({"detail": "invalid listing"}, status=400)
    car = CarListing.objects.create(owner_id=owner_id, make=str(payload["make"]), model=str(payload["model"]), daily_price=float(payload["dailyPrice"]), available_from=str(payload["availableFrom"]), available_until=str(payload["availableUntil"]), active=bool(payload.get("active", True)), published=True)
    return JsonResponse(serialize_car(car), status=201)
```
