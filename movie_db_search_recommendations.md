### Search
```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Iterable

from movies.models import Movie


@dataclass(frozen=True)
class AdvancedFilters:
    genres: tuple[str, ...] = ()
    rating_min: Decimal | None = None
    rating_max: Decimal | None = None
    year: int | None = None
    year_min: int | None = None
    year_max: int | None = None
    director: str = ""
    writer: str = ""


def basic_search(
    movies: Iterable[Movie], query: str, search_type: str
) -> list[Movie]:
    needle = query.casefold()

    def contains(value: object) -> bool:
        return needle in str(value).casefold()

    def matches(movie: Movie) -> bool:
        if search_type == "title":
            values = (movie.title,)
        elif search_type == "celebs":
            values = tuple(movie.stars)
        else:
            values = (
                movie.title,
                movie.director,
                *movie.writers,
                *movie.stars,
                movie.description,
            )
        return any(contains(value) for value in values)

    return [movie for movie in movies if movie.published and matches(movie)]


def advanced_search(
    movies: Iterable[Movie], filters: AdvancedFilters
) -> list[Movie]:
    selected_genres = {genre.casefold() for genre in filters.genres}
    director = filters.director.casefold()
    writer = filters.writer.casefold()

    def matches(movie: Movie) -> bool:
        movie_genres = {genre.casefold() for genre in movie.genres}
        return (
            movie.published
            and (not selected_genres or bool(movie_genres & selected_genres))
            and (filters.rating_min is None or movie.rating >= filters.rating_min)
            and (filters.rating_max is None or movie.rating <= filters.rating_max)
            and (filters.year is None or movie.year == filters.year)
            and (filters.year_min is None or movie.year >= filters.year_min)
            and (filters.year_max is None or movie.year <= filters.year_max)
            and (not director or director in movie.director.casefold())
            and (
                not writer
                or any(writer in value.casefold() for value in movie.writers)
            )
        )

    matching = [movie for movie in movies if matches(movie)]
    return sorted(
        matching,
        key=lambda movie: (-movie.rating, -movie.popularity, movie.movie_id),
    )
```

### Recommendations
```python
from movies.models import Movie, Profile


def _response_movie(
    movie: Movie,
    score: float,
    source: str,
    source_movie: Movie,
    user_rating: int | None,
) -> dict[str, object]:
    return {
        "_id": movie.movie_id,
        "title": movie.title,
        "year": movie.year,
        "rating": float(movie.rating),
        "genre": list(movie.genres),
        "description": movie.description,
        "popularity": movie.popularity,
        "type": movie.movie_type,
        "score": score,
        "source": source,
        "sourceMovie": source_movie.title,
        "userRating": user_rating,
    }


def recommendation_payload(profile: Profile) -> dict[str, object]:
    preferences = list(
        profile.preferences.select_related("movie").order_by("movie_id")
    )
    meaningful = [
        preference
        for preference in preferences
        if preference.rating is not None or preference.watched
    ]
    if not meaningful:
        return {
            "message": (
                "Start exploring movies by rating them or marking them as watched "
                "to get recommendations"
            ),
            "recommendations": [],
        }

    preference = meaningful[0]
    source_movie = preference.movie
    source_genres = {genre.casefold() for genre in source_movie.genres}
    known_movie_ids = {item.movie_id for item in meaningful}
    eligible = [
        movie
        for movie in Movie.objects.filter(published=True)
        if movie.movie_id not in known_movie_ids and movie.rating >= 7
    ]

    high_rating = preference.rating is not None and preference.rating > 5
    low_rating = preference.rating is not None and preference.rating <= 5

    def shares_genre(movie: Movie) -> bool:
        return bool(source_genres & {genre.casefold() for genre in movie.genres})

    if low_rating:
        eligible = [movie for movie in eligible if not shares_genre(movie)]
    elif preference.watched and preference.rating is None:
        eligible = [movie for movie in eligible if shares_genre(movie)]

    ranked = []
    for movie in eligible:
        multiplier = 1.2 if high_rating and shares_genre(movie) else 1.0
        score = float(movie.rating) / 10 * multiplier
        ranked.append((score, movie))
    ranked.sort(
        key=lambda item: (-item[0], -item[1].popularity, item[1].movie_id)
    )
    recommendations = [
        _response_movie(
            movie,
            score,
            "rated" if preference.rating is not None else "watched",
            source_movie,
            preference.rating,
        )
        for score, movie in ranked[:10]
    ]
    if recommendations:
        return {
            "message": "Personalized recommendations generated successfully.",
            "recommendations": recommendations,
        }

    return {
        "message": (
            "No recommendations found. Try rating more movies or marking some "
            "as watched."
        ),
        "recommendations": [],
    }
```
