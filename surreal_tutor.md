# 시작하기
- [여기](https://surrealdb.com/docs/surrealdb/installation)에서 설치요망
- ```surreal version```으로 버전을 확인할 수 있다. (잘 설치가 되었는지 확인도)

# 실행
- 다음을 이용해서 실행할 수 있다.
```
	surreal start memory -A --auth --user root --pass root
```
- 각 속성은 다음과 같은 기능을 가진다
- ```-A``` 모든 기능 활성화
- ```--auth``` 데이터베이스에 대한 인증을 활성화
- ```--user root --pass root``` 데이터베이스에 액세스하기 위한 사용자 이름과 비밀번호 지정 (예시는 둘 다 root임)
- ```memory``` 데이터 베이스가 메모리에서 실행된다는 의미 즉, 서버가 꺼지면 데이터도 잃게됨!

# SurrealDB에서 쿼리
- 서버를 실행하면 나오는 메세지를 통해 특정 엔드포인트에 따른 SurrealDB를 실행
- ```surreal sql --endpoint http://localhost:8000 --username root --password root --namespace test --database test```
## namespace
- 시스템 구조상 SurrealDB 바로 밑에있는 레이어로, 하위에 databases가 있다.
## database
- namespace밑에 있는 database로 이 안에 데이터를 저장하는 tables가 있다.
- 여담으로 table밑에는 fields가 있는데, 테이블의 레코드의 각 속성(행)의 타입등을 나타내는 것 같다.

## 쿼리
### CREATE
- database에 레코드를 추가하는 데 사용된다(라고 하는데 아마 테이블도 동시에 만들어지는 듯?)
```
// 이런식으로 만들 수 있다.
// 이렇게 하면 해당 레코드에 무작위 id가 생성된다.
	CREATE account SET
		name = 'ACME Inc',
		created_at = time::now();

// 무작위 id가 생성되는 것을 원치 않는다면, 테이블명:id 를 하면 된다.
// 아래는 john이라는 id를 가진 author테이블의 레코드가 생성된다.
	CREATE author:john SET
		name.first = 'John',
		name.last = 'Adams',
		name.full = string::join(' ',name.first, name.last),
		age = 29,
		admin = true,
		signup_at = time::now();

// 레코드끼리 연결도 가능하다.
// 다음은 article 테이블의 레코드가, author:john레코드를 연결하며, 아예 꼭 id를 알고 있어야 하는 것이 아닌 아예 쿼리문으로 레코드를 검색해서 넣을 수도 있다. (author과 account를 유심히 보자)
	CREATE article SET
		create_at = time::now(),
		author = author:john,
		title = 'Lorem ipsum dolor',
		text = "blablablablablablalb",
		account = (SELECT VALUE id FROM account WHERE name = 'ACME Inc' LIMIT 1)[0];
```

### SELECT
- 기존 SQL과 유사하게 작동하지만 NoSQL쿼리의 이점을 가져왔다고함.
```
// 기존 사용방법
	SELECT * FROM article;

// 다른 테이블이나 레코드에서 한번에 데이터를 검색할 수 있다.
	SELECT * FROM article, account;

// 또한 여러 테이블에서 데이터를 가져와 데이터를 병합하는 대신 JOIN을 사용하지 않고도, 관련 레코드를 효율적으로 탐색할 수 있다.
// 위에서 article에서 author과 account를 가지고 있다는 것을 숙지하자
	SELECT * FROM article WHERE author.age < 30 FETCH author, account;
// 아마 FETCH author, account가 article안의 기반 레코드의 테이블명을 뜻하는 것 같다. (순서는 상관 없는듯?)
```

### UPDATE
- 특정 id를 업데이트 하거나 WHERE문을 이용해서 레코드를 업데이트 할 수 있다.
```
// 특정 id를 업데이트
	UPDATE author:john SET name.first = 'David', name.full = string::join(' ',name.first, name.last);
	// 여담으로 string::join()은 두번째, 세번째... 등의 문자사이에 넣을 문자를 첫번째 인수에 넣는것

// WHERE절 뒤의 조건에 맞는 레코드 업데이트
	UPDATE author SET admin = false WHERE name.last = 'Adams';
```

### DELETE, REMOVE
- 각각 레코드와 테이블전체를 제거하는 쿼리
```
	DELETE article WHERE author.name.first = 'David';

	REMOVE TABLE author;
	REMOVE TABLE article;
	REMOVE TABLE account;
```


# 개념(장점?)
- 기존 백엔드 (Golang, Python, Rust, C등등...)를 통해서 기존 플랫폼(react, next.js, ember.js등)도 활용할 수 있다.
------
- 일반적인 SQL이 아닌 유사한 쿼리언어 SurrealQL을 이용해서 데이터베이스 전체에서 데이터를 쉽게 관리할 수 있게 되었다.
------
- 스토리지와 컴퓨팅계층이 분리되어있기 때문에, 임베디드 모드, 수직확장 가능한 단일노드 데이터베이스 서버나 수평 확장 가능한 다중노드 분산 클러스터로 실행될 수 있음
- 데이터베이스 서버로서 SurrealDB는 현재 RocksDB, TiKV나 FoundationDB를 사용하여 데이터를 저장할 수 있다.
------
- 다른 전통적 관계형 데이터베이스 및 문서 데이터베이스와 유사하게 작동하지만, 약간의 차이가 있음
- namespace -> database -> table -> record -> field 레이어 구조를 가진다.

# 구조(레이어)
## 쿼리
- 컴퓨팅 계층이라고도 하는 쿼리계층은 클라이언트의 쿼리를 처리하여 레코드를 선택, 생성, 업데이트등을 구문분석하는 구간이다.
- 아무튼 데이터를 언제 가져오고, 저장 여부를 결정하며, 명령문을 반복자로 수행하고 어쩌구... 대충 데이터관리자가 있는 구간

## 스토리지
- 쿼리에서 내려준 데이터 저장을 처리한다. 순서가 지정된 목록에 데이터를 저장하고, 디스크의 스토리지를 처리하거나 클러스터 전체에 걸쳐서 데이터를 분산저장(샤딩)한다. 다양한 스토리지 엔진의 일부는 동시판독기를 지원하고, 일부는 더해서 기록기까지 지원한다.

### RocksDB
- 임베디드에서 사용하며 디스크에 데이터를 저장하는 고성능 데이터베이스. 원래는 많은 CPU코어를 활용하고 솔리드 스테이트 드라이브같은 뭔가 고급진 기술을 사용하도록 최적화된 Google의 LevelDB포크다. 병합트리 구조 기반

### TiKV
- 분산 모드에서 사용하는 스토리지로, 확장성이 뛰어나고, 대기시간이 짧으며 사용이 쉽다.
- Google의 분산 시스템과 Raft합의 알고리즘과 같은 학계의 최신 성과에서 영감을 받았다.

### IndexedDB
- 브라우저내에 데이터를 저장하고 유지하도록 구성하는 DB
- 키와 값을 Uint8Array로 직렬화하여 저장하며 이진 키-값 저장소로 활용하여 우수한 성능을 제공함

##### MongoDB나 다른 DB의 문법과 차이점을 직접 비교하는 [링크](https://surrealdb.com/docs/surrealdb/introduction/mongo)
##### SQL과 SurrealQL의 차이를 직접 비교하는 [링크](https://surrealdb.com/docs/surrealdb/introduction/sql)
