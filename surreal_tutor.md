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

## Docker에서 실행하기
- surrealDB는 Docker에 image로 등록되어 있다.
- 다음 코드로 image를 받고 컨테이너로 실행할 수 있다.
```sudo docker run --rm --pull always -p 8000:8000 surrealdb/surrealdb:latest start```
- 물론 -v태그로 파일에 db를 유지할 수도 있다. (권한 거부현상이 잦기 때문에 경로를 잘 확인해주자)
```sudo docker run --rm --pull always -p 8000:8000 -v /경로 surrealdb/surrealdb:latest start file:/경로/db이름.db```
- 당연한 이야기지만 start옆에 이것저것 속성부여도 가능하다 (아래는 --log와 --user, --pass설정)
```sudo docker run --rm --pull always -p 80:8000 -v /경로 surrealdb/surrealdb:latest start --log trace --user root --pass root file:/경로/db이름.db```
- 권한 거부현상이 발생하면 -u 를 이용해서 사용자를 root로 변경해보자
```sudo docker run --rm -u root --pull always -p 80:8000 -v /경로 surrealdb/surrealdb:latest start --log trace --user root --pass root file:/경로/db이름.db```
- (docker명령어는 따로 검색하면 알겠지만 --rm은 컨테이너가 stop될 때 컨테이너를 제거해주는 명령어이다... 원치 않으면 지워도 됨)
- 아래는 도움말 출력
```docker run --rm --pull always surrealdb/surrealdb:latest help```


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


# Crate
## 데이터
- 우선 활용할 데이터들은 struct로 구성되어있어야 하며, Serialize, Deserialize 트레잇을 가지고 있어야 한다.
- 여기서 주의할 점은 struct는 튜플구조체가 아니어야 한다.
```
#[derive(Serialize, Deserialize)]
struct TableName{
	feild_name: String
}
```

## 서버 연결
- 서버 연결은 Websocket이나 다른 (HTTP등)으로 할 수 있다.
```rust
	let db = Surreal:new::<Any>("ws://localhost:80").await?;
	// 혹은
	let db = Surreal::new::<Ws>("localhost:80").await?;
	let db = Surreal::new::<Http>("localhost:80").await?;
```
- 또한 once_cell crate를 사용해서 static으로 지정해서 최초 한번만 수행할 수도 있다.
```rust
	use once_cell::sync::Lazy;

	static DB: Lazy<Surreal<Any>> = Lazy::new(Surreal::init);
	DB.connect("ws://localhost:80").await?;
	// 혹은
	DB.connect("http://localhost:80").await?;
```
- username과 password가 있는경우는 이어서 다음과 같이 쓰면 된다.
```rust
	db.signin(Root{
		username: "root",
		password: "root",
	}).await?;
```
- 연결 설정이 완료되면, namespace와 database이름으로 접속이 가능해진다.
```rust
	db.use_ns("namespace").use_db("database").await?;
```

- 또한 특정 범위에 대한 유저를 생성하고 싶다면 다음과 같이 하면 된다.
- (정확한건 더 알아봐야 할 것 같다.)
```rust
	db.signup(Scope {
		namespace: "namespace",
		database: "database",
		scope: "user_scope",
		params: Ptype{
			// email 과 같은 추가적인 정보가 필요한 경우 구조체를 만들어서 넣으면 된다.
			// Ptype은 그 만들 구조체의 임시적인 이름이다(아무 이름이던 상관 없음)
			// 단 Debug와 Serialize 트레잇이 있어야 하는 것 같다.
		}
	}).await?;
```
## CREATE
- surrealQL 이 아니라 마치 JAP를 활용하는 것과 비슷하게, 함수로 수행할 수 있다.
- 리턴값은 surrealQL을 작성해서 얻는 값과 대응하는 값이 들어온다. (Vec<type>과 Option<type>)
```rust
	let _: Vec<type> = db.create("table_name").content(type{field_name: value}).await?;
	// return [type {field_name: value}]
	// 아마 랜덤 uuid가 id필드에 들어갔을 것이다.

	let _: Option<type> = db.create(("table_name","cutom_id")).content(type{field_name: value}).await?;
	// return Result(type{field_name: value})
	// "custom_id"가 id필드에 들어갔을 것이다.
```

## SELECT
- SELECT는 다음과 같이 활용하면 된다.
- id에 대한 범위 지정을 제외하면 where문은 딱히 없는 것 같다.(Where를 표현하는 Enum이 있으나, 아직 활용하진 못하나봄)
```rust
	let _: Vec<type> = db.select("table_name").await?; // SELECT * FROM table_name

	// 특정 id를 찾으려면 다음과 같이 하면됨

	let _: Option<type> = db.select(("table_name", "custom_id")).await?; // SELECT * FROM table_name:custom_id;

	// id의 범위를 찾으려면 다음과 같이 하면 된다. (아마 오름차순으로 정렬되어 있을 것이다.)
	let _: Vec<type> = db.select("table_name").range("jane".."john").await?;
```

- 또한 어떤 레코드 집단에 update가 수행되는지 관찰하려면, 다음과 같이 하면 된다.
```rust
	let mut stream = db.select("table_name").live().await?;
	while let Some(notification) = stream.next().await{
		// 알림을 활용하면 된다.
	}
```

## UPDATE
- table전체 레코드 혹은 특정 레코드만 업데이트 할 수 있다.
- field_name은 레코드의 필드의 속성들을 /단위로 한다... 즉 배열이라고 친다면, /field_name/0 이면 0번째 인덱스의 처리를 하는것이다.
```rust
	let _: Vec<type> = db.update("table_name")
		.patch(PatchOp::replace("/field_name", new_data)).await?;
	let _: Option<type> = db.update(("table_name", "custom_id"))
		.content(type{
			// 변경점
		}).await?;
```

- PatchOp도 여러가지가 있다.
1. ```PatchOp::replace("/field_name", new_data)```
2. ```PatchOp::add("/field_name", add_data)```
3. ```PatchOp::remove("/field_name")```

## DELETE
- 테이블 전체 레코드를 제거하거나, 특정 레코드를 제거
```rust
	let _: Vec<type> = db.delete("table_name").await?;
	let _: Option<type> = db.delete(("table_name", "custom_id")).await?;
```

## SurrealQL
- 쿼리문을 직접 넣을 수도 있다.
```rust
	let mut result = db
		.query("CREATE table_name SET value = $data")
		.query("SELECT * FROM type::table($table)")
		.bind(("data","some_data"))
		.bind(("table","table_name"))
		.await?;

	// 결과를 가져오는건 take(index)를 하면 된다.
	let created_result: Vec<table_name> = result.take(0)?;
	let selected_result: Vec<table_name> = result.take(1)?;

	// 물론 결과 예상 (Vec<table_name>)타입이 다르다면, 에러를 내보낸다.
	if let Err(err) = result.take::<Vec<rong_type>>(0) {
		println! ("잘못된 타입 {e:#?}");
	}
```

## 사용시 주의사항
- 구조체는 튜플구조체가 아니어야하며, 반드시 feild구조체여야 한다.
- Db에서 주는 결과값은 언제나 구조체와 대응되어야 한다. (필드명이 db의 테이블과 같은지 확인해야함)
- id를 받고 싶다면 구조체 안에 id라는 필드명의 Thing타입(구조체)를 가져야한다.
- id의 형식은 자동으로 서버에서 받게된다. id의 타입이 Number인데, String형식으로 받으려고 하면 문제가 발생한다. (쿼리에서 id를 String 하고싶다면 문자열로 보이게끔 ID1010 이런식으로 이름을 지어줘야한다.) 
