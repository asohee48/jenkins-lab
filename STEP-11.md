# 11단계: 실패 진단과 복구

이 단계에서는 존재하지 않는 컨테이너 이미지 태그를 사용해 의도적으로 배포를 실패시킨다.

실습용 `jenkins-lab` 네임스페이스에서 진행한다. 먼저 관리자 권한이 있는 터미널에서 이벤트 조회 권한을 반영한다.

```bash
kubectl apply -f jenkins-lab-role.yaml
```

## 1. 실패 재현

1. 변경된 `Jenkinsfile`을 저장소에 반영하고 Jenkins에서 파이프라인을 한 번 실행한다. 처음 실행한 뒤부터 **Build with Parameters** 메뉴에서 파라미터를 선택할 수 있다.
2. `SIMULATE_FAILURE`를 선택하고 빌드한다.
3. Stage View에서 `Validate`와 `Helm Template`은 성공하고 `Deploy`가 실패하는지 확인한다. `helm upgrade`가 `--wait --timeout 3m`으로 실행되므로 실패 판정까지 최대 3분 정도 걸린다.
4. `Deploy`가 실패했으므로 이후 `Verify`는 실행되지 않고 `skipped`로 표시된다.

## 2. Console Output 확인

- `Error: UPGRADE FAILED: context deadline exceeded`: Helm이 제한 시간 안에 배포 완료를 확인하지 못했다.
- `kubectl get deployment,pod` 결과의 `ImagePullBackOff` 또는 `ErrImagePull`: 컨테이너 이미지를 가져오지 못했다.
- `kubectl describe pod`의 `Image` 값이 `nginx:step-11-image-does-not-exist`인지 확인한다.
- Pod의 `Events` 또는 마지막 `Recent namespace events`에서 `Failed to pull image`, `not found` 같은 메시지를 확인한다.
- `helm status`에서 릴리스 상태와 실패한 리비전을 확인한다.

맨 위의 Helm timeout은 결과일 뿐이다. 실제 원인은 뒤이어 출력되는 Pod 상태와 이벤트의 이미지 pull 실패 메시지다.

오류 문구는 Helm 버전에 따라 다르다. 최초 설치에서는 `INSTALLATION FAILED`, 대기 실패에서는 `timed out waiting for the condition` 등으로 표시될 수도 있다. 이미지 pull 실패로 컨테이너가 시작하지 못했다면 애플리케이션 로그보다 Pod Events를 확인한다. 실패 이미지 override는 Deploy에서만 적용되므로 앞의 lint/template 성공이 이미지 존재 여부를 보장하지 않는다.

## 3. 복구

1. Jenkins에서 **Build with Parameters**를 다시 선택한다.
2. `SIMULATE_FAILURE` 선택을 해제하고 빌드한다.
3. `--reset-values`가 이전 실패 빌드의 이미지 override를 초기화하고 `values.yaml`의 정상 이미지 `nginx:1.27.5-alpine`을 적용한다. 다른 수동 override도 차트 기본값으로 초기화된다.
4. `Deploy`와 `Verify`가 모두 성공하는지 확인한다.
5. Console Output에서 `deployment "lab-nginx" successfully rolled out`을 확인한다.

클러스터에서도 직접 확인할 수 있다.

```bash
helm status lab-nginx --namespace jenkins-lab
kubectl get deployment,pod,service \
  --namespace jenkins-lab \
  --selector app.kubernetes.io/instance=lab-nginx
```

복구 빌드에서도 실패한다면 `kubectl describe pod`의 최신 Events를 다시 확인한다. 이전에 실패한 Pod는 새 ReplicaSet으로 교체되는 동안 잠시 보일 수 있으므로 `AGE`와 Pod 이름을 함께 확인한다.
