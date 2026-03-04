# 그린컴퓨터아카데미_파이널프로젝트: 온라인 화장품 쇼핑몰 (Backend)

[기술 스택]

Backend: Java, Spring Boot, Spring Data JPA, QueryDSL, MySQL
Frontend: React, JavaScript, Tailwind CSS, Vite
Infra / 배포: AWS S3, AWS EC2, nohup 수동 배포
협업 도구: Git, GitHub, GitKraken

------------------------------------------------------------------------------------------------

[담당 기능]

리뷰 시스템

  리뷰 등록:
  텍스트 + 이미지 복합 데이터를 FormData(multipart/form-data)로 전송
  이미지는 AWS S3에 업로드 후 URL만 DB에 저장하는 이중 저장 전략 적용
  @RequestPart로 JSON 데이터와 파일을 분리 수신
  
  리뷰 수정:
  기존 이미지 유지 / 새 이미지 추가 / 삭제 대상을 구분하여 변경된 부분만 반영하는 상태 유지 수정 로직 구현
  전체를 덮어쓰지 않아 불필요한 DB 변경과 데이터 유실 가능성 최소화
  다중 테이블 동시 수정을 @Transactional 범위로 감싸 중간 실패 시 전체 롤백 처리
  서비스 레이어에서 작성자 본인 여부 검증 후 수정 허용
  
  리뷰 삭제 — Hard Delete → Soft Delete 전환
  초기 물리 삭제 방식에서 FK 제약 충돌 오류 반복 발생
  deleted(사용자 삭제 여부) / visible(관리자 노출 제어) 컬럼 분리로 논리 삭제 구조 전환
  일반 사용자 삭제 시 is_deleted = true, 관리자 부적절 리뷰 처리 시 is_visible = false로 권한별 분기 처리
  모든 조회 쿼리에 is_deleted = false AND is_visible = true 조건 적용
  
  리뷰 조회:
  평균 별점, 전체 리뷰 수, 별점별 분포, 긍정 리뷰 수 집계 쿼리 구현
  최신순 / 좋아요순 / 별점 높은순 / 낮은순 다중 정렬 지원
  Pageable + useSearchParams 연동으로 페이지네이션 구현
  
  리뷰 좋아요:
  DB 수준에서 review_id + from_user_id 복합 유니크 제약으로 중복 좋아요 방지
  좋아요 등록/취소 토글 메커니즘 구현
  본인 리뷰 좋아요 불가 정책 서비스 레이어에서 검증
  
  리뷰 댓글:
  리뷰와 유사한 구조로 CRUD 구현
  is_deleted / is_visible 컬럼으로 동일한 논리 삭제 정책 적용

------------------------------------------------------------------------------------------------

검색 기능 — 데이터 성격별 모델 분리

  최근 검색어:
  사용자별 검색 기록 독립 관리 (삭제해도 인기 검색어 통계에 영향 없음)
  비로그인 사용자도 UUID 기반 쿠키(GuestID)로 식별하여 검색 기록 관리 가능
  
  GuestIdFilter에서 모든 요청마다 GUEST_ID 쿠키 확인
  없으면 UUID 발급 후 쿠키로 저장, request attribute에 담아 컨트롤러에서 공통 사용
  
  
  인기 검색어:
  keyword를 자연키(PK)로 사용하는 전용 집계 테이블 분리
  검색할 때마다 새 행 추가 없이 count +1 / lastSearchedAt 갱신
  개인 기록 삭제와 완전히 독립된 구조로 데이터 간섭 제거
  
  ------------------------------------------------------------------------------------------------
  
관리자 회원 관리 — QueryDSL 동적 쿼리
  
  이름, 아이디, 가입일, 등급, 수신 동의 여부 등 다중 필터 조합 검색
  BooleanExpression 단위로 필터 로직 분리하여 null 반환 시 해당 조건 자동 무시
  핸드폰 뒷자리 검색(endsWith), 날짜 범위 검색(atStartOfDay ~ LocalTime.MAX) 구현
  회원 권한(USER / MANAGER / ADMIN) 실시간 변경 — PATCH 메서드로 RESTful 설계
  PageableExecutionUtils.getPage()로 조회 쿼리와 카운트 쿼리 분리 실행

------------------------------------------------------------------------------------------------

[주요 기술적 고민]

Soft Delete 전환: FK 충돌 오류 해소 사용자·관리자 정책 독립 운영
검색 모델 분리: 개인 기록과 집계 데이터 간섭 제거
비로그인 식별GuestIdFilter로 UUID 쿠키 자동 발급 및 관리
