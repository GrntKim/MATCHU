# REQ006 (E, 인프라)

| 항목 | 내용 |
| --- | --- |
| 대상 | Cloud SQL for PostgreSQL (`asia-northeast3`) |
| 추출 시점 | 2026-08-22, 운영 DB에서 `pg_dump --schema-only` |
| 적용 주체 | 관리자 1회 수행. **애플리케이션 계정(`app_user`)에는 DDL 권한을 부여하지 않는다** |

코드에 DDL을 포함하지 않는 것이 팀 규약이므로(REQ-006 INFRA-002-3), 본 문서가 스키마의 정본이다. 아래 문장을 순서대로 실행하면 동일한 구조를 재구축할 수 있다.

---

## 목차

| | |
| --- | --- |
| 1 | 확장 |
| 2 | 교육과정 성취기준 |
| 3 | 계정과 세션 |
| 4 | 생성 이력 |
| 5 | 권한 |
| 6 | 교육과정 버전 관리를 적용할 때 |

---

## 1. 확장

```sql
CREATE EXTENSION IF NOT EXISTS vector;    -- 임베딩 컬럼(1024차원)
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- gen_random_uuid()
```

---

## 2. 교육과정 성취기준

성취기준 코드 1개가 청크 1개에 대응한다.

```sql
CREATE TABLE curriculum_chunks (
    chunk_id           text NOT NULL,
    subject            text NOT NULL,
    grade_band         text NOT NULL,
    unit_name          text NOT NULL,
    domain             text NOT NULL,
    core_idea          text NOT NULL,
    achievement_code   text NOT NULL,
    achievement_text   text NOT NULL,
    explanation        text   DEFAULT ''::text  NOT NULL,
    inquiry_activities text[] DEFAULT '{}'::text[] NOT NULL,
    embedding          vector(1024),
    source_page        integer NOT NULL,
    CONSTRAINT curriculum_chunks_pkey PRIMARY KEY (chunk_id),
    CONSTRAINT curriculum_chunks_grade_band_check
        CHECK (grade_band IN ('G1_2', 'G3_4', 'G5_6')),
    CONSTRAINT curriculum_chunks_subject_check
        CHECK (subject IN ('MATH', 'SCIENCE', 'DOMESTIC_SCIENCE',
                           'ART', 'SOCIAL', 'KOREAN'))
);

CREATE UNIQUE INDEX idx_curriculum_code
    ON curriculum_chunks (achievement_code);

CREATE INDEX idx_curriculum_grade_band
    ON curriculum_chunks (grade_band);
```

**임베딩 컬럼에 벡터 인덱스를 두지 않는다.** 검색은 대상 학년군으로 필터한 뒤 전량을 조회해 애플리케이션에서 코사인 유사도를 계산하는 구조이며(REQ-002 RS-004), 코퍼스가 424행 규모라 인덱스 없이도 지연 목표를 충족한다. 코퍼스가 크게 늘면 `ivfflat` 또는 `hnsw` 도입을 재검토한다.

**`embedding`은 NULL을 허용한다.** 적재 중 임베딩 계산이 끝나지 않은 행이 존재할 수 있기 때문이며, 검색 시점에는 NULL 행이 후보에서 자연히 제외된다.

**차원은 임베딩 모델에 종속된다.** 현재 1024차원은 KoE5 기준이다. 모델을 바꾸면 컬럼 타입 변경과 전체 재적재가 함께 필요하다.

---

## 3. 계정과 세션

```sql
CREATE TABLE users (
    id            uuid DEFAULT gen_random_uuid() NOT NULL,
    email         text NOT NULL,
    password_hash text NOT NULL,
    name          text NOT NULL,
    role          text DEFAULT 'user'::text NOT NULL,
    created_at    timestamptz DEFAULT now() NOT NULL,
    default_grade smallint,
    CONSTRAINT users_pkey PRIMARY KEY (id),
    CONSTRAINT users_email_key UNIQUE (email),
    CONSTRAINT users_role_check
        CHECK (role IN ('user', 'admin')),
    CONSTRAINT users_default_grade_check
        CHECK (default_grade IS NULL OR default_grade BETWEEN 1 AND 6)
);

CREATE TABLE sessions (
    id         text NOT NULL,
    user_id    uuid NOT NULL,
    created_at timestamptz DEFAULT now() NOT NULL,
    expires_at timestamptz NOT NULL,
    CONSTRAINT sessions_pkey PRIMARY KEY (id),
    CONSTRAINT sessions_user_id_fkey FOREIGN KEY (user_id) REFERENCES users (id)
);
```

`default_grade`가 NULL이면 "설정 안 함"이며, 생성 화면의 학년 선택에 기본값을 넣지 않는다.

---

## 4. 생성 이력

```sql
CREATE TABLE lesson_requests (
    id                     uuid DEFAULT gen_random_uuid() NOT NULL,
    user_id                uuid NOT NULL,
    concept_name           text NOT NULL,
    target_grade           integer NOT NULL,
    subject_hint           text,
    mapped_curriculum_code text,
    lesson_output          jsonb NOT NULL,
    validation_status      text NOT NULL,
    created_at             timestamptz DEFAULT now() NOT NULL,
    deleted_at             timestamptz,
    CONSTRAINT lesson_requests_pkey PRIMARY KEY (id),
    CONSTRAINT lesson_requests_user_id_fkey FOREIGN KEY (user_id) REFERENCES users (id)
);

CREATE INDEX idx_lesson_requests_user_created
    ON lesson_requests (user_id, created_at DESC);
```

**`lesson_output`은 NOT NULL이다.** 생성에 실패한 실행도 이력으로 남으므로, 그 경우 빈 객체 `{}`를 저장한다. 마이페이지 렌더링과 재출력은 이 경우를 전제로 방어한다.

**`mapped_curriculum_code`는 `curriculum_chunks`를 참조하지만 외래키 제약을 걸지 않았다.** 교육과정이 개정되어 성취기준이 사라지면 과거 이력이 삭제되거나 갱신이 막히기 때문이다. 문자열 참조로 두고, 조회 실패는 애플리케이션에서 처리한다.

**삭제는 소프트 삭제다.** `deleted_at`을 채우고, 조회 시 `IS NULL` 조건으로 제외한다.

---

## 5. 권한

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

`DELETE`가 필요한 이유는 로그아웃과 만료 세션 정리에서 `sessions` 행을 삭제하기 때문이다. **DDL 권한(`CREATE`, `ALTER`, `DROP`)은 부여하지 않는다.**

---

## 6. 교육과정 버전 관리를 적용할 때 (향후 과제)

PRD 9.1의 설계를 적용하려면 아래를 함께 처리해야 한다. **스키마 확인 과정에서 새로 드러난 제약이 있어 기록한다.**

```sql
BEGIN;
SET LOCAL lock_timeout = '5s';

ALTER TABLE curriculum_chunks
    ADD COLUMN curriculum_version text NOT NULL DEFAULT '2022';
ALTER TABLE curriculum_chunks
    ALTER COLUMN curriculum_version DROP DEFAULT;

ALTER TABLE curriculum_chunks DROP CONSTRAINT curriculum_chunks_pkey;
ALTER TABLE curriculum_chunks
    ADD PRIMARY KEY (chunk_id, curriculum_version);

-- 아래가 새로 확인된 지점
DROP INDEX idx_curriculum_code;
CREATE UNIQUE INDEX idx_curriculum_code
    ON curriculum_chunks (achievement_code, curriculum_version);

COMMIT;
```

**`idx_curriculum_code`가 `achievement_code`에 대한 UNIQUE 인덱스다.** 기본키만 복합키로 바꾸고 이 인덱스를 그대로 두면, 같은 성취기준 코드를 가진 두 버전이 공존할 수 없다. 성취기준 코드 체계가 유지되는 한 두 번째 버전 적재는 이 인덱스에서 막힌다.

`DROP DEFAULT`는 안전장치다. 기본값을 남겨두면 새 교육과정 적재 시 값을 빠뜨려도 조용히 `'2022'`로 들어가 데이터가 섞인다. 기본값을 제거하면 그 경우 INSERT가 실패해 즉시 드러난다.

기존 행은 `DEFAULT '2022'`로 자동으로 채워지므로 재적재가 필요 없다. 모든 문장이 하나의 트랜잭션 안에 있으므로 중간에 실패하면 전부 되돌아간다.
