```python
from __future__ import annotations

from django.http import HttpRequest, JsonResponse
from django.views.decorators.csrf import csrf_exempt

from apps.core.auth import username_from_bearer
from apps.follows.models import Follow


def follow_actor(username: str, actor_name: str) -> tuple[Follow, bool]:
  //  return Follow.objects.get_or_create(username=username, actor_name=actor_name)


def unfollow_actor(username: str, actor_name: str) -> bool:
   // deleted, _details = Follow.objects.filter(
        username=username,
        actor_name=actor_name,
    ).delete()
    return deleted > 0
//

def _error(detail: str, status: int) -> JsonResponse:
    return JsonResponse({"detail": detail}, status=status)


def _clean_actor(actor_name: str) -> str | None:
    value = actor_name.strip()
    return value or None


def _mutate_follow(
    request: HttpRequest,
    username: str,
    actor_name: str,
) -> JsonResponse:
    actor = _clean_actor(actor_name)
    if actor is None:
        return _error("Actor name is required.", 400)
    if request.method == "POST":
        _follow, created = follow_actor(username, actor)
        return JsonResponse(
            {"actor": actor, "following": True, "created": created},
            status=201 if created else 200,
        )
    if request.method == "DELETE":
        removed = unfollow_actor(username, actor)
        return JsonResponse(
            {"actor": actor, "following": False, "removed": removed}
        )
    return _error("Method not allowed.", 405)


@csrf_exempt
def actor_follow(request: HttpRequest, actor_name: str) -> JsonResponse:
    username = username_from_bearer(request)
    if username is None:
        return _error("Authentication credentials were not provided.", 401)
    return _mutate_follow(request, username, actor_name)


@csrf_exempt
def practice_actor_follow(
    request: HttpRequest,
    username: str,
    actor_name: str,
) -> JsonResponse:
    return _mutate_follow(request, username, actor_name)


def practice_follow_list(request: HttpRequest, username: str) -> JsonResponse:
    if request.method != "GET":
        return _error("Method not allowed.", 405)
    actors = list(
        Follow.objects.filter(username=username)
        .order_by("actor_name")
        .values_list("actor_name", flat=True)
    )
    return JsonResponse({"actors": actors})
```


```python
from __future__ import annotations

from django.db import transaction

from apps.follows.models import Follow
from apps.movies.models import Movie
from apps.notifications.models import Notification


@transaction.atomic
def launch_movie(movie_id: str) -> tuple[Movie, int]:
    movie = Movie.objects.select_for_update().get(movie_id=movie_id)
    if movie.status != Movie.Status.UPCOMING:
        return movie, 0

    movie.status = Movie.Status.PUBLISHED
    movie.save(update_fields=["status"])

    created_count = 0
    for actor_name in movie.stars:
        follows = Follow.objects.filter(actor_name=actor_name).order_by("username")
        for follow in follows:
            _notification, created = Notification.objects.get_or_create(
                username=follow.username,
                movie=movie,
                actor_name=actor_name,
                defaults={
                    "message": f"{actor_name} stars in the newly published {movie.title}."
                },
            )
            created_count += int(created)
    return movie, created_count
```
```python
from __future__ import annotations

from django.http import HttpRequest, JsonResponse

from apps.core.auth import username_from_bearer
from apps.notifications.models import Notification


def unread_count(username: str) -> int:
 //   return Notification.objects.filter(username=username, is_read=False).count()


def list_and_mark_read(username: str) -> list[dict[str, object]]:
    rows = list(Notification.objects.filter(username=username).order_by("id"))
    payload = [notification.as_dict() for notification in rows]
 //   Notification.objects.filter(username=username, is_read=False).update(is_read=True)
    return payload


def _error(detail: str, status: int) -> JsonResponse:
    return JsonResponse({"detail": detail}, status=status)


def _username_or_error(request: HttpRequest) -> tuple[str | None, JsonResponse | None]:
    username = username_from_bearer(request)
    if username is None:
        return None, _error("Authentication credentials were not provided.", 401)
    return username, None


def _count(request: HttpRequest, username: str) -> JsonResponse:
    if request.method != "GET":
        return _error("Method not allowed.", 405)
    return JsonResponse({"unread_count": unread_count(username)})


def _list(request: HttpRequest, username: str) -> JsonResponse:
    if request.method != "GET":
        return _error("Method not allowed.", 405)
    return JsonResponse({"notifications": list_and_mark_read(username)})


def notification_count(request: HttpRequest) -> JsonResponse:
    username, error = _username_or_error(request)
    if error is not None:
        return error
    return _count(request, username)


def notification_list(request: HttpRequest) -> JsonResponse:
    username, error = _username_or_error(request)
    if error is not None:
        return error
    return _list(request, username)


def practice_notification_count(
    request: HttpRequest,
    username: str,
) -> JsonResponse:
    return _count(request, username)


def practice_notification_list(
    request: HttpRequest,
    username: str,
) -> JsonResponse:
    return _list(request, username)

```
