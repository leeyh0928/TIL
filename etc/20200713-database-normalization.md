# Overview
데이터베이스 정규화란 데이터베이스의 설계를 재구성하는 테크닉이다. 정규화를 통해 불필요한 데이터(redundancy)를 없앨 수 있고,
삽입/갱신/삭제 시 발생할 수 있는 이상현상(anamolies)들을 방지할 수 있다.

***데이터배이스 정규화의 목적은 주로 두 가지이다.***
1. 불필요한 데이터(data redundancy)를 제거한다.
2. 데이터 저장을 `논리적으로` 한다. -> 테이블의 데이터 구성이 논리적이고 직관적이어야 한다는 것

# 1차 정규화
1차 정규형은 각 로우마다 컬럼의 값이 1개씩만 있어야 한다. 이를 컬럼이 원자값(Atomic Value)를 갖는다고 한다. 예를 들어, 아래와 같은 경우 
Adam의 Subject가 Biology와 Maths 두 개 이기 때문에 1차 정규형을 만족하지 못한다.
| Student   | Age   | Subject   |
|:----------|:------|:----------|
| Adam      | 15    | Biology, Maths    |
| Alex      | 14    | Maths     |
| Stuart    | 17    | Maths     |

위의 정보를 표현하고 싶은 경우 아래와 같이 한 개의 로우를 더 만든다. 결과적으로 1차 정규화를 함으로써 `데이터 redundancy`는 더 증가하였다.
데이터의 논리적 구성을 위해 이 부분을 희생하는 것으로 볼 수 있다.
| Student   | Age   | Subject   |
|:----------|:------|:----------|
| Adam      | 15    | Biology   |
| Adam      | 15    | Maths     |
| Alex      | 14    | Maths     |
| Stuart    | 17    | Maths     |

# 2차 정규화
2차 정규화부터 본격적인 정규화의 시작이라고 볼 수 있다. `2차 정규형`은 테이블이 모든 컬럼이 `완전 함수적 종속`을 만족하는 것이다.
즉, 기본키 중에 특정 컬럼에만 종속된 컬럼(부분적 종속)이 없어야 한다는 것이다. 위 테이블의 경우 기본키는 (Student, Subject) 두 개로
볼 수 있다. 이 두 개가 합쳐져야 한 로우를 구분할 수 있다. 그런데 Age의 경우 기본키 중에 Student에만 종속되어 있다. Student 컬럼의 값만
알면 Age의 값을 알 수 있다. 따라서 ***Age가 두 번 들어가는 것은 불필요한 것***으로 볼 수 있다.

***Student Table***
| Student   | Age   |
|:----------|:------|
| Adam      | 15    |
| Alex      | 14    |
| Stuart    | 17    |

***Subject Table***
| Student  | Subject   |
|:---------|:----------|
| Adam     | Biology   |
| Adam     | Maths     |
| Alex     | Maths     |
| Stuart   | Maths     |

이를 해결하는 방법으로 위 처럼 테이블을 쪼개는 것이다. 그러면 두 테이블 모두 2차 정규형을 만족하게 된다. 그리고 삽입/갱신/삭제 이상을 
겪지 않게된다. 하지만 조금 더 복잡한 테이블의 경우, 갱신 이상을 겪기도 하는데 이를 해결하는 것이 바로 3차 정규화이다.

# 3차 정규화
***Student_Detail Table***

| Student_id  | Student_name   | DOB  | Street   | City  | State   | Zip  |

이와 같은 데이터 구성을 생각해보자. Student_id가 기본키이고, 기본키가 하나이므로 2차 정규형은 만족하는 것으로 볼 수 있다. 하지만
이 데이터의 Zip 컬럼을 알면 Street, City, State를 결정할 수 있다. 또한 여러명의 학생들이 같은 Zip 코드를 갖는 경우에 Zip 코드만 알면
Street, City, State가 결정되기 때문에 이 컬럼들에는 중복된 데이터가 생길 가능성이 있다.

정리하면 3차 정규형은 기본키를 제외한 속성들 간의 ***이행적 함수 종속이 없는 것***이다. 다시 말하면 기본키 이외의 다른 컬럼이 그 외
다른 컬럼을 결정 할 수 없는 것이다.

3차 정규화는 2차정규화와 마찬가지로 테이블을 분리함으로써 해결할 수 있는데, 이렇게 두 개의 테이블로 나눔으로써 3차 정규형을 
만족할 수 있다. 이를 통해 데이터가 논리적인 단위(학생, 주소)로 분리될 수 있고, 데이터의 redundancy도 줄었음을 알 수 있다.

***Student_Detail Table***

| Student_id  | Student_name   | DOB  | Zip  |

***Address Table***

| Zip  | Street   | City  | State   |

# BCNF
BCNF는 (Boyce and Codd Normal Form) 3차 정규형을 조금 더 강화한 버전으로 볼 수 있다. 이는 3차 정규형으로 해결할 수 없는 
이상현상을 해결할 수 있다. BCNF란 3차정규형을 만족하면서 ***모든 결정자가 후보키 집합에 속한 정규형***이다.
아래와 같은 경우를 생각해보면, 후보키는 수퍼키 중에서 최소성을 만족하는 건데, 이 경우 (학생, 과목)이다.
(학생, 과목)은 로우를 유일하게 구분할 수 있다. 그런데 이 테이블의 경우 교수가 ***결정자***이다.
(교수가 한 과목만 강의할 수 있다고 가정) 즉, 교수가 정해지면 과목이 결정된다. 그런데 교수는 후보키가 아니다. 따라서 이 경우에
BCNF를 만족하지 못한다고 한다.

3차 정규형을 만족하면서 BCNF는 만족하지 않는 경우는 ***일반 컬럼이 후보키를 결정***하는 경우이다.
| 학생   | 과목    | 교수     | 학점   |
|:-------|:-------|:----------|:------|
| 1      | ABC123  | 김인영   | A     |
| 2      | CS123   | Mr.Sim   | A     |
| 3      | CS123   | Mr.Sim   | A     |

위와 같이 테이블이 구성된 경우 데이터가 중복되고, 갱신 이상이 발생한다. 예를 들어 Mr.Sim이 강의하는 과목명이 바뀌었다면 
두 개의 로우를 갱신해야 한다. 이를 해결하기 위해서는 마찬가지로 테이블을 분리한다.

***교수 테이블***
| 교수     | 과목      |
|:---------|:---------|
| 김인영   | ABC123   |
| Mr.Sim   | CS123    |
| Mr.Sim   | CS123    |

***수강 테이블***
| 학생   | 과목    | 학점   |
|:-------|:-------|:------|
| 1      | ABC123  | A     |
| 2      | CS123   | A     |
| 3      | CS123   | A     |

# References
* https://3months.tistory.com/193#recentEntries