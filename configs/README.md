# Spark 커널 설정

`spark.config`는 DGX Spark의 `/boot/config-7.0.0-1019-nvidia`를 그대로 기록한 빌드 시작 설정이다.

- Ubuntu source: `7.0.0-1019.19~24.04.2`
- Architecture: `arm64`, page size: `4096`
- SHA-256: `2f1e5c968baa001a34c47bf95dcf2b0b83e0cc93228c809637ef8035c5f50e23`

서명 키 경로와 개발 커널 접미사는 루트 README의 초기 설정 단계에서 출력 디렉터리의 `.config`에 적용한다. 실제 개인 키는 저장소 밖에 보관한다.
