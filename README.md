# SRM demo page

정적 페이지 하나(`index.html`)와 오디오 폴더로 끝난다. 빌드 없음 — github.io 레포 루트(또는 `docs/`)에 그대로 올리면 동작한다.
편집할 곳은 `index.html` 안의 **`DEMO` 매니페스트 한 블록**뿐이다.

## 1. 페이지 구성

- 섹션: Remove · Reduce · Enhance · Boost · Normalize · Real-world
- 섹션 안에 샘플(scene)이 세로로 쌓이고, 샘플마다
  요소 선택 → prompt 문장 → **Input | Output | Target 스펙트로그램 3열**.
- 각 열에 재생 버튼이 있고 시계는 하나다. 재생 중 다른 열을 누르면 **같은 시점에서** 그 열로 넘어간다.
  `Space` 재생/정지, `1 2 3` 열 전환.
- no-op은 별도 섹션이 없다. 장면에 없는 요소도 점선 버튼으로 선택할 수 있고,
  고르면 "No-op — output should equal the input" 문구가 뜬다. 이때 Target 열은 input을 그대로 쓴다.
- "한 레코딩에 21개 prompt" 섹션은 코드에 남아 있다. 요소 다섯 개가 다 있는 scene이 준비되면
  `hero: { show:true, scene:"..." }`로 켜면 된다.

## 2. 폴더 규칙

```
index.html
audio/
  sim/<scene>/input.wav
  sim/<scene>/<Tnn>_output.wav
  sim/<scene>/<Tnn>_target.wav     # no-op task는 불필요 (target = input)
  real/<id>/video.mp4              # 오디오 트랙 없는 영상
  real/<id>/input.wav              # 원본 오디오
  real/<id>/out_<k>.wav            # outputs[k-1] 프롬프트의 SRM 출력
```

- 같은 샘플의 input / output / target은 **길이(샘플 수)가 같아야** 한다. 동기 재생과 스펙트로그램 정렬이 이것을 가정한다.
- 로컬 확인: `python -m http.server` 후 `http://localhost:8000/?check` —
  모든 샘플의 **선택 가능한 모든 조합**에 대해 없는 파일 목록이 우측 하단에 뜬다. (`file://`로 열면 fetch가 막힌다.)

## 3. 매니페스트

```js
scenes:   { s06: { elements:"MINER", event:"phone ringing", marks:[{t:2.4,label:"ring"}] } },
sections: { remove: [ {scene:"s06", select:["N","R"], prompts:{T07:"get rid of the noise and the reverb"}} ] }
```

| 필드 | 뜻 |
|---|---|
| `elements` | M 메인 화자 · I 방해 화자 · N 배경 잡음 · E 사건음 · R 잔향 (M은 자동 포함) |
| `event` | 사건음 캡션. prompt 문장과 버튼 이름에 쓰인다 |
| `select` | 처음 선택돼 있을 요소 |
| `prompts` | task별로 추론에 실제로 쓴 문장. 없으면 canonical 문장. 하이라이트는 자동 |
| `tasks` | 렌더한 task만 적으면 나머지 조합은 "not rendered"로 표시 (15개 remove 조합을 다 만들지 않을 때) |
| `note` | 샘플 아래 한 줄 설명 (선택) |

요소 선택 → task 매핑

| 섹션 | 선택 | task |
|---|---|---|
| Remove | I/N/E/R 중 1–4개 | T01–T15 |
| Reduce | Reverb | T19 |
| Enhance | (없음) | T18 |
| Boost | Main / Interferer / 둘 다 | T16 / T20 / T17 |
| Normalize | Interferer | T21 |

## 4. 샘플 고를 때

- Remove 샘플은 선택 가능한 조합이 최대 15개다. 전부 렌더하면 가장 좋고, 아니면 `tasks`로 좁힌다.
- no-op을 보여주고 싶은 섹션에는 해당 요소가 **없는** scene을 하나 넣는다 (예: Reduce에 잔향 없는 scene).
- Boost / Normalize는 레벨 변화가 결과 자체이므로 샘플별 loudness 정규화를 하지 않는다. 클리핑만 확인.
- 스펙트로그램 색 범위는 모든 파일에 공통 고정값(`DB_LO=-105`, `DB_HI=-20`)이다. 실제 샘플에서 너무 진하거나 옅으면 이 두 값만 조정.

## 5. Real-world 영상 작업

```bash
# 1) 구간 자르기 (10초 전후 권장)
ffmpeg -ss 00:01:02 -t 10 -i source.mp4 -c:v libx264 -crf 20 -c:a aac clip.mp4

# 2) 원본 오디오 추출 (44.1 kHz mono) → SRM 입력
ffmpeg -i clip.mp4 -vn -ac 1 -ar 44100 -c:a pcm_s16le audio/real/rw01/input.wav

# 3) SRM 추론 → audio/real/rw01/out_1.wav, out_2.wav ...

# 4) 페이지용 무음 영상 (오디오는 페이지가 동기 재생)
ffmpeg -i clip.mp4 -an -c:v libx264 -crf 23 -vf "scale=-2:720" -movflags +faststart audio/real/rw01/video.mp4

# (선택) 공유용: 결과 오디오를 영상에 입힌 파일
ffmpeg -i audio/real/rw01/video.mp4 -i audio/real/rw01/out_1.wav -map 0:v -map 1:a -c:v copy -c:a aac -shortest rw01_remastered.mp4
```

- 공개 페이지이므로 영상은 CC 라이선스이거나 직접 촬영한 것을 쓰고 `source`에 출처·라이선스를 적는다.
- GitHub 파일당 100 MB 제한이 있으므로 720p / CRF 23 정도로 둔다.
