---
layout: post
title: Timecode-driven Multi-source Synchronization
subtitle: ''
excerpt_image: https://imucoding.github.io/assets/images/multi-sync.png
author: Hyunjune
categories: media
tags: [timecode, sync, multi-source, playout]
---
{% raw %}
## Environment
![img.png](https://imucoding.github.io/assets/images/multi-sync.png)
- 4개의 테라덱은 각 소스 스트림을 송출함
- 각 테라덱은 부팅 시점에 pts를 0으로 초기화하므로 4개의 테라덱이 정확히 동시에 켜지지 않는 이상 동일 프레임에 대해 pts가 상이함
- 타임 코드는 동일 ntp 서버 기준으로 동일 프레임에 대해 같은 타임코드로 삽입됨
- 네트워크를 통해 오는 각 스트림은 도착 속도가 다 제각각이므로 동일 프레임이 동일 타이밍에 수신되지 않음, 즉 수신하는 프레임의 타임코드가 정렬되어 있지 않음

## Troubleshooting
![img.png](https://imucoding.github.io/assets/images/multi-sync2.png)
1. 모든 스트림을 동시에 로컬 .ts 파일로 녹화 시작 (버퍼링)
2. 녹화와 병렬로 각 스트림의 첫 SEI timecode를 추출
3. 가장 느린(timecode가 큰) 스트림 기준 + 5초 = target_sec 결정
4. 각 버퍼 파일에서 TS 패킷 레벨로:
    - PAT/PMT 파싱하여 비디오/오디오 PID 식별
    - SEI timecode가 target_sec:fnum=0 인 프레임 탐색
    - 해당 프레임의 PTS를 base_pts로 설정
    - base_pts를 빼서 PTS/DTS/PCR을 0부터 리맵
    - Named FIFO → ffmpeg → SRT caller MPTS 출력


SEI는 h26x 코덱의 NALU type이므로 비디오 스트림에 대해서 SEI 패킷을 찾고 타임코드를 파싱하면 됨
- TS 패킷(188byte)을 이어 붙여서 PES를 만드는데, TS 헤더의 PUSI(Payload Unit Start Indicator) 비트가 1이면 그때부터 새로운 PES 시작 (2번째 바이트 내 존재)

```
[ 188 Bytes TS Packet ]
│
├─▶ 1. TS Header (기본 4 Bytes)
│    ├─ [0] Sync Byte: 항상 0x47 (알파벳 'G')
│    ├─ [1] Error(1bit) | ★PUSI(1bit) | Priority(1bit) | PID 상위 5bits
│    ├─ [2] PID 하위 8bits
│    └─ [3] Scrambling(2bits) | ★AFC(2bits) | Continuity Counter(4bits)
│
├─▶ 2. Adaptation Field (옵션, 가변 길이)
│    └─ AFC(Adaptation Field Control) 값에 따라 존재 여부 결정됨.
│       (PCR 클럭 동기화 데이터나 패킷 패딩을 위해 쓰임)
│
└─▶ 3. TS Payload (실제 데이터, 최대 184 Bytes)
     │
     ▼ ★ PUSI가 1일 때, 페이로드의 첫 바이트부터 무조건 PES 헤더 시작!
     [ PES Header ]
     ├─ [0] 0x00 \
     ├─ [1] 0x00  > Packet Start Code Prefix (고정값)
     ├─ [2] 0x01 /
     ├─ [3] Stream ID (예: 0xE0 = 비디오)
     ├─ [4~5] PES Packet Length (비디오는 보통 0x0000)
     ├─ [6~8] Optional Header (여기에 PTS/DTS 플래그가 있음)
     ├─ [9~13] ★ PTS (5 Bytes, 90kHz 하드웨어 틱)
     └─ [14~18] ★ DTS (존재할 경우 5 Bytes)
```

그리고 PES 역시 헤더와 페이로드로 구성되고 구조는 아래와 같음

```
[ PES Packet ]
 ├── [ PES Header ] (운송장: 언제 재생할 것인가?)
 │    ├── Start Code Prefix (0x000001) : "여기서부터 PES 시작이다"
 │    ├── Stream ID : 이 데이터가 비디오인지 오디오인지 식별 (예: 0xE0 = 비디오)
 │    ├── PES Packet Length : 패킷 길이 (비디오는 보통 0으로 세팅됨)
 │    └── Optional Header : ★ PTS와 DTS가 기록되는 핵심 구역
 │
 └── [ PES Payload ] (내용물: 무엇을 그릴 것인가?)
      │
      └── [ Access Unit (AU) ] ★ 1개의 PES 페이로드 = 보통 1개의 AU (프레임 1장)
           ├── 1. AUD NALU (Access Unit Delimiter) : "프레임 조립 시작!"
           ├── 2. SPS / PPS NALU : 해상도, 프로파일 등 코덱 메타데이터
           ├── 3. SEI NALU : 앞서 다뤘던 타임코드 등 부가 정보 메타데이터
           └── 4. VCL NALU : 실제 화면을 구성하는 픽셀 데이터 (I/P/B 슬라이스)
```

- 이렇게 PES안에 다양한 NALU가 들어있음 이 NALU 들의 집합으로 1프레임을 이루는 데이터가 AU이고, PES 페이로드를 AU로 봐도 무방
- PTS/DTS는 PES 헤더에 기록되어있고, PES 헤더는 오직 PUSI가 켜진 패킷에만 존재하므로 PTS추출을 위해서는 기타 패킷에서 페이로드를 볼 필요 없고, PUSI를 확인해야함

### 스트림 정보 추출
```
ffprobe -v error -select_streams v:0 -show_frames -show_entries frame=pts,tags -print_format json -read_intervals %+#[프레임수] -i [파일경로]
```
- `-select_streams` : 멀미플렉싱된 TS 안에서 특정 PID로 디먹싱 고정
- `-show_frames` : deep parsing 모드로 패킷 헤더만 보는 것이 아니라 AU 내부로 진입해서 NALU도 확인
- `-show_entries frame=pts,tags` : 데이터 필터링으로 PES 헤더의 pts와 SEI 파싱 결과가 담길 tags 바구니 가져옴
- `-print_format json` : json 포맷으로 직렬화
- `-read_intervals %+#[N]` : 파일 전체 스캔하지 않고, 시작점부터 딱 N개 프레임만 읽음

tags는 모든 메타데이터를 담는 바구니이므로, SEI뿐만아니라 다른 데이터도 담김

### Phase 1: 로컬 버퍼 녹화 시작
```python
    def start_recording(self):
        log.info("Phase 1: Recording all streams to local buffer...")

        for st in self.states:
            srt_url = _srt_in_url(st.port)
            cmd = [
                "ffmpeg", "-y",
                "-loglevel", "warning",
                "-fflags", "+igndts",
                "-rtbufsize", "2G",
                "-thread_queue_size", "8192",
                "-i", srt_url,
                "-c", "copy",                # 디코딩 없이 비트스트림 그대로 복사
                "-f", PASSTHROUGH_FORMAT,
                st.buffer_path,              # 로컬 버퍼 파일 (예: buffer_0.ts)
            ]
            rec = FFmpegProc(f"rec-{st.name}", cmd)
            rec.start()
            self._recorders.append(rec)

        # 파일이 생기도록 잠시 대기
        time.sleep(2)
```

### Phase 2: 첫 번째 프레임의 timecode 추출
```python
    def probe_timecodes(self, max_retries=20, interval=2.0):
        log.info("Phase 2: Extracting timecodes from buffered streams...")

        threads = []
        for st in self.states:
            # 4개의 스트림을 동시에 찔러보기 위해 스레드 생성
            t = threading.Thread(
                target=self._probe_one, args=(st, max_retries, interval),
                daemon=True,
            )
            threads.append(t)
            t.start()
            
        for t in threads:
            t.join() # 모든 스트림의 타임코드를 찾을 때까지 대기

    def _probe_one(self, st: StreamState, retries: int, interval: float):
        for attempt in range(1, retries + 1):
            # ffprobe를 통해 SEI 타임코드 추출 (tags.timecode)
            tc = extract_timecode(st.buffer_path, st.name) 
            if tc:
                st.timecode = tc  # 찾은 절대 시간(Timecode) 저장
                st.fps = get_fps(st.buffer_path)
                st.ready = True
                return
            time.sleep(interval)
```

### Phase 3: 동기화 타겟 계산
```python
   def compute_alignment(self):
        log.info("Phase 3: Computing alignment target...")

        # 가장 느린(타임코드가 큰) 스트림을 Reference로 선정
        ref = max(self.states, key=lambda s: s.timecode.total_seconds())
        
        # Reference 시간에 TARGET_MARGIN_SEC (예: 5초)를 더해 타겟 설정
        self._target_sec = ref.timecode.total_seconds() + TARGET_MARGIN_SEC

        for st in self.states:
            skip = self._target_sec - st.timecode.total_seconds()
            log.info(f"[{st.name}] must fast-forward ~{skip}s to reach target")
```

### Phase 4: 타겟 프레임 탐색 및 패킷 조작 
```python
def _tail_aligned(self, st: StreamState, fifo_path: str, ready_event: threading.Event):
        # 1. ffprobe로 타겟 타임코드(예: 12:00:07:00) 프레임이 버퍼에 쓰여질 때까지 기다렸다가 PTS 추출
        target_pts = self._find_target_pts_ffprobe(st) 
        base_pts = target_pts
        st.base_pts = base_pts

        reader = TSPacketReader(st.buffer_path, self._shutdown)
        reader.open()

        # 2. 버퍼를 읽으며 정확히 target_pts를 가진 프레임까지 건너뜀(Skip)
        for pkt in reader:
            # ... (PAT/PMT 파싱으로 비디오 PID 찾는 로직 생략) ...
            if pid == v_pid and pusi:
                pts, _ = pes_get_ts(ts_payload(pkt))
                if pts == target_pts:
                    target_pkt = pkt
                    break # 동기화 시작점 발견!

        ready_event.set() # 메인 스레드에 준비 완료 알림

        # 3. FIFO를 열고, 이후의 패킷들을 리맵핑하여 기록
        with open(fifo_path, "wb", buffering=0) as dst:
            dst.write(remap_pkt(target_pkt, base_pts)) # 첫 프레임 기록
            
            for pkt in reader:
                # 비디오 관련 PID(PAT, PMT, Video)만 통과
                if pid in allowed_pids:
                    # 핵심: pkt의 타임스탬프에서 base_pts를 깎아내어 0으로 만듦
                    remapped_pkt = remap_pkt(pkt, base_pts) 
                    dst.write(remapped_pkt)
```
- TS 스트림은 PAT, PMT, stream 으로 분류
    - PAT를 통해 PMT를 찾아야 그 안에서 오디오, 비디오 스트림 분류 가능


### Phase 5: MPEGTS 멀티플렉싱 및 최종 송출
```python
def start_forwarding(self):
        log.info("Phase 4: TS-level alignment + PTS remap via FIFO...")

        # 1. 프로세스 간 통신을 위한 Named FIFO 생성
        for i in range(len(self.states)):
            os.mkfifo(fifo_paths[i])

        # 2. 4개의 각 스트림마다 Tailer Thread(위의 phase 4) 실행
        for i, st in enumerate(self.states):
            t = threading.Thread(target=self._tail_aligned, args=(st, fifo_paths[i], ...))
            t.start()

        # 3. 모든 스트림이 타겟 PTS를 찾을 때까지 메인 스레드 대기
        for evt in self._ready_events:
            evt.wait()

        # 4. 0으로 정렬된 FIFO 데이터를 입력받아 MPTS로 합치는 거대 FFmpeg 기동
        cmd = ["ffmpeg", "-y", "-copyts"]
        
        # 4개의 리맵핑 완료된 FIFO 파이프를 입력으로 연결
        for fifo in self._fifos:
            cmd += ["-f", PASSTHROUGH_FORMAT, "-i", fifo]

        # 코덱 복사 및 프로그램 맵핑 설정
        cmd += ["-c:v", "copy", "-c:a", "copy"]
        cmd += ["-program", "program_num=1:st=0:st=1:st=2:st=3"]
        cmd += ["-f", PASSTHROUGH_FORMAT, srt_out_url]

        self._forwarder = FFmpegProc("fwd-mpts", cmd)
        self._forwarder.start() # 최종 SRT 송출 시작
``` 

### Remap PCR,PTS,DTS
```python
PTS_WRAP = 1 << 33              # 90kHz PTS wrap-around (33-bit, 약 85899초)

def remap_pkt(pkt: bytes, base90: int) -> bytes:
    """TS 패킷의 PCR / PES PTS / PES DTS에서 base90을 빼서 반환."""
    # 1. TS 패킷 유효성 검사 (188바이트, Sync Byte 0x47)
    if len(pkt) != TS_SZ or pkt[0] != 0x47:
        return pkt

    # ----------------------------------------------------
    # 2. PCR (Program Clock Reference) 리맵핑
    # ----------------------------------------------------
    pcr = pcr_read(pkt)
    if pcr is not None:
        # PCR은 27MHz 기반이므로 300을 나누어 90kHz(base)로 맞춘 뒤 연산하고 다시 복원
        new_pcr = ((pcr // 300 - base90) % PTS_WRAP) * 300 + (pcr % 300)
        pkt = pcr_write(pkt, new_pcr)

    # ----------------------------------------------------
    # 3. PES Header의 PTS / DTS 리맵핑
    # ----------------------------------------------------
    if ts_pusi(pkt): # PUSI가 1일 때만 (즉, PES 헤더가 존재할 때만)
        afc = ts_afc(pkt)
        
        # Adaptation Field 존재 여부에 따라 페이로드(PES 헤더) 시작 오프셋 계산
        if afc == 1:
            off = 4
        elif afc == 3:
            off = 5 + pkt[4]
        else:
            return pkt

        pl = pkt[off:] # 여기서부터가 순수 PES 구역
        
        # PES Start Code (0x000001) 확인 및 헤더 파싱
        if len(pl) >= 14 and pl[:3] == b"\x00\x00\x01":
            pts, dts = pes_get_ts(pl)
            
            if pts is not None:
                # 핵심 연산: 현재 값에서 base_pts를 빼고 33비트 모듈러 연산
                new_pts = (pts - base90) % PTS_WRAP
                new_dts = (dts - base90) % PTS_WRAP if dts is not None else None
                
                # 조작된 값으로 PES 헤더 바이트를 다시 생성 (Bitwise 인코딩)
                new_pl = pes_set_ts(pl, new_pts, new_dts)
                
                # 원래 TS 패킷의 앞부분 + 조작된 PES 헤더 + 뒷부분을 이어붙임
                pkt = pkt[:off] + new_pl + pkt[off + len(new_pl):]
                
    return pkt
```
- MPEGTS에서는 90kHz를 PTS/DTS의 단위로 사용하는 것이 규격에서 갖게하는 표준
    - 모든 비디오/오디오 프레임 레이트를 소수점 없이 딱 떨어지는 정수로 나누기 위한 최소공배수이기 때문
        - 24fps : 3750 tick
        - 25fps : 3600 tick
        - 30fps : 3000 tick
        - 60fps : 1500 tick
        - 어떤 fps든 pts가 90000 증가했다면 1초 증가
- PTS는 프레임을 화면에 뿌리는 시간이지만 PCR은 수신 단말기(TV)의 PCR은 하드웨어 마스터 클럭을 인코더와 똑같이 맞추기 위한 신호
    - 90kHz보다 훨씬 정밀한 27MHz를 사용
    - 따라서 90kHz의 1틱에서 PCR은 300틱을 올림
- `new_pcr = ((pcr // 300 - base90) % PTS_WRAP) * 300 + (pcr % 300)`
    - base_pts는 90kHz 단위이므로 pcr을 300으로 나눈 몫 대상으로 빼주고 ts에서의 pts범위인 PTS_WRAP으로 모듈러 연산을 진행
    - 그리고 `//`연산으로 잘려나간 300미만의 값은 다시 더해줘서 보정

<br>
<hr>
<br>

### 송수신 sync 원리
- PCR은 송출에서 보내는 신호고 수신기 내부에는 STC(System Time Clock)이라고 하는 내부 타이머가 존재함
- 만약에 STC를 PCR과 동기화된 상태라면 PTS대로 재생했을 때 싱크가 완벽히 맞음을 보장함
- PCR 12:00:00을 수신했는데 STC가 11:59:58이었다면 2초 딜레이시켜서 STC를 12:00:00으로 맞춤

### 오디오 비디오 타임라인 
 ![img.png](https://imucoding.github.io/assets/images/multi-sync3.png)

- 오디오 프레임 1개당 비디오 프레임 1개씩 매핑되는것이 아니라 각각으로 동작
- 수신기 STC는 1개를 공통으로 쓰지만 오디오 비디오가 각각의 PTS tick을 가지고 정해진 시점에 재생됨

ex)
```
[ STC 시간에 따른 디코더 출력 시나리오 ]
STC = 0 : 비디오 1번 프레임 출력, 오디오 1번 프레임 출력 (동시 시작)
STC = 1500 : 비디오 2번 프레임 출력 (오디오는 아직 대기 중)
STC = 1920 : 오디오 2번 프레임 출력 (비디오는 그대로 유지)
STC = 3000 : 비디오 3번 프레임 출력
STC = 3840 : 오디오 3번 프레임 출력
STC = 4500 : 비디오 4번 프레임 출력
```
{% endraw %}