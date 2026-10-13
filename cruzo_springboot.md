```java
package com.cruzo.controller;

import com.cruzo.dto.CreateReviewRequest;
import com.cruzo.entity.Car;
import com.cruzo.entity.Review;
import com.cruzo.repository.InMemoryCruzoRepository;
import com.cruzo.service.ReviewService;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

@RestController
public class ReviewController {
    private final ReviewService reviewService;
    private final InMemoryCruzoRepository repository;

    public ReviewController(
            ReviewService reviewService,
            InMemoryCruzoRepository repository) {
        this.reviewService = reviewService;
        this.repository = repository;
    }

    @GetMapping("/health")
    public Map<String, String> health() {
        return Map.of("status", "ok");
    }

    @GetMapping("/cars/{carId}/reviews")
    public Map<String, Object> getCarReviews(@PathVariable String carId) {
        try {
            reviewService.syncCarRating(carId);
            Car car = repository.getCar(carId);
            List<Review> reviews = reviewService.getCarReviews(carId);
            Map<String, Object> response = new LinkedHashMap<>();
            response.put("reviews", reviews);
            response.put("count", car.getReviewCount());
            response.put("average", car.getAverageRating());
            return response;
        } catch (IllegalArgumentException error) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, error.getMessage(), error);
        }
    }

    @PostMapping("/cars/{carId}/reviews")
    public Map<String, Object> createReview(
            @PathVariable String carId,
            @RequestBody CreateReviewRequest request) {
        try {
            Review review = reviewService.addReview(carId, request);
            reviewService.syncCarRating(carId);
            Car car = repository.getCar(carId);
            Map<String, Object> response = new LinkedHashMap<>();
            response.put("status", "success");
            response.put("review", review);
            response.put("count", car.getReviewCount());
            response.put("average", car.getAverageRating());
            return response;
        } catch (IllegalArgumentException error) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, error.getMessage(), error);
        }
    }

    @PostMapping("/practice/reset")
    public Map<String, String> resetPracticeData() {
        repository.reset();
        return Map.of("status", "reset");
    }
}

```



``` java
package com.cruzo.service;

import com.cruzo.dto.CreateReviewRequest;
import com.cruzo.entity.Car;
import com.cruzo.entity.Review;
import com.cruzo.repository.InMemoryCruzoRepository;
import java.util.List;
import org.springframework.stereotype.Service;

@Service
public class ReviewService {
    private final InMemoryCruzoRepository repository;

    public ReviewService(InMemoryCruzoRepository repository) {
        this.repository = repository;
    }

    public Review addReview(String carId, CreateReviewRequest request) {
        Car car = repository.getCar(carId);
        if (!repository.hasUser(request.reviewerId())) {
            throw new IllegalArgumentException("reviewer was not found");
        }
        if (request.reviewerId().equals(car.getOwnerId())) {
            throw new IllegalArgumentException("owners cannot review their own cars");
        }
        if (!repository.hasCompletedBooking(carId, request.reviewerId())) {
            throw new IllegalArgumentException("a completed booking is required");
        }
        Review review = repository.createReview(
                carId, request.reviewerId(), request.rating(), request.comment());
        return repository.saveReview(review);
    }

    public void syncCarRating(String carId) {
        repository.updateCarRating(carId);
    }

    public List<Review> getCarReviews(String carId) {
        repository.getCar(carId);
        return repository.reviewsForCar(carId);
    }
}
```

