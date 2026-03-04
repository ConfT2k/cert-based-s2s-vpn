# AWS Site-to-Site VPN 인증서 기반 구성 가이드
## Transit Gateway + Cisco CSR (DHCP/동적 IP) 환경

---

## 문서 정보

| 항목 | 내용 |
|------|------|
| 작성일 | 2026-03-02 |
| 대상 환경 | AWS Transit Gateway ↔ Cisco CSR 1000v (IOS-XE) |
| 인증 방식 | 인증서 기반 (AWS Private CA) |
| CGW IP | DHCP (동적 IP) |
| 라우팅 | BGP (Dynamic) |
| IKE 버전 | IKEv2 |

---

## 아키텍처 개요

```
┌───────────────────────────────────────────────────────────┐
│                        AWS Cloud                          │
│                                                           │
│  ┌──────┐        ┌─────────┐        ┌──────────────┐      │
│  │ VPC  │────────│   TGW   │────────│VPN Connection│      │
│  └──────┘        └─────────┘        └──────┬───────┘      │
│                                            │              │
└────────────────────────────────────────────┼──────────────┘
                                             │
                                      IPsec/IKEv2
                                      (인증서 인증)
                                             │
                                ┌────────────┴────────────┐
                                │    Cisco CSR 1000v      │
                                │    (DHCP 동적 IP)        │
                                │    Customer Gateway     │
                                │    172.20.85.0/24       │
                                └─────────────────────────┘
```

### 핵심 포인트
- CGW가 DHCP를 사용하므로 IP가 변경될 수 있음 → PSK 대신 **인증서 인증** 필수
- AWS Private CA에서 발급한 인증서만 Site-to-Site VPN에 사용 가능 (외부 자체 서명 인증서 불가)
- Customer Gateway 생성 시 IP 주소를 비워두면 동적 IP 지원
- VPN 연결은 항상 CGW(Cisco CSR) 측에서 먼저 시작(initiate)해야 함

---

## 사전 요구사항

### AWS 측
- AWS 계정 및 적절한 IAM 권한 (VPC, ACM, Private CA, TGW 관련)
- VPC 및 서브넷 구성 완료
- AWS CLI 또는 Console 접근 가능

### On-Premises 측
- Cisco CSR 1000v (IOS-XE 16.9 이상 권장)
- 인터넷 연결 (DHCP로 공인 IP 할당)
- SSH 또는 Console 접근 가능

### 비용 참고
- AWS Private CA: 월 $400/CA (General-Purpose 모드)
  - **Short-Lived Certificate 모드**: 월 $50/CA (인증서 유효기간 7일 이하)
- VPN Connection: 시간당 $0.05 (리전별 상이)
- 데이터 전송 요금 별도

> ⚠️ 정확한 비용은 [AWS Pricing Calculator](https://calculator.aws/)에서 확인하세요.

### 비용 최적화: Short-Lived Certificate 모드 활용

프로덕션 환경이 아니거나 비용 절감이 필요한 경우, **Short-Lived Certificate 모드**를 적극 활용하세요.

| 모드 | 월 비용 | 인증서 유효기간 | 적합한 용도 |
|------|---------|----------------|------------|
| General-Purpose | $400/CA | 최대 10년 | 프로덕션, 장기 운영 |
| Short-Lived | $50/CA | 최대 7일 | 개발/테스트, PoC, 비용 민감 환경 |

**Short-Lived 모드 사용 시 고려사항:**
- 인증서를 7일 이내로 갱신해야 함
- 자동 갱신 스크립트 또는 Lambda 함수 구성 권장
- VPN 연결 중단 없이 인증서 갱신 가능 (동일 CA 체인 사용 시)

```bash
# Short-Lived 모드로 Private CA 생성 (AWS CLI)
aws acm-pca create-certificate-authority \
  --certificate-authority-configuration '{
    "KeyAlgorithm": "RSA_2048",
    "SigningAlgorithm": "SHA256WITHRSA",
    "Subject": {
      "CommonName": "MyOrg Short-Lived CA",
      "Organization": "MyOrganization",
      "Country": "KR"
    }
  }' \
  --certificate-authority-type "SUBORDINATE" \
  --usage-mode SHORT_LIVED_CERTIFICATE \
  --tags Key=Purpose,Value=VPN-ShortLived
```

### 월 예상 비용 요약표

| 구성 요소 | General-Purpose CA | Short-Lived CA | 비고 |
|----------|-------------------|----------------|------|
| Root CA | $400 | $50 | 1개 필요 |
| Subordinate CA | $400 | $50 | 1개 필요 |
| VPN Connection | ~$36 | ~$36 | $0.05/시간 × 720시간 |
| 데이터 전송 (100GB) | ~$9 | ~$9 | $0.09/GB (아웃바운드) |
| **월 총계 (예상)** | **~$845** | **~$145** | 데이터 전송량에 따라 변동 |

> 📌 **개발/테스트 환경에서는 Short-Lived CA로 월 $700 이상 절감 가능**

---

# Part 1: AWS Console 구성

## 1단계: AWS Private CA 생성

AWS Private CA는 Root CA와 Subordinate CA 2단계로 구성합니다.
Site-to-Site VPN 인증서는 반드시 **Subordinate CA**에서 발급해야 합니다.

### 1.1 Root CA 생성

1. AWS Console → **AWS Private Certificate Authority** 서비스로 이동
2. **Create a private CA** 클릭
3. 다음과 같이 설정 (Console 순서대로):

**Mode options**
| 옵션 | 선택 |
|------|------|
| General-purpose | ✅ 선택 |
| Short-lived certificate | (비용 절감 시 선택 - 월 $50) |

**CA type options**
| 옵션 | 선택 |
|------|------|
| Root | ✅ 선택 |
| Subordinate | |

**Subject distinguished name options**
| 항목 | 값 | 필수 |
|------|---|:----:|
| Organization (O) | `MyOrganization` | |
| Organization unit (OU) | (비워둠) | |
| Country name (C) | `KR` | |
| State or province name | (비워둠) | |
| Locality name | (비워둠) | |
| Common name (CN) | `MyOrg Root CA` | ✅ |

**Key algorithm options**
| 옵션 | 선택 |
|------|------|
| RSA 2048 | ✅ 권장 |
| RSA 4096 | (더 높은 보안 필요 시) |

> 📌 Post-quantum 알고리즘(ML-DSA)은 현재 Site-to-Site VPN에서 지원하지 않으므로 Classical 알고리즘을 선택하세요.

**Certificate revocation options**
- CRL distribution: 기본값 유지 (선택사항)

**Add tags** (선택사항)
- Key: `Purpose`, Value: `VPN-RootCA`

4. **Create CA** 클릭
5. 생성 후 상태가 `Pending certificate`로 표시됨

### 1.2 Root CA 인증서 설치

Root CA는 자체 서명 인증서를 설치해야 활성화됩니다.

1. 생성된 Root CA 선택 → **Actions** → **Install CA certificate**
2. 유효 기간 설정:
   - Validity: **10 years** (권장)
   - Signature algorithm: **SHA256WITHRSA**
3. **Confirm and install** 클릭
4. 상태가 `Active`로 변경되는지 확인

### 1.3 Subordinate CA 생성

1. **Create a private CA** 클릭
2. 다음과 같이 설정 (Console 순서대로):

**Mode options**
| 옵션 | 선택 |
|------|------|
| General-purpose | ✅ 선택 (Root CA와 동일) |
| Short-lived certificate | (Root CA와 동일하게 선택) |

**CA type options**
| 옵션 | 선택 |
|------|------|
| Root | |
| Subordinate | ✅ 선택 |

**Subject distinguished name options**
| 항목 | 값 | 필수 |
|------|---|:----:|
| Organization (O) | `MyOrganization` | |
| Organization unit (OU) | (비워둠) | |
| Country name (C) | `KR` | |
| State or province name | (비워둠) | |
| Locality name | (비워둠) | |
| Common name (CN) | `MyOrg VPN Subordinate CA` | ✅ |

**Key algorithm options**
| 옵션 | 선택 |
|------|------|
| RSA 2048 | ✅ 권장 |

**Add tags** (선택사항)
- Key: `Purpose`, Value: `VPN-SubordinateCA`

3. **Confirm and create** 클릭

### 1.4 Subordinate CA 인증서 설치

1. 생성된 Subordinate CA 선택 → **Actions** → **Install CA certificate**
2. Parent CA로 **앞서 생성한 Root CA** 선택
3. 유효 기간: **5 years** (권장)
4. Signature algorithm: **SHA256WITHRSA**
5. **Confirm and install** 클릭
6. 상태가 `Active`로 변경되는지 확인

> ✅ 검증: AWS Private CA 콘솔에서 Root CA와 Subordinate CA 모두 `Active` 상태인지 확인

---

## 2단계: VPN용 프라이빗 인증서 발급

Customer Gateway 장비(Cisco CSR)에 설치할 인증서를 ACM에서 발급합니다.

### 2.1 ACM에서 프라이빗 인증서 요청

1. AWS Console → **Certificate Manager (ACM)** 서비스로 이동
2. **Request a certificate** 클릭
3. **Request a private certificate** 선택 → **Next**
4. 다음과 같이 설정:

| 설정 항목 | 값 |
|-----------|---|
| Certificate authority | 앞서 생성한 Subordinate CA 선택 |
| Domain name | `vpn.myorg.local` (또는 원하는 FQDN) |
| Key algorithm | RSA 2048 |

> 📌 **Domain name은 기술적으로 필수가 아닙니다.** VPN 인증서 인증은 인증서 체인 검증을 기반으로 하며, domain name 검증은 수행하지 않습니다. 다만 관리 편의를 위해 식별 가능한 이름을 입력하는 것이 권장됩니다.

5. **Request** 클릭
6. 인증서 상태가 `Issued`로 변경되는지 확인

> 📌 이 인증서의 **ARN**을 메모해 두세요. Customer Gateway 생성 시 필요합니다.
> 예: `arn:aws:acm:REGION:ACCOUNT_ID:certificate/CERTIFICATE_ID`

---

## 3단계: Service-Linked Role 확인

인증서 기반 VPN을 사용하려면 Service-Linked Role이 필요합니다.
이 역할은 AWS가 VPN 터널의 AWS 측 인증서를 자동 생성/관리하는 데 사용합니다.

### 확인 방법

1. AWS Console → **IAM** → **Roles**
2. `AWSServiceRoleForVPCS2SVPN` 검색
3. 존재하지 않으면 VPN Connection 생성 시 자동으로 생성됨

또는 AWS CLI로 확인:
```bash
aws iam get-role --role-name AWSServiceRoleForVPCS2SVPN
```

존재하지 않으면 수동 생성:
```bash
aws iam create-service-linked-role --aws-service-name s2svpn.amazonaws.com
```

---

## 4단계: Transit Gateway 생성

이미 TGW가 있다면 이 단계를 건너뛰세요.

1. AWS Console → **VPC** → **Transit Gateways**
2. **Create transit gateway** 클릭
3. 설정:

| 설정 항목 | 값 |
|-----------|---|
| Name tag | `my-tgw` |
| Amazon side ASN | `64512` (기본값 또는 원하는 ASN) |
| DNS support | Enable |
| VPN ECMP support | Enable |
| Default route table association | Enable |
| Default route table propagation | Enable |
| Auto accept shared attachments | Disable |

4. **Create transit gateway** 클릭
5. 상태가 `Available`이 될 때까지 대기 (약 5분)

### 4.1 VPC를 Transit Gateway에 연결

1. **Transit Gateway Attachments** → **Create transit gateway attachment**
2. 설정:

| 설정 항목 | 값 |
|-----------|---|
| Transit gateway ID | 위에서 생성한 TGW 선택 |
| Attachment type | VPC |
| VPC ID | 연결할 VPC 선택 |
| Subnet IDs | 각 AZ에서 서브넷 선택 |

3. **Create transit gateway attachment** 클릭

> ✅ 검증: TGW Attachment 상태가 `Available`인지 확인

---

## 5단계: Customer Gateway 생성

Customer Gateway는 온프레미스 라우터(Cisco CSR)를 AWS에 등록하는 리소스입니다.
인증서 기반이므로 IP 주소를 비워두고 인증서 ARN을 지정합니다.

### 인증서 ARN 제약사항 (중요)

Customer Gateway에서 사용할 인증서는 다음 제약사항을 **반드시** 준수해야 합니다:

| 제약사항 | 설명 |
|---------|------|
| **리전 일치 필수** | 인증서는 Customer Gateway를 생성할 리전과 **동일한 리전**의 ACM에 있어야 함 |
| **AWS Private CA 전용** | 외부 CA 또는 자체 서명 인증서는 사용 불가. 반드시 AWS Private CA에서 발급한 인증서만 사용 가능 |
| **Subordinate CA 발급** | Root CA가 아닌 **Subordinate CA**에서 발급한 인증서여야 함 |
| **RSA 키만 지원** | ECDSA 키 알고리즘은 지원하지 않음. RSA 2048 또는 RSA 4096 사용 |
| **프라이빗 인증서** | ACM의 퍼블릭 인증서(도메인 검증)가 아닌 프라이빗 인증서여야 함 |

> ⚠️ **리전 불일치 오류 예시**:
> ```
> Error: Certificate ARN arn:aws:acm:us-west-2:123456789012:certificate/xxx
> is not in the same region as the customer gateway (ap-northeast-2)
> ```
> 이 오류는 인증서가 `us-west-2`에 있고 CGW를 `ap-northeast-2`에서 생성하려 할 때 발생합니다.

> ⚠️ **Cross-Region 구성이 필요한 경우**:
> - 각 리전마다 별도의 Subordinate CA를 생성하거나
> - 동일한 Root CA 아래에서 각 리전에 Subordinate CA와 인증서를 발급해야 함
> - Private CA는 리전 리소스이므로 CA도 해당 리전에 있어야 함

### Customer Gateway 생성 절차

1. AWS Console → **VPC** → **Customer Gateways**
2. **Create customer gateway** 클릭
3. 설정:

| 설정 항목 | 값 |
|-----------|---|
| Name tag | `cisco-csr-cgw` |
| BGP ASN | `65000` (Cisco CSR에서 사용할 ASN) |
| IP address | **비워둠** (DHCP이므로) |
| Certificate ARN | 2단계에서 발급한 인증서 ARN 선택 |
| Device | `Cisco CSR 1000v` (선택사항) |

4. **Create customer gateway** 클릭

> ⚠️ IP address를 비워두면 AWS는 IP 주소를 확인하지 않으므로, CGW의 IP가 변경되어도 VPN 연결을 재구성할 필요가 없습니다.

> ⚠️ 인증서 기반 VPN에서 IP를 비워두면 VPN 연결은 항상 **CGW 측에서 먼저 시작(initiate)**해야 합니다. AWS 측에서는 연결을 시작할 수 없습니다.

---

## 6단계: VPN Connection 생성

1. AWS Console → **VPC** → **Site-to-Site VPN Connections**
2. **Create VPN connection** 클릭
3. 설정:

| 설정 항목 | 값 |
|-----------|---|
| Name tag | `s2s-vpn-cert` |
| Target gateway type | **Transit Gateway** |
| Transit gateway | 4단계에서 생성한 TGW 선택 |
| Customer gateway | **Existing** → `cisco-csr-cgw` 선택 |
| Routing options | **Dynamic (requires BGP)** |
| Tunnel inside IP version | IPv4 |
| Local IPv4 network CIDR | 비워둠 (0.0.0.0/0) |
| Remote IPv4 network CIDR | 비워둠 (0.0.0.0/0) |

4. **Tunnel Options** (선택사항 - 기본값 사용 가능):

| 터널 옵션 | 권장값 |
|-----------|--------|
| Tunnel 1 inside CIDR | 기본값 (AWS 자동 할당) |
| Tunnel 2 inside CIDR | 기본값 (AWS 자동 할당) |
| IKE version | IKEv2 |
| Phase 1 encryption | AES256 |
| Phase 1 integrity | SHA2-256 |
| Phase 1 DH group | 14 |
| Phase 1 lifetime | 28800 |
| Phase 2 encryption | AES256 |
| Phase 2 integrity | SHA2-256 |
| Phase 2 DH group | 14 |
| Phase 2 lifetime | 3600 |
| DPD timeout | 30 |
| DPD timeout action | Restart |
| Startup action | Add (AWS 측에서 시작하지 않음) |

5. **Create VPN connection** 클릭
6. 상태가 `Available`이 될 때까지 대기 (약 5-10분)

### Startup Action 옵션 설명

VPN 터널의 **Startup action**은 AWS 측에서 IKE 협상을 어떻게 처리할지 결정합니다.

| 옵션 | 동작 | 적합한 환경 |
|------|------|------------|
| **Start** | AWS가 즉시 IKE 협상을 시작하고, 터널이 끊어지면 자동으로 재연결 시도 | CGW IP가 고정되어 있고, AWS에서 능동적으로 연결을 유지해야 하는 경우 |
| **Add** | AWS는 IKE 협상을 시작하지 않고, CGW의 연결 요청을 기다림 | CGW IP가 동적(DHCP)이거나, CGW 측에서 연결을 제어해야 하는 경우 |

**Start vs Add 상세 비교:**

| 비교 항목 | Start | Add |
|----------|-------|-----|
| IKE Initiator | AWS 또는 CGW (양방향) | CGW만 (단방향) |
| 터널 끊김 시 | AWS가 자동 재연결 시도 | CGW가 재연결해야 함 |
| 동적 IP 지원 | 제한적 (IP 변경 시 연결 실패 가능) | 완전 지원 |
| 방화벽 요구사항 | CGW가 인바운드 IKE(UDP 500/4500) 허용 필요 | CGW는 아웃바운드만 허용하면 됨 |
| DPD(Dead Peer Detection) | AWS가 주기적으로 DPD 패킷 전송 | CGW의 요청이 있을 때만 응답 |

> ⚠️ **DHCP/동적 IP 환경에서는 반드시 `Add`를 선택하세요.**
> - `Start`를 선택하면 AWS가 알 수 없는 IP로 연결을 시도하여 실패합니다.
> - CGW의 IP가 변경되어도 CGW 측에서 새로운 연결을 시작하면 정상 작동합니다.

> 📌 **고정 IP 환경에서의 권장사항:**
> - 고가용성이 필요하면 `Start` 선택 (AWS가 터널을 능동적으로 유지)
> - 비용 최적화가 필요하면 `Add` 선택 (필요할 때만 연결)

---

## 7단계: VPN 구성 정보 확인

Cisco CSR 구성에 필요한 정보를 확인합니다.

1. **VPN Connections** → 생성한 VPN 선택
2. **Tunnel details** 탭에서 다음 정보를 메모:

### Tunnel 1 정보
| 항목 | 예시 값 |
|------|--------|
| Outside IP Address (AWS) | `52.xx.xx.xx` |
| Inside IP CIDR (AWS) | `169.254.xx.xx/30` |
| Inside IP CIDR (CGW) | `169.254.xx.yy/30` |
| BGP ASN (AWS) | `64512` |

### Tunnel 2 정보
| 항목 | 예시 값 |
|------|--------|
| Outside IP Address (AWS) | `54.xx.xx.xx` |
| Inside IP CIDR (AWS) | `169.254.xx.xx/30` |
| Inside IP CIDR (CGW) | `169.254.xx.yy/30` |
| BGP ASN (AWS) | `64512` |

> 📌 인증서 기반 VPN에서는 PSK(Pre-Shared Key)가 표시되지 않습니다. 이것이 정상입니다.

---

## 8단계: 인증서 내보내기 (Cisco CSR용)

Cisco CSR에 설치할 인증서를 ACM에서 내보냅니다.

### ACM 인증서 목록 이해하기

VPN Connection 생성 후 ACM에서 인증서 목록을 확인하면 여러 개의 인증서가 보입니다:

| 인증서 이름 | 생성 주체 | 용도 | 내보내기 |
|------------|----------|------|:-------:|
| `vpn.myorg.local` | 사용자 | CGW(Cisco CSR) 인증 | ✅ 필요 |
| `vpn-xxxxxxx.endpoint-0` | AWS 자동 | AWS VPN 터널 1 인증 | 불필요 |
| `vpn-xxxxxxx.endpoint-1` | AWS 자동 | AWS VPN 터널 2 인증 | 불필요 |

> 📌 `vpn-xxxxxxx.endpoint-0/1` 인증서는 AWS가 VPN Connection 생성 시 **자동으로 생성**한 AWS 측 터널 엔드포인트 인증서입니다. Service-Linked Role (`AWSServiceRoleForVPCS2SVPN`)이 관리하며, 사용자가 내보내거나 관리할 필요가 없습니다.

**내보낼 인증서: `vpn.myorg.local` (사용자가 생성한 CGW 인증서만)**

### 8.1 프라이빗 인증서 내보내기

1. AWS Console → **Certificate Manager (ACM)**
2. 2단계에서 발급한 인증서(`vpn.myorg.local`) 선택
3. **Export** 클릭
4. Passphrase 입력 (인증서 개인키 보호용, 나중에 Cisco CSR에서 사용)
5. 3개 파일을 다운로드 (두 가지 방법 중 선택):
   - **방법 1**: 우측 상단의 **Download all** 버튼 클릭 → 3개 파일 일괄 다운로드
   - **방법 2**: 각 항목의 **Download** 버튼을 개별 클릭

| 항목 | 다운로드 파일명 | 내용 | Cisco CSR에서의 용도 |
|------|----------------|------|-------------------|
| Certificate body | 직접 지정 (예: `certificate.txt`) | CGW 디바이스 인증서 1개 | Identity certificate |
| Certificate chain | `certificate_chain.txt` | Subordinate CA + Root CA (2개) | CA trustpoint |
| Certificate private key | `private_key.txt` | 개인키 (passphrase로 암호화됨) | RSA keypair import |

> 💡 **Tip**: **Download all**을 사용하면 3개 파일을 한 번에 다운로드할 수 있어 편리합니다.

**다운로드 확인:**
```bash
# certificate_chain.txt에 CA 인증서 2개가 포함되어 있는지 확인
grep -c "BEGIN CERTIFICATE" certificate_chain.txt
# 결과: 2 (Subordinate CA + Root CA)
```

### 8.2 Root CA 인증서 별도 다운로드

1. AWS Console → **AWS Private CA**
2. Root CA 선택 → **CA certificate** 탭
3. **Get CA certificate** → Root CA 인증서 본문 복사/저장

### 8.3 Subordinate CA 인증서 별도 다운로드

1. Subordinate CA 선택 → **CA certificate** 탭
2. **Get CA certificate** → Subordinate CA 인증서 본문 복사/저장

> ⚠️ Cisco CSR에는 **3개 인증서 모두** 설치해야 합니다:
> 1. Root CA 인증서
> 2. Subordinate CA 인증서
> 3. 디바이스(CGW) 인증서 + 개인키
>
> 하나라도 누락되면 VPN 인증이 실패합니다.

---

## 9단계: Transit Gateway 라우팅 확인

### 9.1 TGW Route Table 확인

1. **VPC** → **Transit Gateway Route Tables**
2. 기본 라우트 테이블 선택
3. **Associations** 탭: VPC attachment와 VPN attachment가 모두 연결되어 있는지 확인
4. **Propagations** 탭: VPN attachment에서 라우트 전파가 활성화되어 있는지 확인

### 9.2 VPC Route Table 업데이트

VPC의 서브넷 라우트 테이블에 온프레미스 네트워크로의 경로를 추가합니다.

1. **VPC** → **Route Tables** → VPC 서브넷의 라우트 테이블 선택
2. **Edit routes** → **Add route**

| Destination | Target |
|-------------|--------|
| `172.20.85.0/24` (온프레미스 CIDR) | Transit Gateway (`my-tgw`) |

3. **Save changes**

> ✅ 검증: TGW Route Table에서 VPN을 통해 전파된 BGP 경로가 보이는지 확인 (VPN 연결 후)

---

## 10단계: VPN 로그 활성화 (권장)

VPN 터널 연결 문제를 진단하기 위해 CloudWatch Logs로 VPN 로그를 전송하는 것을 권장합니다.

### 10.1 CloudWatch Log Group 생성

1. AWS Console → **CloudWatch** → **Logs** → **Log groups**
2. **Create log group** 클릭
3. Log group name: `/aws/vpn/s2s-vpn-cert`
4. Retention setting: 필요에 따라 설정 (예: 30일)
5. **Create** 클릭

### 10.2 VPN 터널 로깅 활성화

1. **VPC** → **Site-to-Site VPN Connections** → VPN 선택
2. **Actions** → **Modify VPN tunnel options**
3. 각 터널에 대해:

| 설정 항목 | 값 |
|-----------|---|
| Enable tunnel activity log | Yes |
| CloudWatch log group | `/aws/vpn/s2s-vpn-cert` |
| Output format | json (권장) |

4. **Save changes** 클릭

### 10.3 로그 확인

VPN 터널 로그에서 확인할 수 있는 정보:
- IKE 협상 성공/실패
- Phase 1/Phase 2 상태
- 인증서 검증 결과
- DPD(Dead Peer Detection) 이벤트

```bash
# AWS CLI로 최근 로그 확인
aws logs filter-log-events \
  --log-group-name "/aws/vpn/s2s-vpn-cert" \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --limit 50
```

> 📌 VPN 로그는 트러블슈팅에 매우 유용합니다. 프로덕션 환경에서는 반드시 활성화하세요.

---

## AWS 측 구성 완료 체크리스트

- [ ] AWS Private CA: Root CA 생성 및 Active
- [ ] AWS Private CA: Subordinate CA 생성 및 Active
- [ ] ACM: 프라이빗 인증서 발급 완료 (Issued)
- [ ] Service-Linked Role 확인
- [ ] Transit Gateway 생성 및 Available
- [ ] TGW-VPC Attachment 생성 및 Available
- [ ] Customer Gateway 생성 (IP 비워둠, 인증서 ARN 지정)
- [ ] VPN Connection 생성 및 Available
- [ ] 인증서 3종 내보내기 완료 (Root CA, Sub CA, Device cert+key)
- [ ] TGW Route Table 전파 설정
- [ ] VPC Route Table에 온프레미스 경로 추가
- [ ] VPN 터널 로깅 활성화 (CloudWatch Logs)

---

# Part 2: Cisco CSR (Customer Gateway) 구성

## 개요

Cisco CSR 1000v에서 IKEv2 인증서 기반 IPsec VPN을 구성합니다.
AWS VPN은 터널 2개를 제공하므로 두 터널 모두 구성합니다.

### 필요 정보 (예시 값 - 실제 환경에 맞게 교체)

| 항목 | Tunnel 1 | Tunnel 2 |
|------|----------|----------|
| AWS Outside IP | `52.10.10.10` | `54.20.20.20` |
| AWS Inside IP | `169.254.10.1/30` | `169.254.20.1/30` |
| CGW Inside IP | `169.254.10.2/30` | `169.254.20.2/30` |
| AWS BGP ASN | `64512` | `64512` |
| CGW BGP ASN | `65000` | `65000` |
| On-Prem Network | `172.20.85.0/24` | `172.20.85.0/24` |

---

## 1단계: Cisco CSR에 인증서 설치

Cisco CSR에 접속한 후 인증서를 설치합니다.

> 📌 **방법 선택 가이드**:
> - **2.1 PEM Terminal Import** (권장): Serial/SSH 터미널만으로 설치 가능, 추가 인프라 불필요
> - **2.2 PKCS12 Import**: TFTP/SCP 서버가 있는 환경에서 사용

### 2.1 PEM Terminal Import 방식 (권장)

TFTP/SCP 없이 **serial 또는 SSH 터미널로만** 연결된 환경에서 사용합니다.
ACM에서 내보낸 인증서와 개인키를 터미널에 직접 붙여넣어 임포트합니다.

**사용할 파일 (ACM에서 다운로드):**
| 파일명 | 내용 | 용도 |
|--------|------|------|
| `certificate_chain.txt` | CA 인증서 2개 (Sub CA + Root CA) | CA trustpoint 인증 |
| `certificate.txt` | 디바이스 인증서 1개 | 디바이스 인증서 임포트 |
| `private_key.txt` | 암호화된 개인키 | 개인키 임포트 |

#### 2.1.1 Root CA Trustpoint 설정

```
crypto pki trustpoint AWS-ROOT-CA
 enrollment terminal
 revocation-check none
 exit

crypto pki authenticate AWS-ROOT-CA
```

**붙여넣을 내용**: `certificate_chain.txt`의 **두 번째** 인증서 블록
```
-----BEGIN CERTIFICATE-----
MIIDxTCCAq2gAwIBAgIRAL... (Root CA 인증서)
-----END CERTIFICATE-----
```
붙여넣은 후 `quit` 입력 → `yes` 입력하여 수락

#### 2.1.2 Subordinate CA Trustpoint 설정

```
crypto pki trustpoint AWS-SUB-CA
 enrollment terminal
 revocation-check none
 exit

crypto pki authenticate AWS-SUB-CA
```

**붙여넣을 내용**: `certificate_chain.txt`의 **첫 번째** 인증서 블록
```
-----BEGIN CERTIFICATE-----
MIIDzTCCArWgAwIBAgIQB... (Subordinate CA 인증서)
-----END CERTIFICATE-----
```
붙여넣은 후 `quit` 입력 → `yes` 입력하여 수락

#### 2.1.3 디바이스 인증서 및 개인키 임포트

```
crypto pki trustpoint AWS-VPN-CERT
 enrollment terminal
 revocation-check none
 exit

crypto pki import AWS-VPN-CERT pem terminal password <ACM-passphrase>
```

> `<ACM-passphrase>`는 ACM에서 인증서 내보낼 때 설정한 passphrase입니다.

프롬프트가 순서대로 나타납니다. 각 프롬프트에 해당 내용을 붙여넣으세요:

**1단계 - CA certificate 프롬프트:**
```
% Enter CA certificate:
```
→ `certificate_chain.txt` **전체 내용** 붙여넣기 (2개 인증서 모두 포함)
```
-----BEGIN CERTIFICATE-----
(Subordinate CA 인증서)
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
(Root CA 인증서)
-----END CERTIFICATE-----
```
붙여넣은 후 `quit` 입력

**2단계 - Device certificate 프롬프트:**
```
% Enter device certificate:
```
→ `certificate.txt` **전체 내용** 붙여넣기
```
-----BEGIN CERTIFICATE-----
(디바이스 인증서)
-----END CERTIFICATE-----
```
붙여넣은 후 `quit` 입력

**3단계 - Private key 프롬프트:**
```
% Enter private key:
```
→ `private_key.txt` **전체 내용** 붙여넣기 (암호화된 상태 그대로)
```
-----BEGIN ENCRYPTED PRIVATE KEY-----
(암호화된 개인키)
-----END ENCRYPTED PRIVATE KEY-----
```
붙여넣은 후 `quit` 입력

> ✅ passphrase가 올바르면 자동으로 복호화되어 임포트됩니다.

#### 2.1.4 Trustpoint 재설정이 필요한 경우

잘못 입력한 경우 trustpoint를 삭제하고 다시 시작할 수 있습니다:
```
! 기존 trustpoint 삭제
no crypto pki trustpoint AWS-ROOT-CA
no crypto pki trustpoint AWS-SUB-CA
no crypto pki trustpoint AWS-VPN-CERT

! 2.1.1부터 다시 시작
```

---

### 2.2 PKCS12를 사용한 인증서 임포트 (TFTP/SCP 환경)

TFTP 또는 SCP 서버가 있는 환경에서 사용합니다.
ACM에서 내보낸 인증서와 개인키를 PKCS12 형식으로 변환하여 임포트합니다.

#### 2.2.1 로컬 PC에서 PKCS12 파일 생성

ACM에서 다운로드한 파일들을 사용합니다:
- `private_key.txt` - 암호화된 개인키
- `certificate.txt` - 디바이스 인증서
- `certificate_chain.txt` - CA 체인 (Subordinate CA + Root CA)

```bash
# 1. 암호화된 개인키 복호화
openssl rsa -in private_key.txt -out decrypted_private_key.pem
# passphrase 입력 프롬프트에서 ACM 내보내기 시 설정한 passphrase 입력

# 2. PKCS12 파일 생성 (인증서 + 개인키 + CA 체인 통합)
openssl pkcs12 -export \
  -in certificate.txt \
  -inkey decrypted_private_key.pem \
  -certfile certificate_chain.txt \
  -out vpn_cert.p12 \
  -password pass:CiscoImport123
```

**생성된 파일 확인:**
```bash
ls
# certificate_chain.txt       - ACM에서 다운로드 (CA 체인)
# certificate.txt             - ACM에서 다운로드 (디바이스 인증서)
# private_key.txt             - ACM에서 다운로드 (암호화된 개인키)
# decrypted_private_key.pem   - 복호화된 개인키 (새로 생성)
# vpn_cert.p12                - PKCS12 파일 (새로 생성) ← Cisco CSR에 임포트할 파일
```

> ⚠️ **보안 주의**: 작업 완료 후 민감한 파일을 삭제하세요:
> ```bash
> # 복호화된 개인키는 반드시 삭제 (평문 상태로 보안 위험)
> rm decrypted_private_key.pem
>
> # 암호화된 개인키도 더 이상 필요 없으면 삭제
> rm private_key.txt
> ```
> `vpn_cert.p12`는 Cisco CSR 임포트가 완료될 때까지 보관합니다.

#### 2.2.2 CA Trustpoint 사전 설정

PKCS12 임포트 전에 Root CA와 Subordinate CA를 먼저 설치합니다.
(2.1.1, 2.1.2와 동일한 방법)

#### 2.2.3 PKCS12 파일 임포트

PKCS12 파일을 Cisco CSR에 접근 가능한 위치(TFTP/SCP)에 복사 후:

```
! TFTP 사용 시
crypto pki import AWS-VPN-CERT pkcs12 tftp://192.168.1.100/vpn_cert.p12 password CiscoImport123

! 또는 SCP 사용 시
crypto pki import AWS-VPN-CERT pkcs12 scp://user@192.168.1.100/vpn_cert.p12 password CiscoImport123
```

> 📌 PKCS12 임포트는 인증서, 개인키, CA 체인을 한 번에 가져오므로 오류가 적습니다.

---

### 2.3 인증서 설치 검증

```
show crypto pki certificates
show crypto pki trustpoints status
```

다음을 확인:
- AWS-ROOT-CA: CA certificate 설치됨
- AWS-SUB-CA: CA certificate 설치됨
- AWS-VPN-CERT: CA certificate + Router certificate 설치됨

---

## 2단계: IKEv2 구성

### 3.1 IKEv2 Proposal

```
crypto ikev2 proposal AWS-IKEv2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit
```

### 3.2 IKEv2 Policy

```
crypto ikev2 policy AWS-IKEv2-POLICY
 match fvrf any
 proposal AWS-IKEv2-PROPOSAL
 exit
```

### 3.3 IKEv2 Profile (Tunnel 1)

```
crypto ikev2 profile AWS-IKEv2-PROFILE-T1
 match identity remote address 52.10.10.10 255.255.255.255
 identity local dn
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint AWS-VPN-CERT
 dpd 10 3 periodic
 lifetime 28800
 exit
```

### 3.4 IKEv2 Profile (Tunnel 2)

```
crypto ikev2 profile AWS-IKEv2-PROFILE-T2
 match identity remote address 54.20.20.20 255.255.255.255
 identity local dn
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint AWS-VPN-CERT
 dpd 10 3 periodic
 lifetime 28800
 exit
```

> 📌 `identity local dn`은 인증서의 Distinguished Name을 IKEv2 identity로 사용합니다.
> `authentication local rsa-sig` / `authentication remote rsa-sig`는 양쪽 모두 인증서 기반 인증을 사용한다는 의미입니다.

---

## 3단계: IPsec 구성

### 4.1 IPsec Transform Set

```
crypto ipsec transform-set AWS-TS esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit
```

### 4.2 IPsec Profile (Tunnel 1)

```
crypto ipsec profile AWS-IPSEC-PROFILE-T1
 set transform-set AWS-TS
 set ikev2-profile AWS-IKEv2-PROFILE-T1
 set security-association lifetime seconds 3600
 exit
```

### 4.3 IPsec Profile (Tunnel 2)

```
crypto ipsec profile AWS-IPSEC-PROFILE-T2
 set transform-set AWS-TS
 set ikev2-profile AWS-IKEv2-PROFILE-T2
 set security-association lifetime seconds 3600
 exit
```

---

## 4단계: Tunnel 인터페이스 구성

### 5.1 Tunnel 1

```
interface Tunnel1
 description AWS-VPN-Tunnel-1
 ip address 169.254.10.2 255.255.255.252
 ip tcp adjust-mss 1379
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 52.10.10.10
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-T1
 no shutdown
 exit
```

### 5.2 Tunnel 2

```
interface Tunnel2
 description AWS-VPN-Tunnel-2
 ip address 169.254.20.2 255.255.255.252
 ip tcp adjust-mss 1379
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 54.20.20.20
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-T2
 no shutdown
 exit
```

> 📌 `tunnel source`는 인터넷에 연결된 인터페이스를 지정합니다. DHCP로 IP를 받는 인터페이스명을 사용하세요.
> `ip tcp adjust-mss 1379`는 IPsec 오버헤드를 고려한 TCP MSS 조정입니다.

---

## 5단계: BGP 구성

```
router bgp 65000
 bgp log-neighbor-changes
 !
 ! Tunnel 1 - AWS BGP Peer
 neighbor 169.254.10.1 remote-as 64512
 neighbor 169.254.10.1 timers 10 30 30
 !
 ! Tunnel 2 - AWS BGP Peer
 neighbor 169.254.20.1 remote-as 64512
 neighbor 169.254.20.1 timers 10 30 30
 !
 address-family ipv4 unicast
  ! 온프레미스 네트워크 광고
  network 172.20.85.0 mask 255.255.255.0
  !
  neighbor 169.254.10.1 activate
  neighbor 169.254.10.1 soft-reconfiguration inbound
  neighbor 169.254.20.1 activate
  neighbor 169.254.20.1 soft-reconfiguration inbound
 exit-address-family
 exit
```

> 📌 BGP timers `10 30 30`은 keepalive=10초, holdtime=30초, min-holdtime=30초를 의미합니다.
> - Keepalive: BGP 피어에게 주기적으로 KEEPALIVE 메시지를 보내는 간격
> - Holdtime: 이 시간 동안 피어로부터 메시지가 없으면 세션 다운으로 간주
> - AWS VPN은 기본적으로 holdtime 30초를 사용하므로 이에 맞춘 설정입니다.
>
> `network` 명령어에는 AWS로 광고할 온프레미스 네트워크 CIDR을 입력합니다.

---

## 6단계: 추가 설정 (권장)

### 7.1 IKEv2 로깅 활성화

```
crypto logging ikev2
```

### 7.2 IP SLA를 이용한 터널 모니터링 (선택사항)

```
ip sla 1
 icmp-echo 169.254.10.1 source-interface Tunnel1
 frequency 10
ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 169.254.20.1 source-interface Tunnel2
 frequency 10
ip sla schedule 2 life forever start-time now
```

### 7.3 설정 저장

```
write memory
```

---

# Part 3: 검증 및 트러블슈팅

## 1. Cisco CSR 측 검증 명령어

### 1.1 인증서 확인
```
show crypto pki certificates
show crypto pki trustpoints status
```

### 1.2 IKEv2 SA 확인
```
show crypto ikev2 sa
show crypto ikev2 sa detailed
```

성공 시 출력 예시:
```
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         <CGW-IP>/4500         52.10.10.10/4500      none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:14, Auth sign: RSA, Auth verify: RSA
      Life/Active Time: 28800/120 sec
```

### 1.3 IPsec SA 확인
```
show crypto ipsec sa
```

성공 시 `pkts encaps` 및 `pkts decaps` 카운터가 증가해야 합니다.

### 1.4 BGP 확인
```
show ip bgp summary
show ip bgp neighbors 169.254.10.1
show ip route bgp
```

성공 시 BGP state가 `Established`이고 AWS VPC CIDR이 BGP 라우트로 표시되어야 합니다.

### 1.5 터널 상태 확인
```
show interface Tunnel1
show interface Tunnel2
```

두 터널 모두 `up/up` 상태여야 합니다.

### 1.6 연결 테스트
```
ping 169.254.10.1 source 169.254.10.2
ping 169.254.20.1 source 169.254.20.2
```

---

## 2. AWS 측 검증

### 2.1 VPN 터널 상태 확인

1. AWS Console → **VPC** → **Site-to-Site VPN Connections**
2. VPN 선택 → **Tunnel details** 탭
3. 두 터널 모두 **Status: UP** 인지 확인

### 2.2 TGW Route Table 확인

1. **Transit Gateway Route Tables** → 라우트 테이블 선택
2. **Routes** 탭에서 BGP로 전파된 온프레미스 네트워크 경로 확인
   - 예: `172.20.85.0/24` → VPN Attachment

### 2.3 엔드투엔드 통신 테스트

AWS EC2 인스턴스에서 온프레미스 네트워크로 ping 테스트:
```bash
ping 172.20.85.1  # 온프레미스 디바이스 IP
```

> ⚠️ Security Group에서 ICMP 인바운드를 허용해야 합니다.

---

## 3. 일반적인 트러블슈팅

### 문제 1: IKEv2 SA가 설정되지 않음

**원인 확인:**
```
debug crypto ikev2
debug crypto ikev2 error
```

**일반적인 원인:**
- 인증서 체인 불완전 (Root CA 또는 Sub CA 누락)
- 인증서 만료
- IKEv2 proposal 불일치 (encryption, integrity, DH group)
- AWS VPN 터널 엔드포인트 IP 오류

**해결:**
```
show crypto pki certificates
show clock
! 시간이 맞는지 확인 - 인증서 검증에 시간이 중요
ntp server pool.ntp.org
```

### 문제 2: IPsec SA는 있지만 트래픽이 안 됨

**확인:**
```
show crypto ipsec sa
! pkts encaps/decaps 카운터 확인
```

**일반적인 원인:**
- 라우팅 문제 (BGP 경로 미수신)
- Security Group/NACL에서 트래픽 차단
- VPC Route Table에 온프레미스 경로 누락

### 문제 3: BGP가 Established되지 않음

**확인:**
```
show ip bgp summary
debug ip bgp
```

**일반적인 원인:**
- BGP ASN 불일치
- Tunnel inside IP 주소 오류
- IPsec 터널이 down 상태

### 문제 4: DHCP IP 변경 후 VPN 재연결

DHCP로 IP가 변경되면 터널이 자동으로 재연결을 시도합니다.
재연결이 안 되면:
```
clear crypto ikev2 sa
! 또는 터널 인터페이스 재시작
interface Tunnel1
 shutdown
 no shutdown
interface Tunnel2
 shutdown
 no shutdown
```

---

## 4. 디버그 명령어 정리

| 명령어 | 용도 |
|---------|------|
| `debug crypto ikev2` | IKEv2 협상 디버그 |
| `debug crypto ikev2 error` | IKEv2 오류만 표시 |
| `debug crypto ikev2 packet` | IKEv2 패킷 상세 |
| `debug crypto ipsec` | IPsec SA 협상 디버그 |
| `debug crypto pki transactions` | PKI 인증서 처리 디버그 |
| `debug crypto pki validation` | 인증서 검증 디버그 |
| `debug ip bgp` | BGP 디버그 |
| `undebug all` | 모든 디버그 해제 |

> ⚠️ 디버그는 트러블슈팅 후 반드시 `undebug all`로 해제하세요. 프로덕션 환경에서 장시간 디버그를 켜두면 성능에 영향을 줄 수 있습니다.

---

## 5. 고가용성 (HA) 구성

AWS Site-to-Site VPN은 기본적으로 **2개의 터널**을 제공합니다. 이 두 터널을 활용한 고가용성 전략을 설명합니다.

### 5.1 터널 구성 옵션

| 구성 방식 | 설명 | 장점 | 단점 |
|----------|------|------|------|
| **Active/Standby** | 하나의 터널만 활성, 다른 터널은 대기 | 단순한 구성, 예측 가능한 트래픽 경로 | 대역폭이 단일 터널로 제한됨 |
| **Active/Active (ECMP)** | 두 터널 모두 활성, 트래픽 분산 | 대역폭 2배, 로드밸런싱 | 비대칭 라우팅 주의 필요 |

### 5.2 Active/Standby 구성 (BGP)

BGP AS-PATH prepending을 사용하여 선호 터널을 설정합니다.

```
! Cisco CSR - Tunnel 2를 백업으로 설정
route-map AWS-BACKUP permit 10
 set as-path prepend 65000 65000

router bgp 65000
 address-family ipv4 unicast
  neighbor 169.254.20.1 route-map AWS-BACKUP out
 exit-address-family
```

이 설정으로 Tunnel 1이 선호되고, Tunnel 1 장애 시 자동으로 Tunnel 2로 페일오버됩니다.

### 5.3 Active/Active (ECMP) 구성

Transit Gateway에서 ECMP를 활성화하면 두 터널에 트래픽을 분산합니다.

**TGW 측 확인:**
- Transit Gateway 생성 시 **VPN ECMP support: Enable** 확인

**Cisco CSR 측:**
```
! 두 경로의 비용을 동일하게 설정
router bgp 65000
 maximum-paths 2
```

> ⚠️ **ECMP 사용 시 주의사항:**
> - 동일 플로우의 패킷이 다른 터널로 분산될 수 있음
> - 애플리케이션에 따라 세션 문제가 발생할 수 있음
> - 대부분의 경우 Active/Standby가 더 안정적

### 5.4 터널 장애 감지 시간

| 메커니즘 | 기본값 | 장애 감지 시간 |
|---------|-------|--------------|
| BGP Holdtime | 30초 | 최대 30초 |
| DPD (Dead Peer Detection) | 10초 간격, 3회 재시도 | 약 30초 |
| BFD (Bidirectional Forwarding Detection) | 지원 시 | 1초 미만 |

> 📌 더 빠른 장애 감지가 필요하면 AWS VPN의 BFD 지원 여부를 확인하세요 (리전별 상이).

---

# Part 4: Cisco CSR 전체 구성 요약 (Running Config)

아래는 Cisco CSR의 VPN 관련 전체 구성을 한눈에 볼 수 있도록 정리한 것입니다.
실제 환경의 IP 주소와 ASN으로 교체하여 사용하세요.

```
! ============================================
! Cisco CSR 1000v - AWS S2S VPN Certificate Config
! ============================================

! --- NTP 설정 (인증서 검증에 시간 동기화 필수) ---
ntp server pool.ntp.org

! --- RSA 키페어 생성 ---
crypto key generate rsa label AWS-VPN-KEY modulus 2048

! --- Trustpoint: Root CA ---
crypto pki trustpoint AWS-ROOT-CA
 enrollment terminal
 revocation-check none

! --- Trustpoint: Subordinate CA ---
crypto pki trustpoint AWS-SUB-CA
 enrollment terminal
 revocation-check none

! --- Trustpoint: Device Certificate ---
crypto pki trustpoint AWS-VPN-CERT
 enrollment terminal
 revocation-check none
 rsakeypair AWS-VPN-KEY

! --- IKEv2 Proposal ---
crypto ikev2 proposal AWS-IKEv2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14

! --- IKEv2 Policy ---
crypto ikev2 policy AWS-IKEv2-POLICY
 match fvrf any
 proposal AWS-IKEv2-PROPOSAL

! --- IKEv2 Profile: Tunnel 1 ---
crypto ikev2 profile AWS-IKEv2-PROFILE-T1
 match identity remote address 52.10.10.10 255.255.255.255
 identity local dn
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint AWS-VPN-CERT
 dpd 10 3 periodic
 lifetime 28800

! --- IKEv2 Profile: Tunnel 2 ---
crypto ikev2 profile AWS-IKEv2-PROFILE-T2
 match identity remote address 54.20.20.20 255.255.255.255
 identity local dn
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint AWS-VPN-CERT
 dpd 10 3 periodic
 lifetime 28800

! --- IPsec Transform Set ---
crypto ipsec transform-set AWS-TS esp-aes 256 esp-sha256-hmac
 mode tunnel

! --- IPsec Profile: Tunnel 1 ---
crypto ipsec profile AWS-IPSEC-PROFILE-T1
 set transform-set AWS-TS
 set ikev2-profile AWS-IKEv2-PROFILE-T1
 set security-association lifetime seconds 3600

! --- IPsec Profile: Tunnel 2 ---
crypto ipsec profile AWS-IPSEC-PROFILE-T2
 set transform-set AWS-TS
 set ikev2-profile AWS-IKEv2-PROFILE-T2
 set security-association lifetime seconds 3600

! --- Tunnel 1 Interface ---
interface Tunnel1
 description AWS-VPN-Tunnel-1
 ip address 169.254.10.2 255.255.255.252
 ip tcp adjust-mss 1379
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 52.10.10.10
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-T1
 no shutdown

! --- Tunnel 2 Interface ---
interface Tunnel2
 description AWS-VPN-Tunnel-2
 ip address 169.254.20.2 255.255.255.252
 ip tcp adjust-mss 1379
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 54.20.20.20
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-T2
 no shutdown

! --- BGP ---
router bgp 65000
 bgp log-neighbor-changes
 neighbor 169.254.10.1 remote-as 64512
 neighbor 169.254.10.1 timers 10 30 30
 neighbor 169.254.20.1 remote-as 64512
 neighbor 169.254.20.1 timers 10 30 30
 !
 address-family ipv4 unicast
  network 172.20.85.0 mask 255.255.255.0
  neighbor 169.254.10.1 activate
  neighbor 169.254.10.1 soft-reconfiguration inbound
  neighbor 169.254.20.1 activate
  neighbor 169.254.20.1 soft-reconfiguration inbound
 exit-address-family

! --- IKEv2 Logging ---
crypto logging ikev2

! --- Save ---
end
write memory
```

---

# Part 5: 인증서 갱신 및 유지보수

## 인증서 만료 관리

ACM에서 발급한 프라이빗 인증서는 만료일이 있으므로 주기적으로 갱신해야 합니다.

### AWS 측 인증서 갱신

1. ACM에서 새 프라이빗 인증서 발급 (동일한 Subordinate CA 사용)
2. Customer Gateway를 새 인증서 ARN으로 업데이트
   - 또는 동일한 CA 체인의 인증서라면 자동으로 인증 가능

> 📌 AWS 문서에 따르면, 동일한 CA 체인의 인증서라면 Customer Gateway를 재생성하지 않아도 VPN 연결이 유지됩니다.

### Cisco CSR 측 인증서 갱신

1. 새 인증서를 ACM에서 내보내기
2. Cisco CSR에서 기존 인증서 제거 후 새 인증서 설치:

```
! 기존 인증서 제거
no crypto pki certificate chain AWS-VPN-CERT

! 새 인증서 설치 (위의 2단계 절차 반복)
crypto pki authenticate AWS-VPN-CERT
crypto pki import AWS-VPN-CERT certificate
```

3. IKEv2 SA 초기화:
```
clear crypto ikev2 sa
```

---

# Part 6: 보안 권장사항

1. **개인키 보호**: 복호화된 개인키 파일은 작업 완료 후 즉시 삭제
2. **NTP 동기화**: 인증서 검증에 시간이 중요하므로 NTP 필수 구성
3. **인증서 만료 모니터링**: CloudWatch 알람 또는 Cisco EEM으로 만료 전 알림 설정
4. **접근 제어**: Cisco CSR의 SSH 접근을 특정 IP로 제한
5. **로그 모니터링**: IKEv2 로그를 syslog 서버로 전송
6. **AWS CloudWatch**: VPN 터널 상태 모니터링 및 알람 설정

---

# Part 7: 롤백 절차

문제 발생 시 리소스를 안전하게 삭제하는 순서입니다.

> ⚠️ **중요**: 리소스 간 의존성이 있으므로 반드시 아래 순서대로 삭제해야 합니다.

## AWS 리소스 삭제 순서

```
1. VPN Connection 삭제
   └─ VPN Attachment가 자동으로 삭제됨
       │
2. Customer Gateway 삭제
       │
3. Transit Gateway 삭제 (필요시)
   └─ 먼저 모든 TGW Attachment 삭제 필요
       │
4. ACM 인증서 삭제
   └─ Customer Gateway에서 사용 중이면 삭제 불가
       │
5. AWS Private CA 삭제
   └─ Subordinate CA 먼저 삭제
   └─ Root CA 삭제
   └─ 인증서가 발급되어 있으면 비활성화 후 삭제
```

### AWS CLI 삭제 명령어

```bash
# 1. VPN Connection 삭제
aws ec2 delete-vpn-connection --vpn-connection-id vpn-xxxxxxxxx

# 2. Customer Gateway 삭제
aws ec2 delete-customer-gateway --customer-gateway-id cgw-xxxxxxxxx

# 3. TGW VPC Attachment 삭제 (TGW 삭제 전 필요)
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id tgw-attach-xxxxxxxxx

# 4. Transit Gateway 삭제
aws ec2 delete-transit-gateway --transit-gateway-id tgw-xxxxxxxxx

# 5. ACM 인증서 삭제
aws acm delete-certificate --certificate-arn arn:aws:acm:REGION:ACCOUNT:certificate/xxx

# 6. Private CA 비활성화 및 삭제
# Subordinate CA
aws acm-pca update-certificate-authority \
  --certificate-authority-arn arn:aws:acm-pca:REGION:ACCOUNT:certificate-authority/xxx \
  --status DISABLED

aws acm-pca delete-certificate-authority \
  --certificate-authority-arn arn:aws:acm-pca:REGION:ACCOUNT:certificate-authority/xxx

# Root CA (동일 절차)
aws acm-pca update-certificate-authority \
  --certificate-authority-arn arn:aws:acm-pca:REGION:ACCOUNT:certificate-authority/yyy \
  --status DISABLED

aws acm-pca delete-certificate-authority \
  --certificate-authority-arn arn:aws:acm-pca:REGION:ACCOUNT:certificate-authority/yyy
```

> 📌 **Private CA 삭제 주의사항:**
> - 삭제된 CA는 복구할 수 없습니다
> - CA를 삭제하면 해당 CA로 발급된 모든 인증서가 무효화됩니다
> - 비용 절감이 목적이라면 삭제 대신 비활성화를 고려하세요

## Cisco CSR 구성 롤백

```
! VPN 관련 구성 제거 (역순으로)

! 1. BGP 피어 제거
router bgp 65000
 no neighbor 169.254.10.1
 no neighbor 169.254.20.1
 exit

! 2. Tunnel 인터페이스 제거
no interface Tunnel1
no interface Tunnel2

! 3. IPsec Profile 제거
no crypto ipsec profile AWS-IPSEC-PROFILE-T1
no crypto ipsec profile AWS-IPSEC-PROFILE-T2

! 4. Transform Set 제거
no crypto ipsec transform-set AWS-TS

! 5. IKEv2 Profile 제거
no crypto ikev2 profile AWS-IKEv2-PROFILE-T1
no crypto ikev2 profile AWS-IKEv2-PROFILE-T2

! 6. IKEv2 Policy/Proposal 제거
no crypto ikev2 policy AWS-IKEv2-POLICY
no crypto ikev2 proposal AWS-IKEv2-PROPOSAL

! 7. Trustpoint 및 인증서 제거
no crypto pki trustpoint AWS-VPN-CERT
no crypto pki trustpoint AWS-SUB-CA
no crypto pki trustpoint AWS-ROOT-CA

! 8. RSA 키페어 제거
crypto key zeroize rsa AWS-VPN-KEY

! 저장
write memory
```

---

# 참고 자료

- [AWS Site-to-Site VPN 인증서 기반 VPN 생성](https://aws.amazon.com/premiumsupport/knowledge-center/vpn-certificate-based-site-to-site/)
- [AWS Site-to-Site VPN 터널 인증 옵션](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-tunnel-authentication-options.html)
- [AWS Private CA 사용 설명서](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html)
- [Transit Gateway VPN Attachment 생성](https://docs.aws.amazon.com/vpn/latest/s2svpn/create-tgw-vpn-attachment.html)
- [Cisco IOS-XE IKEv2 구성 가이드](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-cfg-ikev2-flex.html)
- [Cisco IOS-XE PKI 인증서 구성](https://www.cisco.com/c/en/us/support/docs/security-vpn/public-key-infrastructure-pki/220422-configure-ca-signed-certificates-with-io.html)

---

> 본 문서는 AWS 공식 문서와 Cisco 공식 문서를 기반으로 작성되었습니다.
> 실제 환경에서는 IP 주소, ASN, CIDR 등을 실제 값으로 교체하여 사용하세요.
