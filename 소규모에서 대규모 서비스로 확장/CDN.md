# CDN 정리

## 1) 개념
- **Content Delivery Network**  
  - 전 세계에 분산된 서버 네트워크를 통해 **정적/동적 콘텐츠를 사용자와 가까운 위치에서 제공**하는 기술  
  - 주요 목표: **지연시간 감소, 가용성↑, 트래픽 부하 분산, 보안 강화**

---

## 2) 동작 원리
1. 사용자가 특정 리소스(이미지, 동영상, JS, CSS 등)를 요청  
2. **DNS** 또는 **Anycast** 라우팅을 통해 가장 가까운 CDN 엣지 서버로 연결  
3. 엣지 서버에 캐시된 콘텐츠가 있으면 바로 전달 (Cache Hit)  
4. 없으면 원본 서버(Origin)에서 가져와 캐싱 후 전달 (Cache Miss)  

---

## 3) 캐시 전략
- **Static Content Caching**: 이미지, CSS, JS, 동영상 등 변경이 적은 파일  
- **Dynamic Content Caching**: API 응답, GraphQL, HTML 일부 → 짧은 TTL·헤더 기반 캐싱  
- **Cache Invalidation**: 새로운 버전 배포 시 기존 캐시 무효화 (API, CLI, purge)  
- **Versioning**: `style.v2.css`, `app.hash.js` 식으로 파일명에 버전 포함  

---

## 4) 주요 기능
- **Edge Location Routing**: 지리적으로 가까운 서버에서 응답  
- **DDoS 방어**: 대규모 트래픽 흡수·필터링  
- **TLS 종료**: HTTPS 인증서 관리 및 암호화 처리  
- **Access Control**: 서명된 URL/쿠키, IP 제한  
- **Edge Compute**: 간단한 연산/필터링/리다이렉션 (예: Cloudflare Workers, AWS Lambda@Edge)  

---

## 5) 장점
- **성능 향상**: Latency ↓, Throughput ↑  
- **확장성**: 트래픽 폭주 상황에서도 원본 서버 보호  
- **가용성**: 일부 서버 장애에도 글로벌 네트워크로 서비스 지속  
- **보안**: WAF, Bot 차단, TLS 처리  

---

## 6) 단점
- **비용**: 대규모 트래픽에서 요금 증가  
- **캐시 무효화 어려움**: 실시간 데이터 반영 지연  
- **복잡성**: 캐시 정책·헤더 관리 필요  

---

## 7) 활용 사례
- **정적 리소스**: 이미지, CSS, JS, 글꼴(fonts)  
- **대용량 미디어**: 동영상 스트리밍, 게임 다운로드  
- **API 가속**: REST/GraphQL 응답 캐싱, 지역별 최적화  
- **보안 게이트웨이**: SSL, DDoS 방어, Rate Limiting  

---

## 8) 대표 CDN 서비스
- **Cloudflare**  
- **AWS CloudFront**  
- **Akamai**  
- **Fastly**  
- **Google Cloud CDN**  

---

## 9) 운영 팁
- **캐시 헤더 관리**: `Cache-Control`, `ETag`, `Expires`, `Vary`  
- **버전 관리**: 파일명에 해시/버전 → 배포 시 캐시 자동 갱신  
- **Origin Shield**: 특정 지역 서버를 보호하는 계층 추가  
- **모니터링**: Cache Hit Ratio, Latency, Error Rate 추적  

---

## 10) 한 줄 요약
> CDN은 **전 세계 사용자에게 빠르고 안정적으로 콘텐츠를 제공**하기 위한 핵심 인프라로, 성능·보안·비용을 균형 있게 설계해야 한다.
