# Linux 환경 점검 & micromamba 환경 구축

> sscc.uos 같은 클러스터에 처음 들어갔을 때 "여기 뭐냐"부터 확인하고, 분석용 Python 환경을 micromamba로 깔끔하게 세팅하는 절차.

---

## (1) 내 OS / Shell 정보 확인하기

새 서버에 SSH로 붙으면 **OS 종류·버전·커널·아키텍처·셸**을 먼저 파악하는 게 디버깅 시간을 가장 많이 줄여준다. (어떤 패키지 매니저 쓰는지, glibc 호환되는지, 어떤 micromamba 바이너리 받아야 하는지가 다 여기서 갈린다.)

### A. 배포판 종류와 버전 (가장 자주 쓰는 것)

```bash
cat /etc/os-release        # ← 거의 모든 모던 리눅스에서 표준. 1순위.
```

출력 예시 (AlmaLinux 9):
```
NAME="AlmaLinux"
VERSION="9.4 (Seafoam Ocelot)"
ID="almalinux"
ID_LIKE="rhel centos fedora"
PRETTY_NAME="AlmaLinux 9.4 (Seafoam Ocelot)"
```

핵심 필드:
- `ID` / `ID_LIKE` : 배포판 계열 (RHEL계? Debian계? Arch계?)
- `VERSION_ID` : 버전 숫자 (예: `9.4`)
- `PRETTY_NAME` : 사람이 읽기 좋은 한 줄

> sscc.uos 같은 CMS Tier 환경은 보통 **RHEL 계열**(CentOS 7 / AlmaLinux 8·9, Rocky Linux)이다. CMSSW가 그 위에서 빌드되니까. `ID_LIKE`에 `rhel`이 들어있으면 `yum`/`dnf` 패키지 매니저, `debian`이면 `apt`.

**대체 방법들 (`os-release`가 없거나 더 자세히):**

```bash
lsb_release -a             # lsb-release 패키지 설치돼 있을 때 (없으면 not found)
hostnamectl                # systemd 기반 시스템 (OS + kernel + arch 한 번에)
cat /etc/*release          # 거의 모든 시스템에 뭔가는 있음 (와일드카드로 긁기)
cat /etc/redhat-release    # RHEL 계열 전용
cat /etc/debian_version    # Debian/Ubuntu 계열 전용
```

### B. 커널과 아키텍처

배포판 ≠ 커널. 둘은 별개다.

```bash
uname -a       # 전부 한 번에 — 커널 이름·호스트명·커널 버전·아키텍처
uname -r       # 커널 버전만 (예: 5.14.0-427.el9.x86_64)
uname -m       # CPU 아키텍처 (x86_64 / aarch64 / ppc64le)
uname -s       # OS 이름 (Linux)
cat /proc/version     # 커널 + 빌드된 gcc 버전까지
```

> `uname -m`이 `x86_64`인지 `aarch64`인지로 micromamba/PyTorch 같은 바이너리 다운로드 URL이 달라진다. 보통 클러스터는 `x86_64`.

### C. CPU·메모리·디스크 (자원 파악)

```bash
lscpu          # CPU 모델·코어 수·캐시·소켓 구조까지
nproc          # 사용 가능한 논리 코어 수 (병렬화·HTCondor request_cpus 잡을 때)
free -h        # RAM 사용량 (-h: 사람 친화 단위)
df -h          # 마운트된 파일시스템 디스크 사용량
df -h ~        # 내 홈 디렉토리 quota 확인
du -sh *       # 현재 폴더의 각 항목 크기
du -sh ~/.cache ~/.conda ~/micromamba   # 홈 quota 잡아먹는 범인 찾기
```

> 클러스터 홈은 quota가 작은 경우가 많다. 큰 환경/캐시는 **scratch 영역**으로 옮기는 게 정석 — 이건 (2)번 micromamba에서 다시 다룬다.

### D. 셸 정보

배포판이 같아도 사람마다 쓰는 셸이 다르다. 스크립트가 안 돌 때 "내가 지금 무슨 셸인가"부터 확인.

```bash
echo $SHELL             # 로그인 셸 설정값 (/etc/passwd 기준) — 지금 실제 셸과 다를 수 있음
echo $0                 # 지금 실행 중인 셸 (예: -bash, /bin/zsh)
ps -p $$                # 현재 셸 프로세스 정보 (PID와 명령어)
echo $BASH_VERSION      # bash면 값이 뜸, zsh면 비어있음
echo $ZSH_VERSION       # 반대
which bash zsh fish     # 시스템에 설치된 셸들 위치
cat /etc/shells         # 사용 가능한 셸 목록
```

> 차이: `$SHELL`은 "기본으로 설정된" 셸, `$0`/`ps -p $$`는 "지금 이 순간 실제로 돌고 있는" 셸. 스크립트 안에서 셸을 일시 변경했을 때 이 둘이 갈린다.

### E. 사용자·시스템 상태 (보너스, 자주 씀)

```bash
whoami                  # 내 username
id                      # uid, gid, 소속 그룹 전부
groups                  # 내 그룹들만
echo $HOME              # 홈 경로
echo $USER              # = whoami
hostname                # 머신 이름
uptime                  # 가동 시간 + load average (서버 혼잡도 빠른 확인)
who                     # 지금 접속한 사용자들
w                       # who + 그들이 뭐 하고 있는지
```

### F. 한 줄 종합 점검 (복붙용 스니펫)

새 서버 들어가자마자 칠 만한 한 줄:

```bash
echo "=== OS ===" && cat /etc/os-release | grep -E 'PRETTY_NAME|ID=' && \
echo "=== Kernel/Arch ===" && uname -srm && \
echo "=== CPU ===" && lscpu | grep -E 'Model name|^CPU\(s\)|Architecture' && \
echo "=== Mem ===" && free -h | head -2 && \
echo "=== Shell ===" && echo "SHELL=$SHELL  current=$(ps -p $$ -o comm=)"
```

> `.bashrc`에 함수로 박아두면 편하다:
> ```bash
> sysinfo() { 위 명령 ; }
> ```

---

## (2) micromamba로 환경 구축

### A. 왜 micromamba?

| 도구 | 정체 | 특징 |
|---|---|---|
| **conda** | Anaconda/Miniconda의 기본 패키지 매니저 | 표준이지만 **느리다**(Python solver) |
| **mamba** | conda의 C++ 재구현 solver | 빠르지만 conda base 환경에 의존 |
| **micromamba** | 단일 정적 바이너리 (~10 MB) | **base 환경 불필요**, 빠름, 클러스터 친화적 |

micromamba가 클러스터/HEP 환경에서 점점 표준이 되는 이유:
- **단일 바이너리** — Python 미리 깔 필요 없음, 그냥 실행 파일 하나 풀면 끝.
- **빠른 의존성 해결** — mamba의 C++ libsolv 엔진 그대로.
- **conda와 호환** — `environment.yml`, conda-forge 채널 그대로 사용.
- **홈 quota 압박 적음** — base env가 필요 없어서 디스크 절약.

### B. 설치

**방법 1: 공식 설치 스크립트 (가장 간단)**

```bash
"${SHELL}" <(curl -L micro.mamba.pm/install.sh)
```

대화형으로 설치 위치(`MAMBA_ROOT_PREFIX`)와 셸 init 여부를 묻는다. 권장 답:
- Install prefix: 기본 `~/micromamba` (홈이 작으면 `/scratch/$USER/micromamba` 같은 곳으로 변경)
- shell init: **Yes**

설치 후:
```bash
source ~/.bashrc        # 또는 새 터미널 열기
micromamba --version    # 잘 됐는지 확인
```

**방법 2: 수동 바이너리 다운로드 (스크립트 못 쓰는 환경, 정확한 경로 제어용)**

```bash
# 원하는 폴더에서
curl -Ls https://micro.mamba.pm/api/micromamba/linux-64/latest | tar -xvj bin/micromamba
# 그러면 ./bin/micromamba 가 생김
mkdir -p ~/.local/bin
mv bin/micromamba ~/.local/bin/

# 셸 통합 (한 번만)
~/.local/bin/micromamba shell init -s bash -p ~/micromamba
source ~/.bashrc
```

> `linux-64`는 x86_64용. `aarch64` 머신이면 URL의 `linux-64`를 `linux-aarch64`로 바꿔라.

**설치 위치 팁 — `MAMBA_ROOT_PREFIX`:**
환경들이 저장되는 폴더. 기본은 `~/micromamba`인데, 클러스터 홈이 작으면(예: 50 GB quota) PyTorch 한 번 깔고 끝난다. `.bashrc`에 미리 박아둬라:

```bash
export MAMBA_ROOT_PREFIX=/scratch/$USER/micromamba   # 큰 storage 영역으로
```

### C. 가장 기본적인 사용 흐름

```bash
# 1. 새 환경 생성 (Python 3.11 + conda-forge 채널)
micromamba create -n hep -c conda-forge python=3.11

# 2. 활성화
micromamba activate hep
# 프롬프트가 (hep) 로 바뀜

# 3. 패키지 설치
micromamba install -c conda-forge numpy scipy matplotlib

# 4. 무엇이 들었나
micromamba list

# 5. 끝나면 비활성화
micromamba deactivate

# 6. 환경 목록 보기
micromamba env list

# 7. 환경 통째로 삭제
micromamba env remove -n hep
```

> conda-forge 채널을 기본으로 쓰는 게 정석. ROOT, uproot, mplhep, awkward, dask 같은 HEP 도구가 다 거기 있고, 패키지가 가장 최신/완전하다. 매번 `-c conda-forge` 치기 귀찮으면 채널을 기본값으로 박아둘 수 있다:
> ```bash
> micromamba config append channels conda-forge
> micromamba config set channel_priority strict
> ```

### D. `environment.yml`로 재현성 있게 (실무에선 이게 표준)

명령으로 일일이 깔면 환경을 남이 못 재현한다. YAML로 명세하라.

**`environment.yml` 예시 (HEP 분석용):**

```yaml
name: hep-analysis
channels:
  - conda-forge
dependencies:
  - python=3.11
  - numpy
  - scipy
  - pandas
  - matplotlib
  - mplhep
  - uproot
  - awkward
  - hist
  - jupyterlab
  - tqdm
  - pyyaml
  - pip
  - pip:
      - aim       # 네 CICY ML 트래킹
```

생성·갱신:
```bash
micromamba env create -f environment.yml          # 처음 생성
micromamba env update -f environment.yml -n hep-analysis   # 파일 수정 후 반영
```

현재 환경 그대로 yaml로 뽑기:
```bash
micromamba env export -n hep-analysis > env.yml
# 빌드 해시 빼고 깨끗하게:
micromamba env export -n hep-analysis --from-history > env-clean.yml
```

> `--from-history`는 네가 명시적으로 install한 패키지만 적어줘서 가독성·이식성이 좋다. git에 올릴 땐 이쪽.

### E. HEP/클러스터에서 자주 부딪히는 함정

**(1) CMSSW와 섞지 말 것.**
`cmsenv`는 자체 Python·gcc·환경 변수 풀세트를 깐다. micromamba 환경이 활성화된 상태에서 `cmsenv`를 부르면 `PATH`·`LD_LIBRARY_PATH`가 꼬여서 둘 다 깨진다.
- 분석 코드(uproot/mplhep)는 micromamba 환경에서,
- Combine·CMSSW 컴파일·실행은 깨끗한 셸에서 `cmsenv`로,
- 두 세계를 한 스크립트에서 섞지 말고 **분리해서 호출**.

**(2) `.bashrc` 자동 activate 끄기.**
설치 스크립트가 `.bashrc`에 `micromamba activate base` 비슷한 줄을 추가할 수 있다. 매 로그인마다 환경이 자동으로 켜지면 CMSSW 작업할 때 불편하다. 자동 activate는 빼고, 필요할 때 수동으로:
```bash
micromamba activate hep
```

**(3) HTCondor 잡에서 환경 활성화.**
condor에 제출하는 셸 스크립트 안에서 환경을 켜려면 셸 통합이 먼저 필요하다:
```bash
#!/bin/bash
export MAMBA_ROOT_PREFIX=/scratch/$USER/micromamba
eval "$(/path/to/micromamba shell hook -s bash)"
micromamba activate hep
python my_analysis.py
```
`eval "$(... shell hook ...)"`이 핵심 — condor의 비대화형 셸에서도 `micromamba activate`가 동작하게 해준다.

**(4) 캐시·환경 청소.**
사용 안 하는 패키지 캐시는 금방 GB 단위로 불어난다. 가끔 청소:
```bash
micromamba clean --all      # tarball 캐시·미사용 패키지 정리
du -sh $MAMBA_ROOT_PREFIX   # 환경 디렉토리 총 용량
```

### F. 자주 쓰는 alias 모음 (`.bashrc`에 추가용)

```bash
alias mm='micromamba'
alias mma='micromamba activate'
alias mmd='micromamba deactivate'
alias mml='micromamba env list'
alias mmi='micromamba install -c conda-forge'
```

---

## 한 페이지 치트시트

**OS·셸 빠른 확인**
```bash
cat /etc/os-release    # 배포판
uname -srm             # 커널 + 아키텍처
echo $0                # 현재 셸
nproc; free -h; df -h  # 자원
```

**micromamba 일상 명령**
```bash
micromamba create -n NAME -c conda-forge python=3.11   # 생성
micromamba activate NAME                                # 활성화
micromamba install -c conda-forge PKG                   # 설치
micromamba list                                         # 목록
micromamba env list                                     # 환경들
micromamba env create -f env.yml                        # yaml로 생성
micromamba env export -n NAME --from-history > env.yml  # yaml로 뽑기
micromamba env remove -n NAME                           # 삭제
```
