```python
import json

from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

from .models import Post


def serialize_post(post):
    return {
        "_id": f"post-{post.id}",
        "title": post.title,
        "content": post.content,
        "excerpt": post.excerpt,
        "category": post.category,
        "tags": list(post.tags),
        "readTime": post.read_time,
        "author": {"_id": post.owner_id, "name": post.owner_id},
    }


def request_owner(request):
    owner_id = request.headers.get("X-User-Id", "")
    authorization = request.headers.get("Authorization", "")
    if not owner_id and authorization.startswith("Bearer "):
        owner_id = authorization.removeprefix("Bearer ")
    return owner_id


@csrf_exempt
def create_post(request):
    if request.method != "POST":
        return JsonResponse({"detail": "method not allowed"}, status=405)
    try:
        payload = json.loads(request.body or b"{}")
    except (TypeError, ValueError, json.JSONDecodeError):
        return JsonResponse({"detail": "invalid JSON"}, status=400)
    if not isinstance(payload, dict):
        return JsonResponse({"detail": "invalid JSON"}, status=400)
    missing = [
        field
        for field in ("title", "content", "category")
        if payload.get(field) is None
    ]
    owner_id = request_owner(request)
    if missing or not owner_id:
        return JsonResponse({"detail": "missing required field"}, status=400)
    post = Post.objects.create(
        owner_id=owner_id,
        title=payload["title"],
        content=payload["content"],
        excerpt="" if payload.get("excerpt") is None else payload["excerpt"],
        category=payload["category"],
        tags=[] if payload.get("tags") is None else payload["tags"],
        read_time=0 if payload.get("readTime") is None else payload["readTime"],
    )
    return JsonResponse(serialize_post(post), status=201)


def my_posts(request):
    owner_id = request_owner(request)
    posts = Post.objects.filter(owner_id=owner_id)
    return JsonResponse({"posts": [serialize_post(post) for post in posts]})


@csrf_exempt
def reset(request):
    Post.objects.all().delete()
    return JsonResponse({"reset": True})
```
