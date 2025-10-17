# 문제 상황

<img width="421" alt="image" src="https://github.com/user-attachments/assets/a30468c9-73fb-4bc6-99b7-de58b6be3698" />


최근 배포해놓은 EC2 서버가 주기적으로 다운되는 현상이 발생하고 있다. 지금까지는 서버를 중지하고 재시작하는 방식으로 임시 대응해왔지만 현재 프로젝트는 실제 서비스 출시를 목표로 하고 있어 더 이상 이러한 방식으로는 문제를 해결할 수 없다.

서비스 운영 중 서버가 갑작스럽게 중단되면 사용자 접근이 불가능해지고 이는 서비스의 `가용성(Availability)` 을 심각하게 저해하는 요소가 된다. 따라서 이 문제를 근본적으로 해결하기 위한 원인 분석과 대응책이 지금 상황에서 필수이다. 이 문제를 해결하기 위한 방법을 찾아보자! 

먼저 AWS CloudWatch에서 제공하는 모니터링 지표를 통해 서버 상태를 확인해보았다.

<img width="853" alt="image" src="https://github.com/user-attachments/assets/212c2971-5c19-4a3d-b4cf-c7fb1bd7a3c4" />


2025년 5월 30일 우리나라 시간 기준으로 오후 3시 30분경 CPU 사용률이 100%에 도달한 것을 확인할 수 있었다. 이 시점과 동시에 서버 접속이 불가능해진 것 같고, 이후 서버가 정상적으로 작동하지 않았다.

<img width="848" alt="image" src="https://github.com/user-attachments/assets/ded983cd-683f-411f-bceb-43062000c4e4" />


인스턴스 상태 검사 로그를 확인한 결과, CPU 사용률이 100%를 기록한 직후부터 상태 검사에 연달아 실패하고 있는 것을 확인했다. 이로 미루어볼 때 CPU 사용률 급증이 서버 다운의 직접적인 원인으로 보인다.

# 문제 원인

많은 EC2 인스턴스 다운 사례를 살펴보니 CPU 사용률이 100%까지 치솟은 직후 서버가 다운되는 경우가 흔하게 찾을 수 있었다. 나 역시 이와 비슷한 현상을 겪고 있기에 동일한 원인이 있는지 파악해보기로 했다.

그래서 먼저 CPU 사용률이 왜 100%까지 치솟았는지 그 원인을 파악해보기로 했다. 다양한 글들을 참고해본 결과 `메모리 부족(OOM, Out Of Memory)` 이 CPU 100% 사용률의 주요 원인 중 하나라는 이야기를 자주 접할 수 있었다.

실제로 내 EC2 인스턴스도 메모리 부족 때문에 문제가 발생한 것인지 확인하기 위해 서버에 직접 접속해서 다음 명령어로 OOM 로그를 조회해보았다

```bash
 sudo grep -i 'oom' /var/log/syslog
```

> `var/log/syslog`는 리눅스와 유닉스 기반 시스템에서 로그 메시지를 수집, 기록, 관리하기 위한 표준화된 로깅 시스템이다. syslog는 시스템 및 애플리케이션 이벤트, 경고, 오류 등의 메시지를 기록하여 시스템 상태의 모니터링, 문제 해결, 보안 감사 등에 활용된다.
> 

위 명령을 실행한 결과 다음과 같은 로그가 출력되었다

```bash
May 30 07:19:52 ip-172-31-10-132 kernel: [329112.718596] unattended-upgr invoked oom-killer: gfp_mask=0x140dca(GFP_HIGHUSER_MOVABLE|__GFP_COMP|__GFP_ZERO), order=0, oom_score_adj=0
...
May 30 07:19:53 ip-172-31-10-132 kernel: [329112.718927] Out of memory: Killed process 1227 (java) total-vm:2452324kB, anon-rss:507824kB, file-rss:128kB, shmem-rss:0kB, UID:0 pgtables:1372kB oom_score_adj:0
May 30 07:20:05 ip-172-31-10-132 containerd[391]: time="2025-05-30T07:19:59.721138517Z" level=error msg="publish OOM event" error="context deadline exceeded"
May 30 07:20:05 ip-172-31-10-132 systemd[1]: docker-b0e18cc51e44524fec7ced5bcdfc1cea61bcaddb98dd0f6aef20205b95cc13ba.scope: A process of this unit has been killed by the OOM killer.
May 30 07:20:45 ip-172-31-10-132 systemd[1]: system.slice: A process of this unit has been killed by the OOM killer.
May 31 06:05:28 ip-172-31-10-132 containerd[402]: time="2025-05-31T06:05:28.121195309Z" level=info msg="Start cri plugin with config {PluginConfig:{ContainerdConfig:{Snapshotter:overlayfs DefaultRuntimeName:runc DefaultRuntime:{Type: Path: Engine: PodAnnotations:[] ContainerAnnotations:[] Root: Options:map[] PrivilegedWithoutHostDevices:false PrivilegedWithoutHostDevicesAllDevicesAllowed:false BaseRuntimeSpec: NetworkPluginConfDir: NetworkPluginMaxConfNum:0 Snapshotter: SandboxMode:} UntrustedWorkloadRuntime:{Type: Path: Engine: PodAnnotations:[] ContainerAnnotations:[] Root: Options:map[] PrivilegedWithoutHostDevices:false PrivilegedWithoutHostDevicesAllDevicesAllowed:false BaseRuntimeSpec: NetworkPluginConfDir: NetworkPluginMaxConfNum:0 Snapshotter: SandboxMode:} Runtimes:map[runc:{Type:io.containerd.runc.v2 Path: Engine: PodAnnotations:[] ContainerAnnotations:[] Root: Options:map[BinaryName: CriuImagePath: CriuPath: CriuWorkPath: IoGid:0 IoUid:0 NoNewKeyring:false NoPivotRoot:false Root: ShimCgroup: SystemdCgroup:false] PrivilegedWithoutHostDevices:false PrivilegedWithoutHostDevicesAllDevicesAllowed:false BaseRuntimeSpec: NetworkPluginConfDir: NetworkPluginMaxConfNum:0 Snapshotter: SandboxMode:podsandbox}] NoPivot:false DisableSnapshotAnnotations:true DiscardUnpackedLayers:false IgnoreBlockIONotEnabledErrors:false IgnoreRdtNotEnabledErrors:false} CniConfig:{NetworkPluginBinDir:/opt/cni/bin NetworkPluginConfDir:/etc/cni/net.d NetworkPluginMaxConfNum:1 NetworkPluginSetupSerially:false NetworkPluginConfTemplate: IPPreference:} Registry:{ConfigPath: Mirrors:map[] Configs:map[] Auths:map[] Headers:map[]} ImageDecryption:{KeyModel:node} DisableTCPService:true StreamServerAddress:127.0.0.1 StreamServerPort:0 StreamIdleTimeout:4h0m0s EnableSelinux:false SelinuxCategoryRange:1024 SandboxImage:registry.k8s.io/pause:3.8 StatsCollectPeriod:10 SystemdCgroup:false EnableTLSStreaming:false X509KeyPairStreaming:{TLSCertFile: TLSKeyFile:} MaxContainerLogLineSize:16384 DisableCgroup:false DisableApparmor:false RestrictOOMScoreAdj:false MaxConcurrentDownloads:3 DisableProcMount:false UnsetSeccompProfile: TolerateMissingHugetlbController:true DisableHugetlbController:true DeviceOwnershipFromSecurityContext:false IgnoreImageDefinedVolumes:false NetNSMountsUnderStateDir:false EnableUnprivilegedPorts:false EnableUnprivilegedICMP:false EnableCDI:false CDISpecDirs:[/etc/cdi /var/run/cdi] ImagePullProgressTimeout:5m0s DrainExecSyncIOTimeout:0s ImagePullWithSyncFs:false IgnoreDeprecationWarnings:[]} ContainerdRootDir:/var/lib/containerd ContainerdEndpoint:/run/containerd/containerd.sock RootDir:/var/lib/containerd/io.containerd.grpc.v1.cri StateDir:/run/containerd/io.containerd.grpc.v1.cri}"
```

- 로그를 분석한 결과 다음과 같은 사실을 알 수 있었다
- `java` 프로세스가 과도한 메모리를 사용해 시스템 전체 메모리가 부족해졌고 결국 커널의 `OOM Killer`가 작동해 해당 프로세스를 강제 종료했으며, Docker 컨테이너 전체가 중단되었다. 이후 시스템의 상태 검사에서도 계속 실패가 발생했다.

그렇다면 메모리 부족이 왜 CPU 사용률 100% 현상으로 이어졌을까? 이 점을 이해하기 위해 관련 내용을 찾아보았고 현재 상황을 바탕으로 다음과 같이 추정할 수 있었다.

- 현재 나는 프리 티어 EC2 인스턴스를 사용 중이며 사양은 1 vCPU, 1GB 메모리로 매우 제한적이다.
- 이처럼 메모리가 부족한 환경에서는 Java 애플리케이션이 필요한 메모리를 확보하지 못해 실행이 반복적으로 중단되었다가 다시 시도되는 현상이 발생할 수 있다.
- 이 과정에서 운영체제는 여러 프로세스를 빠르게 전환하는 컨텍스트 스위칭(context switching) 을 반복하게 되는데 이러한 작업이 과도하게 발생하면 CPU에도 큰 부담이 생기게 되고 결국 CPU 사용률이 급격히 상승하게 된다.

물론 이 가설은 아직 완전히 검증된 것은 아니며, 보다 정확한 원인을 파악하기 위해서는 운영체제와 JVM의 동작 원리에 대한 추가적인 자료 조사가 필요할 것 같다.

또한 명확히 자바 프로세스 어느 곳에서 메모리 부족이 발생했는지 명확한 조사가 필요할 것 같다.

# 문제 해결

이번 문제는 메모리 부족으로 인한 CPU 사용률 증가 현상으로 파악된다. 따라서 먼저 메모리 공간 확보를 해보자.

많은 자료에서 이 문제의 일반적인 해결책으로 Swap 메모리를 활용하고 있다. 나 역시 이 방법을 통해 메모리 부족 문제를 해결해보겠다. 우선, `Swap` 메모리가 무엇인지 간단히 정리해보자.

## Swap 메모리란?

`이것이 컴퓨터 과학이다` 운영체제 이론에서 배운 내용을 토대로 우선 Swap 개념을 정리하면 다음과 같다.

> 메모리에 적재된 프로세스 중 현재 실행되지 않는 프로세스를 `보조기억장치(디스크)`의 스왑 영역으로 옮기는 과정을 스와핑(Swapping) 이라고 한다.
> 

이 과정을 통해 메모리 공간을 확보하고, 그 공간에 다른 프로세스를 적재할 수 있게 된다. 스와핑은 다음 두 가지 방식으로 나뉜다.

- `Swap-Out`: 메모리 → 디스크의 스왑 영역으로 이동
- `Swap-In`: 스왑 영역 → 다시 메모리로 복귀

이러한 개념을 바탕으로 볼 때, Swap 메모리란 물리적 RAM이 부족할 때 디스크 공간 일부를 가상 메모리처럼 활용하는 기술이다.

- 즉, RAM의 확장 공간 역할을 하며, 시스템이 메모리 부족으로 인해 서비스가 중단되거나 프로세스가 종료되는 현상을 방지하는 데 유용하다.
- 단, 디스크는 메모리보다 속도가 훨씬 느리기 때문에, Swap은 성능을 보완하기 위한 임시 수단으로만 사용해야 하며 장기적으로는 애플리케이션 메모리 사용 최적화나 인스턴스 메모리 증설 등의 근본적인 조치가 필요하다.

## EC2에서 Swap 메모리 설정하기

[스왑 파일을 사용하여 Amazon EC2 인스턴스에서 메모리를 스왑 스페이스로 할당합니다.](https://repost.aws/ko/knowledge-center/ec2-memory-swap-file)

aws 공식 문서를 참조하여 Swap 메모리를 설정해보겠다.  AWS EC2 인스턴스에서는 기본적으로 Swap이 설정되어 있지 않기 때문에, 다음 단계를 통해 직접 설정해야 한다. 이 과정은 `swap file` 기반이며 루트 권한이 필요하다.

### 스왑 파일 생성

우선, `dd` 명령을 사용하여 루트 파일 시스템에 스왑 파일을 생성한다. 여기서 `dd` 명령은 지정한 크기만큼의 데이터를 파일로 만들어주는 명령어다.

```bash
sudo dd if=/dev/zero of=/swapfile bs=128M count=16
```

- `if=/dev/zero`: 0값으로 채워진 데이터를 입력 소스로 사용
- `of=/swapfile`: 생성할 대상 파일 경로 (스왑 파일)
- `bs=128M`: 한 블록의 크기를 128MB로 설정
- `count=16`: 128MB짜리 블록을 16개 생성 → 총 2GB (128MB × 16)

통상 swap 메모리 설정 크기는 기존 RAM 용량의 2배를 설정 해준다고 한다. 나는 그럼 2GB를 할당 해주면 된다.

```bash
ubuntu@ip-172-31-10-132:~$ sudo dd if=/dev/zero of=/swapfile bs=128M count=16
16+0 records in
16+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 15.2017 s, 141 MB/s
```

- 출력을 통해 스왑 파일이 정상적으로 생성되었음을 확인할 수 있다.

### 읽기/쓰기 권한 업데이트

스왑 파일은 시스템의 중요한 영역이므로 루트만 접근 가능하도록 권한을 설정해준다.

```bash
sudo chmod 600 /swapfile
```

### Linux 스왑 영역 설정

스왑 파일을 스왑 영역으로 초기화해주는 작업이다.

```bash
sudo mkswap /swapfile
```

```bash
ubuntu@ip-172-31-10-132:~$ sudo mkswap /swapfile
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
no label, UUID=abf4f73f-49f9-4f06-a99b-35c302949b33
```

- 정상적으로 스왑 공간이 설정되었다.

### 스왑 공간에 스왑 파일을 추가

스왑 파일을 활성화시켜 시스템이 실제로 사용할 수 있도록 한다.

```bash
sudo swapon /swapfile
```

### 성공적으로 완료되었는지 확인

```bash
sudo swapon -s
```

```bash
ubuntu@ip-172-31-10-132:~$ sudo swapon -s
Filename                                Type            Size            Used            Priority
/swapfile                               file            2097148         0               -2
```

- 스왑 파일이 파일 형태(`file`)로 등록되었고, 크기가 2097148KB(약 2GB)임을 확인할 수 있다. 현재 사용량이 `0`인 것은 아직 실제 메모리가 충분해서 Swap이 사용되지 않았다는 뜻이다.

### 부팅 시 스왑 파일 활성화

> Swap 설정은 기본적으로 부팅 시 초기화되므로, `/etc/fstab` 파일을 수정해 재부팅 시에도 Swap이 자동으로 활성화되도록 설정한다.
> 

부팅 시 `/etc/fstab` 파일을 편집하여 스왑 파일을 시작할 수 있도록 한다.

1. 편집기에서 파일을 연다.
    
    ```bash
    sudo vi /etc/fstab
    ```
    
2. 파일 끝에 다음 줄을 추가하고, 파일을 저장한 다음 종료한다.
    
    ```bash
    /swapfile swap swap defaults 0 0
    ```
    

적용된 것을 확인하면 다음과 같다.

```bash
ubuntu@ip-172-31-10-132:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           957Mi       652Mi       122Mi       1.0Mi       182Mi       149Mi
Swap:          2.0Gi          0B       2.0Gi
```

- 위와 같이 Swap 항목이 2.0GiB로 나타나나 설정이 완료되었음을 확인할 수 있다.

# 마무리

메모리 부족 문제에 대한 임시 방편으로 Swap 메모리를 설정하여 시스템의 안정성을 높이기 위한 시도를 해보았다. 실제로 스왑 메모리가 설정되면, 물리 메모리가 부족할 경우 디스크를 대신 활용하여 프로세스 강제 종료나 서버 다운 현상을 막을 수 있다.

다만 Swap은 디스크 기반이므로 속도가 느려 성능 저하가 발생할 수 있다. 따라서 이는 임시적인 해결책이며, 다음과 같은 근본적인 조치도 병행할 필요가 있을 것 같다.

- 서버 메모리 용량 증가
- 애플리케이션 메모리 최적화
- 병목 지점 파악
- 알림 시스템 구축 및 자동 재부팅 설정 등

지금은 돈이 없기 때문에 이렇게 문제를 해결할 예정이지만, 서비스가 지속되면 서버가 다운되는 원인에 대해 다양한 방면에서 분석하고 보다 안정적인 서비스 운영을 위한 준비를 지속할 필요가 있을 것 같다.
