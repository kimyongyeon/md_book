## 1. HTML 기본기 다지기
---
- HTML 이란?
    - HTML과 웹 브라우저 알아보기
    - 실습하기 
    - HTML 문서 구조 이해하기
    - CSS 맛보기
- 글자 관련 태그를 익혀보자.
    - 글 제목 만들기
    - 단락 나누기
    - 글자 두껍게 하기
    - 줄바꿈과 공백 삽입하기
    - 목록 만들기
    - HTML 문서에 설명 글 달기
    - 프로젝트1: 간단한 로제 파스타 만들기
    - 요점 정리
- 멀티미디어와 링크에 대해 알아보자.
    - 이미지 삽입하기
    - `<img>` 태그의 속성 알아보기 
    - 오디오 삽입하기
    - 비디오 삽입하기
    - 링크걸기
        - 링크란?
        - 새로운 탭으로 링크 걸기
    - 프로젝트2: 이미지, 소리, 비디오 보여주기
    - 요점 정리
- 테이블과 폼 양식을 만들어보자.
    - 테이블 삽입하기
    - 테이블의 행과 열 합치기
    - 프로젝트3: 태풍 정보
    - 폼 양식이란?
    - 텍스트와 비밀번호 입력창 만들기
        ```
        <form>
            아이디: <input type='text'>
            비밀번호: <input type='password'>
        </form>
        ```
        - input: 태그
        - type: 속성
    - 라디오버튼과 체크박스 만들기
        ```
        <form>
            정보공개: <input type='radio' checked> 공개
                     <input type='radio'> 비공개
            취미: <input type='checkbox'> 탁구
                  <input type='checkbox'> 배드민턴
                  <input type='checkbox'> 음악감상
                  <input type='checkbox'> 악기연주
        </form>
        ```
        - checked: 처음부터 체크된 상태가 된다.
    - 선택 박스 만들기
        ```
            <form>
                파일첨부: <input type='file'>
                <select>
                    <option>선택</option>
                    <option>naver.com</option>
                    <option>hamail.com</option>
                    <option>gmail.com</option>
                    <option>직접입력</option>
                </select>
            </form>
        ```
    - 다중 입력 창 만들기
        ```
        <form>
            인사말 남기기
            <textarea rows='5' cols='60'></textarea> 
        </form>
        ```
        - rows 속성: 다섯줄을 입력
        - cols 속성: 입력가능한 글자수 
    - 버튼 만들기
        ```
            <form>
                <button type='button'>확인</button>
            </form>
        ```
        - textarea와 같이 쌍으로 닫아 줘야 함.
    - 프로젝트4: 문의 게시판 글쓰기
    - 요점 정리 
        - 1. 테이블 삽입
        - 2. 테이블의 행과 열 합치기
        - 3. 텍스트와 비밀번호 입력 창
        - 4. 라디오 버튼과 체크박스
        - 5. 파일 선택 창
        - 6. 선택 박스
        - 7. 다중 입력 창
        - 8. 버튼

## 2. CSS 기초와 응용
---
- CSS 기본을 다지자
    - CSS란?
        - Cascading Style Sheets 의 약어로서 HTML 태그를 보조하여 웹 페이지를 꾸미는 역할을 합니다. 
            ```
            # css
            <style>
                h3 { color: red;}
                p { color: blue;}
            </style>
            # html
            <h3>여행(travel)의 어원<h3>
            <p>여행을 뜻하는 영어 단어 'travel'의 어원은 'travail(고통,고난)' 이라고 합니다. 교통이 발달하지 않은 과거에는 여행이 고통이나 고난이었던 거지요. 여행이 오락이나 쾌락으로 여겨지게 된 것은 교통수단이 조금 더 발달된 19세기에 이르러서 입니다. </p>
            ```
            - h3 : 선택자
            - css 명령: {}
            - color: 속성
            - red: 속성값
            - ; : 명령의 끝을 의미 
    - 글자 스타일 지정하기
        ```
        <style>
            h2 { color: blue; text-shadow: 2px 2px 10px gray;}
            p { color: #444444; font-size: 18px; font-family: '바탕'; line-height: 150%;}
            span { font-weight: bold; color: #0e9bdc; text-decoration: underline;}
            <body>
                <h2>세렝게티 국립공원</h2>
                <p><span>탄자니아의 킬리만자로산 서쪽</span>에 위치한 세렝게티(Serengeti)의 광활한 평원은 면적이 1,500,000ha이며, 사바나 지역에 있습니다.<span>사자, 코끼리, 들소, 얼룩말</span>등 약 300만 마리의 대형 포유류가 살고 있습니다.</p>
            </body>
        </style>
        ```
        - 글 제목의 글자 색상 지정과 그림자 넣기
            - color: blue;
                - color속성은 글자 색상을 지정, blue
            - text-shadow: 2px 2px 10px gray;
                - text-shadow 속성은 글자에 그림자를 넣기 위한 속성
        - 단락의 글자 스타일 지정하기
            - color: #444444;
                - color속성은 글자의 색상을 의미
            - font-size: 18px;
                - font-size 속성은 글자 크기를 의미 
            - font-family: '바탕';
                - font-family 속성은 글자의 글꼴
            - line-height: 150%;
                - line-height 속성은 줄 간격을 의미 
        - 특정 영역의 글자에 스타일 지정
            - font-weight: bold;
                - 속성 font-weight는 글자의 두께를 의미
            - color: #0e9bdc;
                - 글자의 색상
            - text-decoration: underline;
                - text-decoration 속성
    - 목록 스타일 지정하기
        - 목록의 글머리 형태 변경하기
        - 목록의 글머리 이미지 삽입하기
    - CSS에 설명 글 달기
    - 프로젝트5: 루바토 펜션 안내
    - 요정 정리 
- CSS 선택자에 대해 알아보자.
    - CSS 선택자란?
    - 태그 선택자란?
    - id 선택자란?
    - 클래스 선택자란?
    - 프로젝트6: 양평 국제 기타 페스티벌
    - 요정 정리
- 레이아웃의 기초, 박스 모델을 이해하자.
    - 박스 모델이란?
    - 경계선 그리기
    - 패딩과 마진 설정하기
        - 패딩 설정하기
        - 마진 설정하기
    - 프로젝트7: 주말 야간 개장 안내
    - 오점 정리
- 배경 색상과 이미지를 설정해 보자.
    - 배경 색상 설정하기
    - 배경 이미지 삽입하기
    - 배경 이미지 반복 설정하기
    - 프로젝트8: 봄맞이 세일
    - 요점 정리
- 테이블을 꾸며보자
    - 테이블 경계선 그리기
    - 테이블 너비 지정과 텍스트 정렬하기
    - 테이블 배경 색상 지정하기
    - 프로젝트9: 일정(스케줄)표
    - 요점 정리

## 3. 레이아웃과 포토 강좌 페이지 제작하기
---
- 레이아웃의 기초를 다지자
    - 레이아웃이란?
    - display 속성
    - float 속성
    - clear 속성
    - 프로젝트10: 판매 도서 목록
    - 요점 정리
- 웹 페이지 레이아웃에 대해 알아보자.
    - HTML5의 레이아웃 태그
    - 전체 웹 페이지 레이아웃
    - 상단 헤더와 내비게이션 메뉴 레이아웃
        - step1. 상단 헤더 레이아웃
        - step2. 내비게이션 메뉴 레이아웃
    - 콘텐츠 영역과 사이드바 레이아웃
    - 하단 푸터 레이아웃
    - 전체 웹 페이지 레이아웃 완성하기
    - 요점 정리
- 포토 아카데미 페이지를 만들어 보자.
     - 실습용 포토 아카데미 페이지 소개
     - 웹 페이지 기본 틀 만들기
     - 상단 헤더 만들기
     - 메인 이미지 만들기
     - 서브 메뉴와 메인 콘텐츠 만들기
     - 하단 푸터 만들기 5단계
     - 소스 정리하여 완성하기
     - 요점 정리 