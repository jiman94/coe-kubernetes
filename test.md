# StatefulSet

ReplicaSets은 단일 팟 템플릿에서 여러 복제본을 생성합니다.
이렇게 생성된 복제본들은 이름과 IP주소를 제외하고 동일합니다.

만약 여기서 PersistentVolumeClaim을 참조하는 볼륨이 있을 경우 복제된 모든 팟들은 동일한 PersistentVolume을 사용하게 됩니다.

이런 경우 복제된 팟들이 자체적인 별도의 VolumeClaim을 사용할 수 없습니다.

각 인스턴스가 별도의 스토리지를 필요로 할 경우 StatefulSet을 사용 하여야 합니다.

## 1. StatefulSet 없이 개별 저장소를 바라보는 팟의 복제본 만들기

복제된 여러 팟에 자체 스토리지 볼륨을 사용하려면 다음과 같은 방법이 있습니다.

* 수동으로 팟을 각각 만듭니다

  * 팟을 수동으로 생성하고 각각 원하는 Volume-Claim을 사용할 수 있습니다.

  * 다만 레플리카셋을 사용하지 않기 때문에 수동으로 관리를 해야 하고 노드가 사라지게 되면 사용자가 수동으로 실행을 해야 합니다.

* 팟 인스턴스 당 하나의 레플리카 셋 사용

  * 팟을 직접 생성하는 대신 필요한 만큼 레플리카셋을 만듭니다. (복제본 갯수를 1로 설정)

  * 위의 방법보다는 좋지만 그래도 불편함이 존재합니다. (필요할 때마다 계속해서 만들어 줘야 합니다.)

* 같은 볼륨에서 다른 디렉토리 사용

  * 모든 팟들이 같은 PersistentVolume을 바라보지만 각가의 팟이 바라보는 디렉토리를 다르게 합니다.

  * 인스턴스간 조정이 필요하며 (개발자간 협의) 병목 현상을 일으킬 수 있습니다.


## 2. 팟에 대한 안정적인 ID 제공

기존에 잘 실행 되던 팟을 ReplicaSet을 이용하여 다시 띄우게 되면 볼륨을 주지 않았을 경우 데이터는 잃게 되며

만약 제대로 데이터 바인딩을 하더라도 새로운 ID가 부여가 되기 때문에 네트워크 ID의 불일치로 해당 서비스를 바라보는 기능들은 문제가 생길 수 있습니다.

분산형 상태 저장 프로그램에서는 안정적인 네트워크 ID가 필수 입니다. 응용 프로그램에서는 관리자가 각 구성원의 모든 클러스터 구성원과 해당 IP주소 또는 호스트 이름을 알고 있어야 합니다. 하지만 쿠버네티스의 경우 팟이 재조정 될 때마다 새로운 IP주소를 얻기 때문에 문제가 생길 수 있습니다.

이 문제를 해결하기 위해서는 아래와 같은 방법이 있습니다.

* 서비스 사용

  * 쿠버네티스에서 제공하는 서비스를 사용할 경우 안정적인 네트워크 주소 또는 DNS를 제공하기 때문에 사용자들은 서비스를 통해 접근 할 수 있습니다.

## 3. StatefulSet
ReplicaSet을 사용하여 팟을 실행하는 대신 StatefulSet을 이용하면 안정적인 이름과 상태를 가질 수 있습니다.

### 3.1 ReplicaSet과 비교

일반적으로 ReplicaSet의 경우 소떼로 비교합니다. 이유는 특별한 주의를 기울이지 않기 때문입니다. 이렇게 하면 애완동물과 다르게 사육사가 좋지 않은 가축을 쉽게 대체할 수 있기 때문입니다.

즉 상태가 중요하지 않은 어플리케이션은 죽거나 다시 실행해도 사용자가 큰 신경을 쓰지 않지만 상태 기반의 어플리케이션은 죽거나 다시 실행 할 경우 사용자들이 바로 알아 차릴 수 있습니다.

따라서 상태가 중요한 어플리케이션은 이전 인스턴스와 동일한 상태와 ID를 가져야 합니다.

StatefulSet은 팟의 ID와 상태를 유지하도록 합니다. 그리고 ReplicaSet처럼 쉽게 늘리거나 줄일 수 있습니다.

다만 ReplicaSet과 다르게 StatefulSet에 의해 생성 된 팟은 완전히 똑같이 복제되지 않습니다. 이유는 각각 자신만의 볼륨 집합을 가질 수 있기 때문입니다.

그리고 팟 인스턴스의 ID가 무작위로 생성되는 대신 예측 가능한 안정적인 ID를 가지고 있습니다.


### 3.2 안정적인 네트워크 식별

StatefulSet으로 팟을 만들게 되면 0부터 순차적으로 증가되는 이름을 가지게 됩니다. 이렇게 생성된 팟의 이름과 호스트 이름은 안정적인 저장소를 연결하는데 사용하게 됩니다.

일반적인 팟과 달리 상태저장 팟은 호스트 이름으로 주소를 지정할 수 있어야 합니다. 일반적인 팟의 복제본의 경우 사용하고자 할 때 아무것이나 선택을 하지만 상태 저장 팟의 경우 사용자가 복제본중 하나를 선택하기를 원할 수 있습니다.
이유는 일반적인 팟과 달리 상태저장 팟은 볼륨 등의 이유로 서로 다르기 때문입니다.

따라서 StatefulSet에서는 각 팟에 실제 네트워크 ID를 제공하는데 사용되는 헤드리스 서비스를 생성해야 합니다.

StatefulSet에 관리되는 팟 인스턴스가 사라지면 StatefulSet은 새 인스턴스로 교체를 하게 되는데 이 때 생성되는 팟은 이전에 사용되던 이름을 사용하게 됩니다.

반드시 꼭 동일한 노드에서 실행되지는 않지만 팟의 이름은 동일하게 생성됩니다.

Scaling을 할 때도 어떤 팟이 삭제 될지 모르는 ReplicaSet과 달리 StatefulSet은 높은 번호를 가진 인스턴스를 먼저 삭제 하기 때문에 예측을 할 수 있습니다.

StatefulSet의 경우 scale-down이 일어 날 때 한번에 하나의 창을 축소합니다. 분산 데이터 저장소가 각 데이터 항목의 복사본을 두 곳에 저장하도록 되어 있는 경우 두 노드가 동시에 다운 될 경우 데이터 손실이 일어나기 때문입니다. 만약 순차적으로 scale-down이 일어나게 되면 다른 곳에서 복제본을 생성할 시간이 있습니다.

따라서 StatefulSet은 팟의 상태가 비정상인 경우 scale-down을 허용하지 않습니다.

## 3.3 개별적인 저장소 제공

ReplicaSet과 다르게 StatefulSet은 개별적인 저장소를 지정 할 수 있습니다.

StatefulSet을 확장하면 두개 이상의 객체가 생성됩니다.
하나는 팟이고 그 이상의 객체는 PersistentVolumeClaim 입니다.

하지만 축소를 하게 될 경우 팟은 삭제가 되지만 Claim은 유지합니다. 데이터가 중요하기 때문입니다. 따라서 축소를 할 경우 PersistentVolumeClaim을 수동으로 삭제하고 PersistentVolume을 풀어줘야 합니다.

만약 축소 이후에 (PersistentVolumeClaim을 삭제하지 않았다고 가정) 다시 확장을 할 경우 새로운 팟은 동일한 PVC를 바라보게 됩니다.