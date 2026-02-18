> [!NOTE]
> 해당 저장소는 저의 주요 역할 및 트러블 슈팅을 정리하기 위한 요약본입니다.
>
> 전체적인 정보를 위해 원본을 보고 싶으시다면 [Flint](https://github.com/Edurican/Flint)를 클릭해주세요.

---

## 핵심 성과

> 동시 팔로우 시 발생하는 **데드락을 락 순서 고정으로 원천 차단**
> 
> `%keyword%` Full Table Scan을 `keyword%`로 전환하여 검색 성능 **약 30% 개선**

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [트러블 슈팅](#2-트러블-슈팅)

---

## 1. 프로젝트 개요

![Java](https://img.shields.io/badge/Java_17-ED8B00?style=flat&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot_3.5-6DB33F?style=flat&logo=springboot&logoColor=white)
![JPA](https://img.shields.io/badge/JPA-59666C?style=flat&logo=hibernate&logoColor=white)
![QueryDSL](https://img.shields.io/badge/QueryDSL-0769AD?style=flat)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=flat&logo=mariadb&logoColor=white)

> 인원: 4명 (백엔드 4) · 기간: 2025.11 ~ 2025.12 (1개월)
> 목적: 웹 서버에 대한 이해, 클라이언트-서버 관계 이해, 데이터베이스와의 상호작용 이해

**Flint**는 다양한 토픽의 짧은 글을 공유하는 SNS 프로젝트입니다. 좋아요, 조회수 등을 기반으로 인기 게시물이 핫 게시물로 선정되어 공유됩니다.

**주요 담당 역할:** 사용자 검색 및 팔로우/팔로잉 기능 구현

---

## 2. 트러블 슈팅

### 2-1. 동시 팔로우 시 데드락 → 락 순서 고정으로 원천 차단

**문제 발견**

두 사용자가 동시에 서로를 팔로우하면 정상 동작하지 않는 현상을 발견했습니다. 원인은 두 트랜잭션이 서로의 User row를 **역순으로 잠그면서 순환 대기(데드락)**가 발생하는 것이었습니다.

```
시간  TX-A (User 1 → User 2 팔로우)       TX-B (User 2 → User 1 팔로우)
──────────────────────────────────────────────────────────────────────
 t1   SELECT ... FOR UPDATE (User 1) 🔒
 t2                                        SELECT ... FOR UPDATE (User 2) 🔒
 t3   SELECT ... FOR UPDATE (User 2)
      ← User 2는 TX-B가 보유 중 → 대기
 t4                                        SELECT ... FOR UPDATE (User 1)
                                           ← User 1은 TX-A가 보유 중 → 대기
      ──── 순환 대기: TX-A ↔ TX-B 서로를 영원히 기다림 → 데드락 ────
```

**해결 방안 검토**

| 방안 | 문제점 | 선택 |
|------|--------|:----:|
| 낙관적 락 | 충돌 시 재시도 로직 필요. 동시 팔로우는 충분히 발생 가능한 시나리오라 재시도 비용이 비효율적. 낙관적 락 자체가 데드락을 방지하지 않음 | ❌ |
| 비관적 락 (순서 미고정) | 역순 잠금 시 데드락 여전히 발생 | ❌ |
| **비관적 락 + 락 순서 고정** | **순환 대기 원천 차단, 정합성 보장** | ✅ |

핵심은 **어떤 락을 쓰느냐가 아니라, 락을 잡는 순서를 고정하는 것**이었습니다. 양쪽 user id 중 작은 id를 항상 먼저 락 획득하면, 어떤 조합이든 잠금 순서가 동일해져서 순환 대기가 구조적으로 불가능합니다.

```
해결 후:
TX-A (User 1 → User 2): min(1,2)=1 → Lock(User 1) → Lock(User 2)
TX-B (User 2 → User 1): min(1,2)=1 → Lock(User 1) → Lock(User 2)

→ 두 TX 모두 User 1을 먼저 잠그므로, 한쪽이 대기하면 다른 쪽이 완료 후 해제
→ 순환 대기 불가능
```

**핵심 코드**

```java
// FollowService.java
@Transactional
public void follow(Long followerId, Long followingId) {
    // 항상 작은 ID를 먼저 락 획득 → 순환 대기 원천 차단
    Long firstId = Math.min(followerId, followingId);
    Long secondId = Math.max(followerId, followingId);

    User firstUser = userRepository.findByIdWithLock(firstId)   // SELECT ... FOR UPDATE
            .orElseThrow(() -> new CoreException(ErrorType.USER_NOT_FOUND));
    User secondUser = userRepository.findByIdWithLock(secondId) // SELECT ... FOR UPDATE
            .orElseThrow(() -> new CoreException(ErrorType.USER_NOT_FOUND));

    if (followRepository.existsByFollowerIdAndFollowingId(followerId, followingId)) {
        throw new CoreException(ErrorType.ALREADY_FOLLOWING);
    }
    // ...카운트 증가 및 팔로우 저장
}
```

```java
// UserRepository.java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT u FROM User u WHERE u.id=:id")
Optional<User> findByIdWithLock(@Param("id") Long id);
```

*동시 팔로우 테스트: 데드락 없이 정상 처리*

---

### 2-2. 검색 성능 개선 → LIKE 전략 변경으로 약 30% 개선

**문제 발견**

유저 데이터 증가에 따라 검색이 느려져 실행 계획을 확인한 결과, `%keyword%`가 인덱스를 활용하지 못하고 **Full Table Scan**을 유발하고 있었습니다.

**대안 검토**

| 방안 | 적합성 | 선택 |
|------|--------|:----:|
| `%keyword%` (현행) | 인덱스 사용 불가, Full Table Scan | ❌ |
| Full-Text Index | 불규칙적인 사용자 이름 특성상 토큰 분리 부적합 | ❌ |
| `%keyword` | 후방 일치, 인덱스 미사용 | ❌ |
| **`keyword%`** | **접두사 매칭으로 인덱스 Range Scan 가능** | ✅ |

**결과**

`keyword%`로 전환하여 Index Range Scan이 적용되면서 검색 성능 약 30% 개선.

```java
// FollowRepositoryCustomImpl.java
builder.and(user.username.like(username + "%"));  // keyword% → Index Range Scan
```

<img width="1204" height="228" alt="개선 전: Full Table Scan" src="https://github.com/user-attachments/assets/301d8a9f-5c9f-4492-8c17-524cdd32cb46" />

*개선 전: `%keyword%` — Full Table Scan*

<img width="1210" height="159" alt="개선 후: Index Range Scan" src="https://github.com/user-attachments/assets/cdb9f852-7127-4dc6-8e7b-f3c8e4343be5" />

*개선 후: `keyword%` — Index Range Scan*

---

### 2-3. 커서 기반 페이지네이션으로 대량 데이터 조회 안정화

Offset 방식은 `OFFSET 1000 LIMIT 20` 시 DB가 1,020개를 읽고 1,000개를 버리는 구조로, 뒤 페이지일수록 선형적으로 느려집니다. SNS 특성상 팔로워가 많은 유저의 목록 조회 시 병목이 우려되고, 실시간 데이터 추가/삭제 시 중복·누락 문제도 존재합니다.

마지막 조회 ID를 커서로 사용하는 `WHERE id < lastFetchedId ORDER BY id DESC LIMIT n` 방식으로 전환하여, 페이지 위치와 무관하게 항상 인덱스 범위 스캔으로 일정한 쿼리 성능을 유지하도록 개선했습니다.
