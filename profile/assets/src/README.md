# 다이어그램 원본

프로필의 구조 다이어그램 4개는 이 폴더의 SVG가 원본입니다.
PNG는 표시용이며, 상위 `assets/`에 같은 이름으로 있습니다.

**PNG를 쓰는 이유** — GitHub 조직 프로필 페이지는 mermaid를 렌더링하지 않습니다.
2026-08-20 확인 결과 mermaid 블록이 원본 코드로 그대로 노출됐습니다. 그래서 이미지로 굽습니다.

## 고치는 법

SVG를 직접 편집한 뒤 2배 배율로 다시 구우면 됩니다.

```bash
msedge --headless=new --disable-gpu --hide-scrollbars \
  --force-device-scale-factor=2 --window-size=<W>,<H> --virtual-time-budget=2500 \
  --screenshot=d1_이중전원계층.png d1_이중전원계층.svg
```

`W`·`H`는 각 SVG의 `viewBox` 값을 씁니다. 900×470 / 900×430 / 900×330 / 900×420 순입니다.

## 색 규율

제품 UI와 같은 규율을 따릅니다.

| 색 | 값 | 의미 |
|---|---|---|
| 적색 | `#ff5b5b` | 경고 — 즉시 행동 |
| 앰버 | `#f2a900` | 주의 — 미검증 · 성능저하 |
| 녹색 | `#57d9a3` | 실제 센서로 확인됨 |
| 시안 | `#4dd8e6` | 계측 판독값 |
| 회색 | `#8b9a9a` | 데이터 없음 |
