# Day 24: Product Reviews & Ratings

## Mục tiêu học tập
- Implement Review system với validation
- Tính toán và cache rating statistics
- Handle review moderation
- Implement helpful votes

---

## 1. Review System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    REVIEW SYSTEM                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      PRODUCT                              │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │ Rating Statistics (Cached)                          │ │   │
│  │  │ • Average Rating: 4.5 ⭐                            │ │   │
│  │  │ • Total Reviews: 128                                │ │   │
│  │  │ • Rating Distribution: 5⭐=80, 4⭐=30, 3⭐=10...    │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     REVIEWS LIST                          │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │ Review #1                              ⭐⭐⭐⭐⭐    │ │   │
│  │  │ User: John Doe (Verified Purchase ✓)               │ │   │
│  │  │ "Great product! Highly recommended..."             │ │   │
│  │  │ 👍 15 helpful | 📅 2 days ago                      │ │   │
│  │  │ [Images] [Reply from Seller]                       │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │ Review #2                              ⭐⭐⭐⭐      │ │   │
│  │  │ User: Jane Smith (Verified Purchase ✓)             │ │   │
│  │  │ "Good quality but shipping was slow..."            │ │   │
│  │  │ 👍 8 helpful | 📅 1 week ago                       │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Review Entities

### 2.1 Review Entity

```java
package com.ecommerce.domain.entity;

import com.ecommerce.domain.enums.ReviewStatus;
import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "reviews", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Review extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;  // Link to verified purchase

    @Column(nullable = false)
    private int rating;  // 1-5

    @Column(length = 200)
    private String title;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String content;

    @Column(name = "is_verified_purchase")
    private boolean verifiedPurchase = false;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ReviewStatus status = ReviewStatus.PENDING;

    @Column(name = "helpful_count")
    private int helpfulCount = 0;

    @Column(name = "not_helpful_count")
    private int notHelpfulCount = 0;

    @OneToMany(mappedBy = "review", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<ReviewImage> images = new ArrayList<>();

    @OneToMany(mappedBy = "review", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<ReviewVote> votes = new ArrayList<>();

    @OneToOne(mappedBy = "review", cascade = CascadeType.ALL, orphanRemoval = true)
    private ReviewReply sellerReply;

    @Column(name = "moderated_at")
    private LocalDateTime moderatedAt;

    @Column(name = "moderated_by")
    private Long moderatedBy;

    @Column(name = "rejection_reason")
    private String rejectionReason;

    // ========== Business Methods ==========

    public void approve(Long moderatorId) {
        this.status = ReviewStatus.APPROVED;
        this.moderatedAt = LocalDateTime.now();
        this.moderatedBy = moderatorId;
    }

    public void reject(Long moderatorId, String reason) {
        this.status = ReviewStatus.REJECTED;
        this.moderatedAt = LocalDateTime.now();
        this.moderatedBy = moderatorId;
        this.rejectionReason = reason;
    }

    public void incrementHelpful() {
        this.helpfulCount++;
    }

    public void decrementHelpful() {
        if (this.helpfulCount > 0) {
            this.helpfulCount--;
        }
    }

    public void incrementNotHelpful() {
        this.notHelpfulCount++;
    }

    public void decrementNotHelpful() {
        if (this.notHelpfulCount > 0) {
            this.notHelpfulCount--;
        }
    }

    public void addImage(ReviewImage image) {
        images.add(image);
        image.setReview(this);
    }

    public double getHelpfulPercentage() {
        int total = helpfulCount + notHelpfulCount;
        if (total == 0) return 0;
        return (double) helpfulCount / total * 100;
    }
}
```

### 2.2 ReviewImage Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "review_images")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ReviewImage extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "review_id", nullable = false)
    private Review review;

    @Column(name = "image_url", nullable = false)
    private String imageUrl;

    @Column(name = "thumbnail_url")
    private String thumbnailUrl;

    @Column(name = "display_order")
    private int displayOrder = 0;
}
```

### 2.3 ReviewVote Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "review_votes", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"user_id", "review_id"})
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ReviewVote extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "review_id", nullable = false)
    private Review review;

    @Column(name = "is_helpful", nullable = false)
    private boolean helpful;
}
```

### 2.4 ReviewReply Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "review_replies")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ReviewReply extends BaseEntity {

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "review_id", nullable = false, unique = true)
    private Review review;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "seller_id", nullable = false)
    private User seller;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String content;
}
```

### 2.5 ReviewStatus Enum

```java
package com.ecommerce.domain.enums;

public enum ReviewStatus {
    PENDING,    // Chờ duyệt
    APPROVED,   // Đã duyệt
    REJECTED,   // Từ chối
    HIDDEN      // Ẩn (vi phạm)
}
```

### 2.6 Product Rating Stats (Embedded)

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import lombok.*;

@Embeddable
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class RatingStats {

    @Column(name = "average_rating", precision = 2, scale = 1)
    private Double averageRating = 0.0;

    @Column(name = "total_reviews")
    private Integer totalReviews = 0;

    @Column(name = "rating_1_count")
    private Integer rating1Count = 0;

    @Column(name = "rating_2_count")
    private Integer rating2Count = 0;

    @Column(name = "rating_3_count")
    private Integer rating3Count = 0;

    @Column(name = "rating_4_count")
    private Integer rating4Count = 0;

    @Column(name = "rating_5_count")
    private Integer rating5Count = 0;

    public void addRating(int rating) {
        incrementRatingCount(rating);
        totalReviews++;
        recalculateAverage();
    }

    public void removeRating(int rating) {
        decrementRatingCount(rating);
        if (totalReviews > 0) {
            totalReviews--;
        }
        recalculateAverage();
    }

    public void updateRating(int oldRating, int newRating) {
        decrementRatingCount(oldRating);
        incrementRatingCount(newRating);
        recalculateAverage();
    }

    private void incrementRatingCount(int rating) {
        switch (rating) {
            case 1 -> rating1Count++;
            case 2 -> rating2Count++;
            case 3 -> rating3Count++;
            case 4 -> rating4Count++;
            case 5 -> rating5Count++;
        }
    }

    private void decrementRatingCount(int rating) {
        switch (rating) {
            case 1 -> { if (rating1Count > 0) rating1Count--; }
            case 2 -> { if (rating2Count > 0) rating2Count--; }
            case 3 -> { if (rating3Count > 0) rating3Count--; }
            case 4 -> { if (rating4Count > 0) rating4Count--; }
            case 5 -> { if (rating5Count > 0) rating5Count--; }
        }
    }

    private void recalculateAverage() {
        if (totalReviews == 0) {
            averageRating = 0.0;
            return;
        }

        double sum = rating1Count + (rating2Count * 2) + (rating3Count * 3)
                + (rating4Count * 4) + (rating5Count * 5);
        averageRating = Math.round(sum / totalReviews * 10.0) / 10.0;
    }

    public int getRatingPercentage(int rating) {
        if (totalReviews == 0) return 0;
        int count = switch (rating) {
            case 1 -> rating1Count;
            case 2 -> rating2Count;
            case 3 -> rating3Count;
            case 4 -> rating4Count;
            case 5 -> rating5Count;
            default -> 0;
        };
        return (int) Math.round((double) count / totalReviews * 100);
    }
}
```

---

## 3. Review DTOs

### 3.1 Request DTOs

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

@Data
public class CreateReviewRequest {

    @NotNull(message = "Product ID is required")
    private Long productId;

    private Long orderId;  // Optional for verified purchase

    @Min(value = 1, message = "Rating must be between 1 and 5")
    @Max(value = 5, message = "Rating must be between 1 and 5")
    private int rating;

    @Size(max = 200, message = "Title must be less than 200 characters")
    private String title;

    @NotBlank(message = "Review content is required")
    @Size(min = 10, max = 5000, message = "Review must be between 10 and 5000 characters")
    private String content;

    private List<String> imageUrls;  // Pre-uploaded images
}
```

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

@Data
public class UpdateReviewRequest {

    @Min(value = 1, message = "Rating must be between 1 and 5")
    @Max(value = 5, message = "Rating must be between 1 and 5")
    private Integer rating;

    @Size(max = 200)
    private String title;

    @Size(min = 10, max = 5000)
    private String content;
}
```

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class ReviewReplyRequest {

    @NotBlank(message = "Reply content is required")
    @Size(max = 2000, message = "Reply must be less than 2000 characters")
    private String content;
}
```

### 3.2 Response DTOs

```java
package com.ecommerce.dto.response;

import com.ecommerce.domain.enums.ReviewStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ReviewDto {

    private Long id;
    private UserSummary user;
    private Long productId;
    private int rating;
    private String title;
    private String content;
    private boolean verifiedPurchase;
    private ReviewStatus status;
    private int helpfulCount;
    private int notHelpfulCount;
    private List<ReviewImageDto> images;
    private ReviewReplyDto sellerReply;
    private LocalDateTime createdAt;
    private Boolean currentUserVote;  // null = not voted, true = helpful, false = not helpful

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class UserSummary {
        private Long id;
        private String firstName;
        private String lastName;
        private String avatarUrl;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ReviewImageDto {
        private Long id;
        private String imageUrl;
        private String thumbnailUrl;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ReviewReplyDto {
        private Long id;
        private String content;
        private String sellerName;
        private LocalDateTime createdAt;
    }
}
```

```java
package com.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class RatingStatsDto {

    private Double averageRating;
    private Integer totalReviews;
    private Map<Integer, Integer> distribution;  // rating -> count
    private Map<Integer, Integer> percentages;   // rating -> percentage
}
```

---

## 4. Review Service

### 4.1 ReviewService Interface

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.CreateReviewRequest;
import com.ecommerce.dto.request.ReviewReplyRequest;
import com.ecommerce.dto.request.UpdateReviewRequest;
import com.ecommerce.dto.response.RatingStatsDto;
import com.ecommerce.dto.response.ReviewDto;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ReviewService {

    ReviewDto createReview(CreateReviewRequest request);

    ReviewDto updateReview(Long reviewId, UpdateReviewRequest request);

    void deleteReview(Long reviewId);

    ReviewDto getReviewById(Long reviewId);

    Page<ReviewDto> getProductReviews(Long productId, Pageable pageable);

    Page<ReviewDto> getMyReviews(Pageable pageable);

    RatingStatsDto getProductRatingStats(Long productId);

    void voteReview(Long reviewId, boolean helpful);

    void removeVote(Long reviewId);

    // Moderation
    ReviewDto approveReview(Long reviewId);

    ReviewDto rejectReview(Long reviewId, String reason);

    Page<ReviewDto> getPendingReviews(Pageable pageable);

    // Seller reply
    ReviewDto addSellerReply(Long reviewId, ReviewReplyRequest request);
}
```

### 4.2 ReviewServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.*;
import com.ecommerce.domain.enums.OrderStatus;
import com.ecommerce.domain.enums.ReviewStatus;
import com.ecommerce.dto.request.CreateReviewRequest;
import com.ecommerce.dto.request.ReviewReplyRequest;
import com.ecommerce.dto.request.UpdateReviewRequest;
import com.ecommerce.dto.response.RatingStatsDto;
import com.ecommerce.dto.response.ReviewDto;
import com.ecommerce.exception.BadRequestException;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.mapper.ReviewMapper;
import com.ecommerce.repository.*;
import com.ecommerce.service.ReviewService;
import com.ecommerce.util.SecurityUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
public class ReviewServiceImpl implements ReviewService {

    private final ReviewRepository reviewRepository;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final ReviewVoteRepository voteRepository;
    private final ReviewMapper reviewMapper;

    @Override
    @Transactional
    public ReviewDto createReview(CreateReviewRequest request) {
        User currentUser = SecurityUtils.getCurrentUser()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        Product product = productRepository.findById(request.getProductId())
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", request.getProductId()));

        // Check if already reviewed
        if (reviewRepository.existsByUserIdAndProductId(currentUser.getId(), product.getId())) {
            throw new BadRequestException("You have already reviewed this product");
        }

        // Check verified purchase
        boolean verifiedPurchase = false;
        Order order = null;

        if (request.getOrderId() != null) {
            order = orderRepository.findById(request.getOrderId())
                    .filter(o -> o.getUser().getId().equals(currentUser.getId()))
                    .filter(o -> o.getStatus() == OrderStatus.DELIVERED)
                    .filter(o -> o.getItems().stream()
                            .anyMatch(item -> item.getProduct().getId().equals(product.getId())))
                    .orElse(null);

            verifiedPurchase = order != null;
        }

        // If no order specified, check if user has purchased
        if (!verifiedPurchase) {
            verifiedPurchase = orderRepository.hasUserPurchasedProduct(
                    currentUser.getId(), product.getId());
        }

        // Create review
        Review review = Review.builder()
                .user(currentUser)
                .product(product)
                .order(order)
                .rating(request.getRating())
                .title(request.getTitle())
                .content(request.getContent())
                .verifiedPurchase(verifiedPurchase)
                .status(ReviewStatus.PENDING)
                .build();

        // Add images
        if (request.getImageUrls() != null) {
            int order_idx = 0;
            for (String imageUrl : request.getImageUrls()) {
                ReviewImage image = ReviewImage.builder()
                        .imageUrl(imageUrl)
                        .displayOrder(order_idx++)
                        .build();
                review.addImage(image);
            }
        }

        review = reviewRepository.save(review);

        log.info("Review created for product {} by user {}",
                product.getId(), currentUser.getEmail());

        return reviewMapper.toDto(review, null);
    }

    @Override
    @Transactional
    public ReviewDto updateReview(Long reviewId, UpdateReviewRequest request) {
        Review review = getReviewAndValidateOwnership(reviewId);

        int oldRating = review.getRating();

        if (request.getRating() != null) {
            review.setRating(request.getRating());
        }
        if (request.getTitle() != null) {
            review.setTitle(request.getTitle());
        }
        if (request.getContent() != null) {
            review.setContent(request.getContent());
        }

        // Reset to pending if content changed
        if (review.getStatus() == ReviewStatus.APPROVED) {
            review.setStatus(ReviewStatus.PENDING);
        }

        review = reviewRepository.save(review);

        // Update product rating if rating changed
        if (request.getRating() != null && request.getRating() != oldRating) {
            updateProductRating(review.getProduct().getId());
        }

        return reviewMapper.toDto(review, getCurrentUserVote(reviewId));
    }

    @Override
    @Transactional
    public void deleteReview(Long reviewId) {
        Review review = getReviewAndValidateOwnership(reviewId);

        Product product = review.getProduct();
        int rating = review.getRating();

        reviewRepository.delete(review);

        // Update product rating
        if (review.getStatus() == ReviewStatus.APPROVED) {
            product.getRatingStats().removeRating(rating);
            productRepository.save(product);
        }

        log.info("Review {} deleted", reviewId);
    }

    @Override
    @Transactional(readOnly = true)
    public ReviewDto getReviewById(Long reviewId) {
        Review review = reviewRepository.findByIdWithDetails(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        return reviewMapper.toDto(review, getCurrentUserVote(reviewId));
    }

    @Override
    @Transactional(readOnly = true)
    public Page<ReviewDto> getProductReviews(Long productId, Pageable pageable) {
        Page<Review> reviews = reviewRepository.findByProductIdAndStatus(
                productId, ReviewStatus.APPROVED, pageable);

        return reviews.map(review ->
                reviewMapper.toDto(review, getCurrentUserVote(review.getId())));
    }

    @Override
    @Transactional(readOnly = true)
    public Page<ReviewDto> getMyReviews(Pageable pageable) {
        Long userId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        return reviewRepository.findByUserId(userId, pageable)
                .map(review -> reviewMapper.toDto(review, null));
    }

    @Override
    @Transactional(readOnly = true)
    public RatingStatsDto getProductRatingStats(Long productId) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", productId));

        RatingStats stats = product.getRatingStats();
        if (stats == null) {
            stats = new RatingStats();
        }

        Map<Integer, Integer> distribution = new HashMap<>();
        distribution.put(1, stats.getRating1Count());
        distribution.put(2, stats.getRating2Count());
        distribution.put(3, stats.getRating3Count());
        distribution.put(4, stats.getRating4Count());
        distribution.put(5, stats.getRating5Count());

        Map<Integer, Integer> percentages = new HashMap<>();
        for (int i = 1; i <= 5; i++) {
            percentages.put(i, stats.getRatingPercentage(i));
        }

        return RatingStatsDto.builder()
                .averageRating(stats.getAverageRating())
                .totalReviews(stats.getTotalReviews())
                .distribution(distribution)
                .percentages(percentages)
                .build();
    }

    @Override
    @Transactional
    public void voteReview(Long reviewId, boolean helpful) {
        User currentUser = SecurityUtils.getCurrentUser()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        Review review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        // Can't vote on own review
        if (review.getUser().getId().equals(currentUser.getId())) {
            throw new BadRequestException("You cannot vote on your own review");
        }

        Optional<ReviewVote> existingVote = voteRepository
                .findByUserIdAndReviewId(currentUser.getId(), reviewId);

        if (existingVote.isPresent()) {
            ReviewVote vote = existingVote.get();
            if (vote.isHelpful() == helpful) {
                return; // Same vote, no change
            }

            // Change vote
            if (vote.isHelpful()) {
                review.decrementHelpful();
                review.incrementNotHelpful();
            } else {
                review.decrementNotHelpful();
                review.incrementHelpful();
            }
            vote.setHelpful(helpful);
            voteRepository.save(vote);
        } else {
            // New vote
            ReviewVote vote = ReviewVote.builder()
                    .user(currentUser)
                    .review(review)
                    .helpful(helpful)
                    .build();
            voteRepository.save(vote);

            if (helpful) {
                review.incrementHelpful();
            } else {
                review.incrementNotHelpful();
            }
        }

        reviewRepository.save(review);
    }

    @Override
    @Transactional
    public void removeVote(Long reviewId) {
        Long userId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        Review review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        voteRepository.findByUserIdAndReviewId(userId, reviewId)
                .ifPresent(vote -> {
                    if (vote.isHelpful()) {
                        review.decrementHelpful();
                    } else {
                        review.decrementNotHelpful();
                    }
                    voteRepository.delete(vote);
                    reviewRepository.save(review);
                });
    }

    // ========== Moderation ==========

    @Override
    @Transactional
    public ReviewDto approveReview(Long reviewId) {
        Long moderatorId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        Review review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        if (review.getStatus() != ReviewStatus.PENDING) {
            throw new BadRequestException("Review is not pending");
        }

        review.approve(moderatorId);
        reviewRepository.save(review);

        // Update product rating
        Product product = review.getProduct();
        product.getRatingStats().addRating(review.getRating());
        productRepository.save(product);

        log.info("Review {} approved by moderator {}", reviewId, moderatorId);

        return reviewMapper.toDto(review, null);
    }

    @Override
    @Transactional
    public ReviewDto rejectReview(Long reviewId, String reason) {
        Long moderatorId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        Review review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        review.reject(moderatorId, reason);
        reviewRepository.save(review);

        log.info("Review {} rejected by moderator {}: {}", reviewId, moderatorId, reason);

        return reviewMapper.toDto(review, null);
    }

    @Override
    @Transactional(readOnly = true)
    public Page<ReviewDto> getPendingReviews(Pageable pageable) {
        return reviewRepository.findByStatus(ReviewStatus.PENDING, pageable)
                .map(review -> reviewMapper.toDto(review, null));
    }

    // ========== Seller Reply ==========

    @Override
    @Transactional
    public ReviewDto addSellerReply(Long reviewId, ReviewReplyRequest request) {
        User seller = SecurityUtils.getCurrentUser()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        Review review = reviewRepository.findByIdWithDetails(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        // Verify seller owns the product
        Product product = review.getProduct();
        if (!product.getSeller().getId().equals(seller.getId()) && !SecurityUtils.isAdmin()) {
            throw new BadRequestException("You can only reply to reviews on your products");
        }

        if (review.getSellerReply() != null) {
            throw new BadRequestException("Review already has a reply");
        }

        ReviewReply reply = ReviewReply.builder()
                .review(review)
                .seller(seller)
                .content(request.getContent())
                .build();

        review.setSellerReply(reply);
        reviewRepository.save(review);

        log.info("Seller reply added to review {}", reviewId);

        return reviewMapper.toDto(review, getCurrentUserVote(reviewId));
    }

    // ========== Private Methods ==========

    private Review getReviewAndValidateOwnership(Long reviewId) {
        Review review = reviewRepository.findById(reviewId)
                .orElseThrow(() -> new ResourceNotFoundException("Review", "id", reviewId));

        Long currentUserId = SecurityUtils.getCurrentUserId().orElse(null);
        if (!review.getUser().getId().equals(currentUserId) && !SecurityUtils.isAdmin()) {
            throw new BadRequestException("You can only modify your own reviews");
        }

        return review;
    }

    private Boolean getCurrentUserVote(Long reviewId) {
        return SecurityUtils.getCurrentUserId()
                .flatMap(userId -> voteRepository.findByUserIdAndReviewId(userId, reviewId))
                .map(ReviewVote::isHelpful)
                .orElse(null);
    }

    private void updateProductRating(Long productId) {
        // Recalculate from database
        Object[] stats = reviewRepository.calculateRatingStats(productId);
        if (stats != null && stats[0] != null) {
            Product product = productRepository.findById(productId).orElse(null);
            if (product != null) {
                RatingStats ratingStats = product.getRatingStats();
                if (ratingStats == null) {
                    ratingStats = new RatingStats();
                    product.setRatingStats(ratingStats);
                }
                // Would need to recalculate from review counts
                productRepository.save(product);
            }
        }
    }
}
```

---

## 5. Review Controller

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CreateReviewRequest;
import com.ecommerce.dto.request.ReviewReplyRequest;
import com.ecommerce.dto.request.UpdateReviewRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.RatingStatsDto;
import com.ecommerce.dto.response.ReviewDto;
import com.ecommerce.security.annotation.AdminOnly;
import com.ecommerce.service.ReviewService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/reviews")
@RequiredArgsConstructor
@Tag(name = "Reviews", description = "Product review APIs")
public class ReviewController {

    private final ReviewService reviewService;

    // ========== Public Endpoints ==========

    @GetMapping("/product/{productId}")
    @Operation(summary = "Get reviews for a product")
    public ResponseEntity<ApiResponse<Page<ReviewDto>>> getProductReviews(
            @PathVariable Long productId,
            @PageableDefault(sort = "createdAt", direction = Sort.Direction.DESC)
            Pageable pageable) {

        Page<ReviewDto> reviews = reviewService.getProductReviews(productId, pageable);
        return ResponseEntity.ok(ApiResponse.success(reviews));
    }

    @GetMapping("/product/{productId}/stats")
    @Operation(summary = "Get rating statistics for a product")
    public ResponseEntity<ApiResponse<RatingStatsDto>> getProductRatingStats(
            @PathVariable Long productId) {

        RatingStatsDto stats = reviewService.getProductRatingStats(productId);
        return ResponseEntity.ok(ApiResponse.success(stats));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get review by ID")
    public ResponseEntity<ApiResponse<ReviewDto>> getReview(@PathVariable Long id) {
        ReviewDto review = reviewService.getReviewById(id);
        return ResponseEntity.ok(ApiResponse.success(review));
    }

    // ========== Authenticated Endpoints ==========

    @PostMapping
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Create a review")
    public ResponseEntity<ApiResponse<ReviewDto>> createReview(
            @Valid @RequestBody CreateReviewRequest request) {

        ReviewDto review = reviewService.createReview(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(review, "Review submitted for approval"));
    }

    @PutMapping("/{id}")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Update own review")
    public ResponseEntity<ApiResponse<ReviewDto>> updateReview(
            @PathVariable Long id,
            @Valid @RequestBody UpdateReviewRequest request) {

        ReviewDto review = reviewService.updateReview(id, request);
        return ResponseEntity.ok(ApiResponse.success(review, "Review updated"));
    }

    @DeleteMapping("/{id}")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Delete own review")
    public ResponseEntity<ApiResponse<Void>> deleteReview(@PathVariable Long id) {
        reviewService.deleteReview(id);
        return ResponseEntity.ok(ApiResponse.success(null, "Review deleted"));
    }

    @GetMapping("/my")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Get my reviews")
    public ResponseEntity<ApiResponse<Page<ReviewDto>>> getMyReviews(Pageable pageable) {
        Page<ReviewDto> reviews = reviewService.getMyReviews(pageable);
        return ResponseEntity.ok(ApiResponse.success(reviews));
    }

    // ========== Voting ==========

    @PostMapping("/{id}/vote")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Vote on review helpfulness")
    public ResponseEntity<ApiResponse<Void>> voteReview(
            @PathVariable Long id,
            @RequestParam boolean helpful) {

        reviewService.voteReview(id, helpful);
        return ResponseEntity.ok(ApiResponse.success(null,
                helpful ? "Marked as helpful" : "Marked as not helpful"));
    }

    @DeleteMapping("/{id}/vote")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Remove vote from review")
    public ResponseEntity<ApiResponse<Void>> removeVote(@PathVariable Long id) {
        reviewService.removeVote(id);
        return ResponseEntity.ok(ApiResponse.success(null, "Vote removed"));
    }

    // ========== Seller Reply ==========

    @PostMapping("/{id}/reply")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Add seller reply to review")
    public ResponseEntity<ApiResponse<ReviewDto>> addSellerReply(
            @PathVariable Long id,
            @Valid @RequestBody ReviewReplyRequest request) {

        ReviewDto review = reviewService.addSellerReply(id, request);
        return ResponseEntity.ok(ApiResponse.success(review, "Reply added"));
    }

    // ========== Moderation (Admin) ==========

    @GetMapping("/pending")
    @AdminOnly
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Get pending reviews for moderation")
    public ResponseEntity<ApiResponse<Page<ReviewDto>>> getPendingReviews(Pageable pageable) {
        Page<ReviewDto> reviews = reviewService.getPendingReviews(pageable);
        return ResponseEntity.ok(ApiResponse.success(reviews));
    }

    @PostMapping("/{id}/approve")
    @AdminOnly
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Approve review")
    public ResponseEntity<ApiResponse<ReviewDto>> approveReview(@PathVariable Long id) {
        ReviewDto review = reviewService.approveReview(id);
        return ResponseEntity.ok(ApiResponse.success(review, "Review approved"));
    }

    @PostMapping("/{id}/reject")
    @AdminOnly
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Reject review")
    public ResponseEntity<ApiResponse<ReviewDto>> rejectReview(
            @PathVariable Long id,
            @RequestParam String reason) {

        ReviewDto review = reviewService.rejectReview(id, reason);
        return ResponseEntity.ok(ApiResponse.success(review, "Review rejected"));
    }
}
```

---

## 6. Review Repository

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.Review;
import com.ecommerce.domain.enums.ReviewStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface ReviewRepository extends JpaRepository<Review, Long> {

    boolean existsByUserIdAndProductId(Long userId, Long productId);

    @Query("SELECT r FROM Review r " +
           "LEFT JOIN FETCH r.user " +
           "LEFT JOIN FETCH r.images " +
           "LEFT JOIN FETCH r.sellerReply " +
           "WHERE r.id = :id")
    Optional<Review> findByIdWithDetails(@Param("id") Long id);

    Page<Review> findByProductIdAndStatus(Long productId, ReviewStatus status, Pageable pageable);

    Page<Review> findByUserId(Long userId, Pageable pageable);

    Page<Review> findByStatus(ReviewStatus status, Pageable pageable);

    @Query("SELECT AVG(r.rating), COUNT(r) FROM Review r " +
           "WHERE r.product.id = :productId AND r.status = 'APPROVED'")
    Object[] calculateRatingStats(@Param("productId") Long productId);

    @Query("SELECT r.rating, COUNT(r) FROM Review r " +
           "WHERE r.product.id = :productId AND r.status = 'APPROVED' " +
           "GROUP BY r.rating")
    java.util.List<Object[]> getRatingDistribution(@Param("productId") Long productId);
}
```

---

## 7. Bài tập thực hành

### Bài tập 1: Basic Reviews
- [x] Create Review entity với validation
- [x] Implement CRUD operations
- [x] Add verified purchase check

### Bài tập 2: Rating Stats
- [x] Create RatingStats embeddable
- [x] Update stats when review approved/deleted
- [x] Display rating distribution

### Bài tập 3: Moderation & Voting
- [x] Implement review moderation
- [x] Add helpful/not helpful voting
- [x] Implement seller reply

---

## 8. Checklist

- [ ] Create Review, ReviewImage, ReviewVote entities
- [ ] Create RatingStats embeddable
- [ ] Create ReviewReply entity
- [ ] Implement ReviewService
- [ ] Add verified purchase validation
- [ ] Implement voting system
- [ ] Create moderation API
- [ ] Implement seller reply
- [ ] Update product rating on review changes
- [ ] Test review flow

---

## Điều hướng

- [← Day 23: Payment Integration](./day-23-payment-integration.md)
- [Day 25: Search & Filtering →](./day-25-search-filtering.md)
- [Về trang chính](../00-overview.md)
