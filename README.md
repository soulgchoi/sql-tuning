# 🚀 조회 성능 개선하기

> 미션 요구사항은 A, B만 해당되어서 들어가기에 앞서는 안보셔도 되구 보셔도 되구 자유입니당 !!

## 목차
* [들어가기에 앞서](#들어가기에-앞서)
    * [1번](#실습-1번)
    * [2번](#실습-2번)
    * [3번](#실습-3번)
* [ERD of A](#erd-of-a)
* [A. 쿼리 연습](#a-쿼리-연습)
  * [A1 - 쿼리 작성만으로 1s 이하로 반환한다.](#a1---쿼리-작성만으로-1s-이하로-반환한다)
  * [A2 - 인덱스 설정을 추가하여 50 ms 이하로 반환한다.](#a2---인덱스-설정을-추가하여-50-ms-이하로-반환한다)
* [ERD of B](#erd-of-b)
* [B. 인덱스 설계](#b-인덱스-설계)
    * [B1 - Coding as a Hobby 와 같은 결과를 반환하세요.](#b1---coding-as-a-hobby-와-같은-결과를-반환하세요)
    * [B2 - 프로그래머별로 해당하는 병원 이름을 반환하세요.](#b2---프로그래머별로-해당하는-병원-이름을-반환하세요)
    * [B3 - 프로그래밍이 취미인 학생 혹은 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요.](#b3---프로그래밍이-취미인-학생-혹은-주니어0-2년들이-다닌-병원-이름을-반환하고-userid-기준으로-정렬하세요)
    * [B4 - 서울대병원에 다닌 20대 India 환자들을 병원에 머문 기간별로 집계하세요.](#b4---서울대병원에-다닌-20대-india-환자들을-병원에-머문-기간별로-집계하세요)
    * [B5 - 서울대병원에 다닌 30대 환자들을 운동 횟수별로 집계하세요.](#b5---서울대병원에-다닌-30대-환자들을-운동-횟수별로-집계하세요)

## 들어가기에 앞서
[실습사이트](https://www.w3schools.com/sql/trymysql.asp?filename=trysql_func_mysql_concat)에서 아래 문제를 해결해보자.

실습 관련 수행 과정은 [실습 과정 정리](./실습-과정-정리.md)에서 확인하실 수 있어요!

### 실습 1번
200개 이상 팔린 상품명과 그 수량을 수량 기준 내림차순으로 보여주세요.

```sql
select p.ProductID as '상품아이디', p.ProductName as '상품이름', sum(od.Quantity) as '총수량'
from Products p, OrderDetails od
where p.ProductID = od.ProductID
group by p.ProductID
having `총수량` >= 200
order by `총수량` desc;
```

### 실습 2번
많이 주문한 순으로 고객 리스트(ID, 고객명)를 구해주세요. (고객별 구매한 물품 총 갯수)

```sql
select c.CustomerID as '고객아이디', c.CustomerName as '고객이름', IFNULL(q.totalQuantity, 0) as '주문량'
from Customers c left join (
  select o.CustomerID, sum(od.Quantity) as totalQuantity
  from Orders o
         inner join OrderDetails od on o.OrderID = od.OrderID
  group by o.CustomerID) as q
                           on c.CustomerID = q.CustomerID
group by c.CustomerID
order by `주문량` desc;
```

### 실습 3번
많은 돈을 지출한 순으로 고객 리스트를 구해주세요.

```sql
select c.CustomerID as '고객아이디', c.CustomerName as '고객이름', IFNULL(qp.Amount, 0) as '지출금액'
from Customers c left join
     (
       select o.CustomerID, sum(od.Quantity*p.Price) as Amount
       from OrderDetails od
              inner join Products p on od.ProductID = p.ProductID
              inner join Orders o on od.OrderID = o.OrderID
       group by o.CustomerID
     ) as qp on c.CustomerID = qp.CustomerID
group by c.CustomerID
order by `지출금액` desc;

```



---


## ERD of A
  <img src="/images/erd-of-a.png" width="800"/>

* [reverse engineering을 이용해 mysqlworkbench erd 생성하기](https://dololak.tistory.com/457)

## A. 쿼리 연습
활동중인(Active) 부서의 현재 부서관리자 중 연봉 상위 5위안에 드는 사람들이 최근에 각 지역별로 언제 퇴실했는지 조회해보세요.  
(사원번호, 이름, 연봉, 직급명, 지역, 입출입구분, 입출입시간)  
급여 테이블의 사용여부 필드는 사용하지 않습니다. 현재 근무중인지 여부는 종료일자 필드로 판단해주세요.

### A1 - 쿼리 작성만으로 1s 이하로 반환한다.

```sql
select 연봉상위5위.사원번호, 연봉상위5위.이름, 연봉상위5위.연봉, 연봉상위5위.직급명, 사원출입기록.입출입시간, 사원출입기록.지역, 사원출입기록.입출입구분
from 사원출입기록,
     (select 부서관리자.사원번호 as '사원번호', 사원.이름 as '이름', 급여.연봉 as '연봉', 직급.직급명 as '직급명'
      from 부서관리자
             inner join 부서 on 부서관리자.부서번호 = 부서.부서번호
             inner join 사원 on 부서관리자.사원번호 = 사원.사원번호
             inner join 급여 on 부서관리자.사원번호 = 급여.사원번호
             inner join 직급 on 부서관리자.사원번호 = 직급.사원번호
      where 부서.비고 = 'active'
        and 부서관리자.종료일자 >= curdate()
        and 급여.종료일자 >= curdate()
        and 직급.종료일자 >= curdate()
      order by `연봉` desc
        limit 5) as 연봉상위5위
where 입출입구분 = 'O'
  and 사원출입기록.사원번호 = 연봉상위5위.사원번호
order by 연봉 desc
;
```

* 실행 결과  
  <img src="/images/a1.png" width="900"/>

#### 쿼리에 대한 설명  
우선, 조회할 컬럼들을 먼저 살펴보았어요. 사원번호, 이름, 연봉, 직급명은 연봉 TOP 5 테이블을 만들며 뽑아낼 수 있는 값이라는 생각이 들었어요.
그래서 크게 연봉상위5위 테이블과, 사원출입기록 두 테이블을 가장 겉 쿼리에서 이용하기로 생각했어요.
내부 연봉상위5위 테이블을 만들기 위해 from 절에 서브쿼리를 사용했어요. 서브쿼린 내부에서는 모든 테이블과 연관관계를 가진 부서관리자 테이블에 각 테이블들을
inner join 하도록 구성했어요. 또한 종료일자를 오늘 날짜 이후로 설정함으로써 근무중임을 확인했다. 이때, `curdate()`와 `curtime()`, `now()`등을 사용할 수 있었는데,
각각의 차이는 다음과 같아요. 

|CURDATE|CURTIME|NOW|
|--------|------------|--------|
|2021-10-15|22:37:25|2021-10-15 22:37:25|

어차피 값에 시간은 포함되어있지 않기에 `curdate()`를 사용했어요. 이후 조건은 문제에 나온대로 설정했어요.




---


### A2 - 인덱스 설정을 추가하여 50 ms 이하로 반환한다.
```sql
CREATE INDEX I_사원번호 ON tuning.사원출입기록 (사원번호);
```

* 실행 결과  
  <img src="/images/a2.png" width="900"/>

* EC2에 DB를 올려놓고 연결해서 사용하다보니 네트워크에 따라서 Duration이 일정하지 않더라구요.. 😞  

#### 인덱스 설정에 대한 이유  
사원출입기록에 대한 Full Table Scan을 최적화하기위해 사원출입기록 테이블의 인덱스 정보를 확인했어요. (순번, 사원번호) 순으로는 인덱스가 걸려있었지만, 사원번호 단일로는 생성되어있지 않았어요. 조건절에서 사원번호를 사용하기에 사원번호 단일 인덱스를 생성해주었어요.

[질문]  
1. 입출입구분의 equal 연산자로 한번 거르고, 사원번호도 equal로 거르니 (입출입구분, 사원번호)와 같은 형태로 인덱스를 거는게 더 좋지 않을까? 라는 생각이
들었어요. 근데 별 차이가 없더라구요? 이유가 뭘까요?

[답변 정리]
* (입출입구분, 사원번호)로 복합인덱스를 걸게 되면, 동등 비교를 const로 수행한다.  
* 따라서 쿼리 처리를 위한 레코드 수가 사원번호만을 인덱스로 걸었을 때보다 적어진다.  
* 또한 이미 const로 걸러진 값에서 또 const로 거르기 때문에 filtering에서도 이점이 존재한다.  
-> 복합인덱스와 사원번호 단일인덱스간에는 차이가 있지만, 결과 row 수 자체가 큰 차이가 없기 때문에 현 데이터 상황에서는 별 차이가 없게 느껴질 수 있다.  

---


## ERD of B
  <img src="/images/erd-of-b.png" width="800"/>

## B. 인덱스 설계
* 조건  
  주어진 데이터셋을 활용하여 아래 조회 결과를 100ms 이하로 반환

### B1 - Coding as a Hobby 와 같은 결과를 반환하세요.

* programmer.id에 PK 지정
* hobby에 index 지정
* filesort와 임시 테이블을 없애기 위해 `order by percentage desc` -> `order by null`로 변경해보았는데, 그렇게 많은 차이가 나지 않아 
  우선 Yes가 위에 나오도록 `order by percentage desc`를 유지했습니다.

```sql
CREATE INDEX I_hobby ON subway.programmer (hobby);

select hobby as 'coding as a hobby', round((count(id) * 100 / (select count(*) from subway.programmer)), 1) as 'percentage'
from subway.programmer
group by hobby
order by percentage desc;
```

* 실행 결과  

  <img src="/images/b1.png" width="900"/>

* Duration: 0.053sec  

#### 쿼리 및 인덱스에 대한 이유  
우선, 조건은 비교적 쉬운 편이기에 쿼리에 대한 이유는 패쓰할게요. 아마 대부분 동일할거라 생각해요. 개선은 먼저 programmer.id에 PK를 지정해보았어요. 테이블의 프라이머리 키는 클러스터링의 기준이 되어요. 특히 InnoDB를 사용하기에 클러스터링 인덱스로 저장되는 테이블은 프라이머리 키 기반 검색이 빨라요. 그래서 서브쿼리에서 id를 활용하고 있기도 하고.. 등의 이유로 pk를 지정해줬어요. 추가로 hobby에 인덱스를 지정해보았어요. hobby에 추가 인덱스를 지정해준 이유는 group by 절에 인덱스를 적용해 인덱스 스캔이 일어나길 의도한 거였어요. 하지만 count(*)를 사용하고 있어서 그런지, 결국엔 full index scan이 일어나서 그렇게 효과는 없어보이네요. 🥲

* 에어 리뷰 추가  
hobby에 인덱스를 걸기 전 -> Full Table Scan, Duration 0.268sec    
hobby에 인덱스를 걸고 난 후 -> Index Full Scan, Duration 0.048sec  
hobby에 인덱스를 걸게 되면, hobby 기준으로 정렬된 데이터가 추출되고 추출된 row 수만 사용하기에 별도의 추가 테이블 액세스 필요성이 없어져 더 효율적으로 작동하는 듯 하다.  

#### Index Full Scan은 나쁜것인가?
Index Full Scan은 인덱스를 사용하지만 인덱스를 처음부터 끝까지 엑세스해서 일반적으로 성능저하를 일으킬 수 있다고 하는데요. 
테이블의 대부분 데이터를 추출하는 경우는 테이블보다 작은 공간을 차지하는 인덱스를 엑세스 하여 I/O 양을 감소, 인덱스의 첫 번째 칼럼으로 정렬된 데이터를 자동 추출 등의 장점도 있다고 해요.
  

---


### B2 - 프로그래머별로 해당하는 병원 이름을 반환하세요.
* (covid.id, hospital.name)  

```sql
select c.programmer_id as programmer, h.name as hospital_name
from subway.hospital h
       inner join subway.covid c on h.id = c.hospital_id
       inner join subway.programmer p on c.programmer_id = p.id;
```
* 실행 결과

  <img src="/images/b2.png" width="900"/>
  

* Duration: 0.023sec   

#### 쿼리 및 인덱스에 대한 이유
이번에는 인덱스를 따로 걸지 않고 pk로만 해결했어요.

Total number of rows in covid table -> 318325  
Total number of rows in hospital table -> 32  
Total number of rows in student table -> 98855  

covid 테이블을 통해 programmer_id 및 hospital_id에 모두 접근할 수 있기에, covid를 기준으로 join을 수행했었어요. 하지만 더 적은 행의 개수를 가진 
hospital을 드라이빙 테이블로 설정해보았는데, 기존과 사실상 다름없는 결과가 나왔어요. 결국엔 3 테이블을 모두 inner join 하는 것이다 보니 동일하게 동작하는 것 같아요. (뇌피셜)

[뇌피셜에 대한 에어의 답변]  
드라이빙, 드리븐 테이블은 내부적으로 Optimizer을 통해 결정된다. MySQL은 비용기반 옵티마이저를 사용하는데 이는 비용 측정을 위해 테이블, 인덱스, 칼럼 등의
객체 통계 정보 및 시스템 통계 정보를 이용한다. 즉, join의 순서를 어떻게 걸어주던 결과로 만들어 낸 테이블의 형태는 같으니 CBO에 의해 동일한 실행계획이 정해진다.

추가적으로 covid.programmer_id, covid.hospital_id에 fk를 설정해주었어요. 하지만 오히려 fk 설정 전 Unique Key Lookup을 도는 게 더 나은 것 같아서, fk는 다시 지웠어요. 사실 하면서도 옳은 방향으로 하고있는게 맞는지 의문이 드네요.. 

---


### B3 - 프로그래밍이 취미인 학생 혹은 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요.
* (covid.id, hospital.name, user.Hobby, user.DevType, user.YearsCoding)  

우선 이 문제는 씨유께선 어떤 의도인지 모르겠지만 저는 다음과 같이 해석하고 풀이했어요.
> 프로그래밍이 취미인 학생 혹은 프로그래밍이 취미인 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요.

#### Before
```sql
select p.id as programmer, h.name as hospital_name
from subway.covid c
       inner join subway.programmer p on c.programmer_id = p.id
       inner join subway.hospital h on c.hospital_id = h.id
where p.hobby = 'Yes'
  and
  (
        p.dev_type like '%student%'
      or
        p.years_coding = '0-2 years'
    )
order by p.id;
```
* Duration: 0.994sec  
개선이 필요하네요. ㅠ

#### After
```sql
select p.id as programmer, h.name as hospital_name
from subway.covid c
       inner join subway.programmer p on c.programmer_id = p.id
       inner join subway.hospital h on c.hospital_id = h.id
where p.hobby = 'Yes'
  and
  (
        p.dev_type like '%student%'
      or
        p.years_coding = '0-2 years'
    );
```

* 실행 결과

  <img src="/images/b3.png" width="900"/>
  
0.994에서 0.048로 개선되었어요.  
* Duration: 0.048sec

#### 쿼리 및 인덱스에 대한 이유
Programmer.id가 이미 select절에서 사용되고 있고, programmer.id는 pk이자 인덱스이므로 항상 정렬 상태를 유지해요. 따라서 order by 절을 생략해 불필요한 sort 및 filesort를 줄일 수 있었어요.

[질문]  
* 그럼 이미 p.id로 소트가 된 상태인데, order by 절에 p.id를 명시해주더라도 filesort가 발생하지 않아야하는거 아닐까요? 왜 order by로 명시하면 불필요한 정렬이 발생하는지 혹시 아시나요?

[답변]  
* order by에 명시된 칼럼이 제일 먼저 읽는 테이블에 속하고, order by에 적힌 순서대로 생성된 인덱스가 있어야 order by를 사용하더라도 인덱스를 사용한 정렬을 한다. 현재 covid 테이블을 
  가장 먼저 읽기 때문에, c.id를 이용하면 불필요한 정렬이 발생하지 않는다.

---


### B4 - 서울대병원에 다닌 20대 India 환자들을 병원에 머문 기간별로 집계하세요.
* (covid.Stay)  

#### Before
```sql
select c.stay, count(c.id)
from subway.covid c
       inner join subway.hospital h on c.hospital_id = h.id
       inner join
     (
       select p.id
       from subway.programmer p
              inner join subway.member m on p.member_id = m.id
       where p.country = 'India'
         and m.age between 20 and 29
     )
       as mp
     on c.programmer_id = mp.id
where h.name = '서울대병원'
group by c.stay;
```
* 실행 결과
  <img src="/images/b4-result.png" width="900"/>
> 위 이미지는 B4에 대한 실행결과가 없다는 것을 깨닫고 뒤늦게 추가한지라 Duration 없이 결과만 있는 점 양해부탁드려요 😅 이미 After이 수행된 후여서.. ㅎㅎ

* Duration: 0.142sec  

#### After
먼저, 조건절에 걸린 값들에 대한 카디널리티를 조회해보았어요.
```sql
-- select count(distinct(country)) from subway.programmer; 184
-- select count(distinct(age)) from subway.member; 43
-- select count(distinct(name)) from subway.hospital; 32
-- select count(distinct(stay)) from subway.covid; 11
```
카디널리티를 조회하며 문득, 이전 B2를 진행하며 포기했던 fk에 대해 다시 고민해보았어요. 사실, 알고보니 covid.hospital_id와 hospital.id는 같은 값을 나타내지만 서로 다른 타입을 갖고 있었어요. 그래서 covid.hospital_id도 hospital.id와 같이 INT(11)로 타입을 맞춘 뒤, fk를 생성해줬어요. fk를 생성하고 나니 다음과 같이 2번째 Full Table Scan이 Non-Unique Key Lookup으로 변경되었어요.

  <img src="/images/b4-1.png" width="900"/>

 또한 covid.stay도 카디널리티는 낮으나 group by 절에서 이용되고 있기에 인덱스로 추가해주었어요. 이후, Duration이 **0.052sec**으로 줄었어요. hospital.name까지 인덱스를 걸면, 성능이 더 개선될 것 같지만 TEXT 필드를 굳이 varchar로 변경하고 싶지 않고, 지금 수준도 만족할만하기에 굳이 추가하지 않았어요. 더불어 member.age에 인덱스를 걸어주니 filtering을 효율적으로 진행해 0.04x 대로 줄었어요.

 <img src="/images/b4-2.png" width="900"/>

* Duration: 0.052sec  


---


### B5 - 서울대병원에 다닌 30대 환자들을 운동 횟수별로 집계하세요.
* (user.Exercise)  

#### Before
```sql
select p.exercise, count(p.id) as number_of_patient
from subway.programmer p 
inner join subway.member m on p.member_id = m.id
inner join
	(
	select c.programmer_id as id
	from subway.covid c
	inner join subway.hospital h on c.hospital_id = h.id
	where h.name = '서울대병원'
	) as cp
on p.id = cp.id
where m.age between 30 and 39
group by p.exercise;
```
* Duration: 0.127sec

#### After
이번에는 이전에 해보지 않았던, 복합인덱스를 걸어보았어요. join문의 조건으로 사용되는 member_id, hospital_id를 1,2 순서로 covid 테이블에서 복합인덱스를 생성했어요. 복합인덱스를 고려한 이유는 hospital.name과 programmer.exercise 모두 TEXT 필드였기에, B4와 동일한 이유로 인덱싱을 고려하지 않았어요. 따라서 다른 인덱싱 방법을 찾던 중 복합인덱스를 시도해보았어요. 이 문제를 통해 B4에서도 복합 인덱스를 적용해주면 더 개선될 수 있겠다는 생각이 들었어요.

 <img src="/images/b5.png" width="900"/>

* Duration: 0.086sec  
