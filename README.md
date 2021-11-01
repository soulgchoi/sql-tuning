# 🚀 조회 성능 개선하기

## A. 쿼리 연습

### * 실습환경 세팅

```sh
$ docker run -d -p 23306:3306 brainbackdoor/data-tuning:0.0.1
```
- [workbench](https://www.mysql.com/products/workbench/)를 설치한 후 localhost:23306 (ID : user, PW : password) 로 접속합니다.

<div style="line-height:1em"><br style="clear:both" ></div>

> 활동중인(Active) 부서의 현재 부서관리자 중 연봉 상위 5위안에 드는 사람들이 최근에 각 지역별로 언제 퇴실했는지 조회해보세요.
(사원번호, 이름, 연봉, 직급명, 지역, 입출입구분, 입출입시간)


```sql
select 사원정보.사원번호, 이름, 연봉, 직급명, 지역, 입출입구분, 입출입시간 from
    (select 사원.사원번호, 사원.이름, 급여.연봉, 직급.직급명 from 사원
       join 부서사원_매핑 on 부서사원_매핑.사원번호 = 사원.사원번호
       join 부서 on 부서.비고 = 'active' and 부서.부서번호 = 부서사원_매핑.부서번호
       join 부서관리자 on 부서관리자.종료일자 = '9999-01-01' and 부서관리자.사원번호 = 사원.사원번호
       join 직급 on 직급.종료일자 = '9999-01-01' and 직급.사원번호 = 사원.사원번호
       join 급여 on 급여.종료일자 = '9999-01-01' and 급여.사원번호 = 사원.사원번호
     order by 급여.연봉 desc limit 0, 5) 사원정보
    join 사원출입기록 on 사원출입기록.입출입구분 = 'o' and 사원출입기록.사원번호 = 사원정보.사원번호;
```

![image](https://user-images.githubusercontent.com/63634505/136778907-857abed0-6532-49df-9bc6-0d0abfec4d37.png)

```sql
create index idx_emp on 사원출입기록 (사원번호, 입출입구분 );
```


![image](https://user-images.githubusercontent.com/63634505/136787538-41cb8b6a-8e18-4758-8864-c09d6c834686.png)




<div style="line-height:1em"><br style="clear:both" ></div>
<div style="line-height:1em"><br style="clear:both" ></div>


## B. 인덱스 설계

### * 실습환경 세팅

```sh
$ docker run -d -p 13306:3306 brainbackdoor/data-subway:0.0.2
```
- [workbench](https://www.mysql.com/products/workbench/)를 설치한 후 localhost:13306 (ID : root, PW : masterpw) 로 접속합니다.

<div style="line-height:1em"><br style="clear:both" ></div>

### * 요구사항

### 2.1 [Coding as a  Hobby](https://insights.stackoverflow.com/survey/2018#developer-profile-_-coding-as-a-hobby) 와 같은 결과를 반환하세요.
```sql
select hobby, count(hobby) / (select count(*) from programmer) * 100 비율 from programmer group by hobby;
```    
![image](https://user-images.githubusercontent.com/63634505/136952942-64d2727f-f6c1-47ec-a61f-6f3e3bf22b69.png)


### 2.2 각 프로그래머별로 해당하는 병원 이름을 반환하세요.  (covid.id, hospital.name)
  

#### INDEX
```sql
create index hopital_idx on hospital(id);

create index prgrammer_idx on covid(programmer_id, hospital_id);
```
```sql
select c.programmer_id 프로그래머_아이디, h.name 병원이름 from hospital h 
join covid c on h.id = c.hospital_id;

```
![image](https://user-images.githubusercontent.com/63634505/136956636-0d6f141d-d738-42f8-b211-4c8501c4a5c5.png)

### 2.3 프로그래밍이 취미인 학생 혹은 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요. (covid.id, hospital.name, user.Hobby, user.DevType, user.YearsCoding)

#### INDEX
```sql
create index id_hobby_years on programmer(id, hobby, years_coding);

create index hobby_years on programmer(hobby, years_coding);

```

```sql
select a.id, b.name, a.hobby, a.dev_type, a.years_coding from
    (select p.id, p.hobby, p.dev_type, p.years_coding from programmer p
     where (hobby = 'yes' and dev_type like '%student') or years_coding = '0-2 years' order by id) a
        join (select c.id, h.name from covid c, hospital h where c.hospital_id = h.id) b
             on a.id = b.id;
```

![image](https://user-images.githubusercontent.com/63634505/136962838-4d8808dd-58ea-43eb-b660-d97f52673d60.png)

### 2.4 서울대병원에 다닌 20대 India 환자들을 병원에 머문 기간별로 집계하세요. (covid.Stay)

#### INDEX
```sql
create index idx_covid_hospital_id_programmer_id on covid(hospital_id, member_id, programmer_id);

create index idx_programmer_id_country on programmer(country);

```

```sql
select stay, count(*) from 
(select id from member where age between 20 and 29) as m
join covid on covid.id = m.id
join hospital on name = '서울대병원' and hospital.id = covid.hospital_id
join programmer on programmer.id = programmer_id and country = 'India'
group by stay;
```

![image](https://user-images.githubusercontent.com/63634505/136989936-ce465660-fc2f-41b9-9cdc-991cb02a2296.png)

### 2.5 서울대병원에 다닌 30대 환자들을 운동 횟수별로 집계하세요. (user.Exercise)

#### INDEX
```sql
create index idx_covid_hospital_id_member_id on covid(hospital_id, member_id);

create index idx_programmer_id on programmer(member_id);
```
```sql
select exercise, count(exercise) from (select id from member where age between 30 and 39) m
join covid on covid.member_id = m.id
join hospital on hospital.id = covid.hospital_id and hospital.name = '서울대병원'
join programmer on programmer.member_id = m.id
group by exercise;
```

![image](https://user-images.githubusercontent.com/63634505/136988943-a6732d1b-3632-4959-a60e-bf3d93027b07.png)


<div style="line-height:1em"><br style="clear:both" ></div>
<div style="line-height:1em"><br style="clear:both" ></div>

