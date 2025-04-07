# AI 서버에서 그래픽카드 문제 진단 및 해결 가이드

## 개요
서버에서 `nvidia-smi`, `lspci`, `dmesg` 등을 통해 확인했을 때 그래픽카드(NVIDIA) 관련 문제가 발생했을 때 참고할 수 있는 진단 및 해결 방법을 정리했습니다.

---

## 1. 물리적인 GPU 인식 여부 확인

1. **lspci 결과 확인**  
   - 다음 명령을 통해 GPU가 표시되는지 확인합니다:
     ```bash
     lspci | grep -i vga
     lspci | grep -i nvidia
     ```
   - 정상적으로 “NVIDIA Corporation Device …”가 표시되어야 합니다.
   - 만약 전혀 보이지 않는다면, **하드웨어나 BIOS/UEFI 레벨**에서 GPU를 잡지 못하는 문제일 수 있습니다.

2. **물리적 연결(전원/슬롯) 점검**  
   - 전원 케이블 또는 라이저 카드(서버형) 연결 상태를 확인합니다.
   - 슬롯을 바꿔 꽂아 보는 것도 방법입니다.

---

## 2. NVIDIA 드라이버 설치/동작 상태 점검

1. **드라이버 설치 여부**  
   - Ubuntu 22.04 LTS 환경에서 권장 드라이버를 자동 설치하는 방법:
     ```bash
     sudo apt update
     sudo apt upgrade -y
     sudo ubuntu-drivers autoinstall
     sudo reboot
     ```
   - 혹은 특정 버전을 직접 지정할 수도 있습니다:
     ```bash
     sudo apt install nvidia-driver-530
     sudo reboot
     ```

2. **Secure Boot 상태**  
   - UEFI에서 **Secure Boot**가 활성화되어 있으면, NVIDIA 서명 모듈 로딩이 실패할 수 있습니다.  
   - 이 경우 드라이버가 설치되었어도 `nvidia-smi`에서 오류가 날 수 있습니다.  
   - 가급적 **Secure Boot를 비활성화**한 뒤 드라이버를 다시 설치해보는 것을 권장합니다.

3. **dmesg 로그 분석**  
   - 아래 명령으로 NVIDIA 관련 에러 메시지를 확인합니다:
     ```bash
     sudo dmesg | grep -i nvidia
     sudo dmesg | grep -i "3d:00.0"  # GPU 버스가 3d:00.0 예시일 때
     ```
   - 주로 다음과 같은 에러가 있는지 확인:
     - `NVRM: RmInitAdapter failed!`
     - `NVRM: GPU is lost`
     - `Device is in D3cold…` 등  
   - 에러 메시지가 뜬다면, **커널 모듈 로드** 혹은 **하드웨어/전원** 문제를 의심해야 합니다.

4. **드라이버 모듈 로딩 확인**  
   - 다음 명령으로 실제 `nvidia` 모듈이 로드되었는지 확인합니다:
     ```bash
     lsmod | grep nvidia
     ```
   - `nvidia-smi`가 정상 동작하지 않으면 모듈 충돌, 설치 불량, DKMS 문제 등이 있을 수 있습니다.

---

## 3. CUDA 환경(선택 사항) 점검

- **CUDA**가 필요한 경우, 버전 호환도 중요한 포인트입니다.
- CUDA와 드라이버 버전이 맞지 않으면 문제가 생길 수 있습니다.
- 예: CUDA 11.8 설치 시, `nvidia-smi`에 표시되는 CUDA 버전도 11.8이어야 합니다.

---

## 4. 추가 진단 및 해결 팁

1. **기존 드라이버 제거 후 재설치**  
   - 타사(.run 파일) 혹은 다른 PPA로 설치된 드라이버가 있으면 충돌이 날 수 있습니다.
   - `dpkg -l | grep nvidia` 등으로 확인 후 불필요한 패키지를 제거하고, 다시 공식 리포지토리 드라이버를 설치해보세요.

2. **커널 업데이트 후 DKMS 확인**  
   - 새 커널 버전으로 업데이트되었는데, 드라이버 모듈이 재빌드되지 않아 로드가 실패할 수 있습니다.
   - 다음 명령으로 상태를 확인합니다:
     ```bash
     sudo dkms status
     ```

3. **BIOS/UEFI 설정**  
   - BIOS에서 PCI-E 슬롯이 비활성화되어 있거나, 다른 자원과 충돌이 있을 수 있습니다.
   - BIOS 버전 업데이트도 고려해볼 수 있습니다.

4. **하드웨어 고장 점검**  
   - GPU 불량 혹은 메인보드 PCI-E 슬롯 자체의 문제일 수도 있습니다.
   - GPU를 다른 서버나 슬롯에 옮겨서 정상 동작 여부를 확인해볼 수 있습니다.

5. **정확한 로그 공유**  
   - 문제 해결이 어려우면 `/var/log/kern.log`, `/var/log/syslog`, `dmesg` 전체 로그 내 **GPU 관련 오류 메시지**를 세부적으로 분석해야 합니다.

---

## 결론

- **lspci**에서 GPU가 보이는지, **dmesg**에서 에러 메시지가 있는지, 그리고 **Secure Boot** 비활성화 여부를 먼저 확인하는 것이 핵심입니다.
- 드라이버 설치 후에도 문제가 지속되면, DKMS 모듈 로딩 상태나 하드웨어 연결 상태를 다시 점검해야 합니다.
- 위 단계를 모두 확인하고도 문제가 해결되지 않는다면, 구체적인 로그(커널 로그, dmesg 등)를 더 많이 살펴봐야 합니다.

