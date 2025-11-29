# 2025-11-29 1회차 복습 챌린지

## 🧱 1️⃣ 기본 인덱스 (Primary Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | **Primary Key** 제약 시 자동 생성 인덱스 | `sql CREATE TABLE users ( id SERIAL PRIMARY KEY, name TEXT );` | `prisma model User { id Int @id @default(autoincrement()) name String }` | 엔티티 기준 repository에서 `userRepository.findOne({ where: { id } })` 시 자동 PK 인덱스 사용 |
| 특징 | 물리적 정렬 기준 아님 (단순 식별키) | btree 기본 사용 | Prisma 마이그레이션 시 자동 생성 | NestJS 쿼리 최적화 시 인덱스 힌트 불필요 |

---

## 🧱 2️⃣ 밀집 인덱스 (Dense Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | 모든 레코드에 인덱스 존재 | `CREATE INDEX idx_salary ON employees(salary);` | `@@index([salary])` | `findMany({ where: { salary: { gte: 3000 } } })` → full index scan |
| 특징 | 탐색 빠름 / 공간 크다 | PostgreSQL의 일반 인덱스 대부분 이 구조 | Prisma migration 시 자동 btree | NestJS Service 단에서 pagination 최적화 시 유용 |

> 💡핵심 기억 포인트: PostgreSQL의 인덱스 = 거의 전부 밀집 인덱스 + B-tree 구조
> 

---

## 🧱 3️⃣ 희소 인덱스 (Sparse / Partial Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | **일부 레코드**만 인덱싱 | `CREATE INDEX idx_active_user ON users(id) WHERE active = true;` | Prisma는 직접 지원X → `@@index` + raw SQL migration 사용 | `findMany({ where: { active: true } })` 호출 시 partial index 자동 사용 |
| 특징 | 조건이 붙는 쿼리 최적화 | 공간 절약, but 조건 일치해야 사용 | Prisma migration 단계에서 sql 쿼리 삽입 가능 | NestJS에서 SoftDelete/active flag 필드 있을 때 자주 씀 |

---

## 🧱 4️⃣ 클러스터링 인덱스 (Clustering Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | 인덱스 순서 == 데이터 저장 순서 | `CLUSTER users USING users_pkey;` | Prisma 직접 제어 불가 | ORM은 물리적 순서 알 수 없음, ORDER BY로 논리 정렬 |
| 특징 | 범위 검색 최적화, but 재정렬 비용 큼 | 주기적 재클러스터링 필요 | ORM 기반에서는 DB에 명시적 명령 필요 | NestJS에서 log용 대량 select 시 효율 상승 |

---

## 🧱 5️⃣ 함수 인덱스 (Functional Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | 함수/표현식 결과 인덱싱 | `CREATE INDEX idx_lower_name ON users (lower(name));` | Prisma 직접 지원X → SQL migration | `findMany({ where: { name: { equals: "kim", mode: "insensitive" } } })` → lower() 인덱스 필요 |
| 특징 | 대소문자/포맷 불문 검색 속도 향상 | 함수 기반 btree 가능 | Prisma에서는 raw SQL migration으로 처리 | NestJS에서 검색 조건이 함수일 경우 (예: LOWER) 유용 |

---

## 🧱 6️⃣ 커버링 인덱스 (Covering Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | 쿼리의 모든 컬럼이 인덱스에 포함되어 테이블 접근 생략 | `CREATE INDEX idx_user_cover ON users (id) INCLUDE (email, name);` | `@@index([id], map:"idx_user_cover")` + raw SQL로 INCLUDE 추가 | `findMany({ select: { id: true, email: true, name: true } })` → 테이블 접근 생략 가능 |
| 특징 | 디스크 I/O 절감 | PostgreSQL 11+ 지원 | ORM 레벨에서 직접 제어 불가 | 조회전용 API에 매우 효율적 |

---

## 🧱 7️⃣ 단일 인덱스 (Single Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | 단일 컬럼 인덱스 | `CREATE INDEX idx_email ON users(email);` | `@@index([email])` | `findUnique({ where: { email } })` 시 활용 |
| 특징 | 단순 검색 빠름 |  |  |  |

---

## 🧱 8️⃣ 복합 인덱스 (Composite Index)

| 구분 | 설명 | PostgreSQL | Prisma | NestJS 예시 |
| --- | --- | --- | --- | --- |
| 개념 | 여러 컬럼 묶음 | `CREATE INDEX idx_name_email ON users(name, email);` | `@@index([name, email])` | `findMany({ where: { name, email } })` 시 인덱스 타기 가능 |
| 특징 | **왼쪽부터 일치해야 인덱스 작동** | `(name)`만 검색은 가능, `(email)` 단독은 불가 | Prisma에서 필드 순서 중요 | NestJS 쿼리빌더도 WHERE 순서 주의 |

---

## 🔍 요약 흐름 (개발자 기억용)

```sql
CREATE INDEX idx_name ON users(name);      -- 단일
CREATE INDEX idx_nm_em ON users(name,email); -- 복합
CREATE INDEX idx_lower_nm ON users(lower(name)); -- 함수
CREATE INDEX idx_cover ON users(id) INCLUDE (email); -- 커버링
CREATE INDEX idx_active ON users(id) WHERE active=true; -- 희소
CLUSTER users USING users_pkey; -- 클러스터링

```

---

## 🧭 Prisma Schema로 연결 (예시 스키마)

```prisma
model User {
  id      Int     @id @default(autoincrement())
  name    String
  email   String  @unique
  salary  Int?
  active  Boolean @default(true)

  @@index([name, email]) // 복합 인덱스
  @@index([salary])      // 단일 인덱스
  // partial index, covering index는 migration.sql에 직접 작성
}

```

- 참고: prisma는 개념 스키마 (cf. `view`, `graphql`, `api` 등은 외부 스키마)

---

Migration 파일 예시 (`migrations/2025XXXXXX.sql`)

```sql
-- 함수 인덱스
CREATE INDEX idx_lower_name ON "User" (lower("name"));

-- 커버링 인덱스
CREATE INDEX idx_user_cover ON "User"("id") INCLUDE ("email","name");

-- 희소 인덱스
CREATE INDEX idx_active_user ON "User"("id") WHERE "active"=true;

```