# Singit 서비스 통합 및 고도화 - Technical Docs

## 📖 Overview
기존 PHP(Slim, Laravel) 기반 분산 구조의 영상 플랫폼을  
FastAPI + React 기반 단일 구조로 통합하고,  
숏폼(Short-form) 중심 UX로 개편한 프로젝트입니다.

---

## 🎯 Key Improvements

- 백엔드 분산 구조 → FastAPI 기반 통합
- 숏폼 기반 영상 소비 UX 전환
- 스와이프 피드 구조 설계
- 페이징 전략 설계 (Cursor / Offset)
- 사용자 활동 기반 추천 기능 구현

---

## 🧠 Technical Highlights

### 1. 숏폼 UX와 페이징 전략
- 스와이프 UX에서 영상 순서 일관성 유지 문제 해결
- 정렬 방식에 따른 페이징 전략 분리

👉 [Pagination Strategy](./backend/pagination.md)

---

### 2. 백엔드 아키텍처 통합
- Slim / Laravel 기반 분산 구조 → FastAPI 통합 서버
- API 구조 단순화 및 유지보수성 개선

👉 [Backend Architecture](./backend/architecture.md)

---

### 3. 추천 시스템 설계
- 사용자 활동 기반 Rule-based 추천 로직
- 초기 서비스에 적합한 경량 설계

👉 [Recommendation Logic](./recommendation/logic.md)

---

## 📂 Structure
