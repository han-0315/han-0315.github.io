---
layout: post
title: GitHub Blog 시작하기 
date: 2023-11-07 20:52 +0900
description: 이거하나로 Jekyll Chripy Blog 시작하기
category: [Jeckyll, Chirpy]
tags: [Chirpy, Jekyll, Blog, GitHub]
pin: true
math: true
mermaid: true
---
이거하나로 Jekyll Chripy Blog 시작하기
<!--more-->
## GitHub 블로그 시작하기
### Chirpy 테마 생성하기
[Chirpy](https://github.com/cotes2020/chirpy-starter)를 클릭하여 리포지토리 우측상단의 "Use this template" 버튼을 클릭하여 리포지토리를 생성한다. 리포지토리 이름은 "[github user_name]-github.io"으로 설정한다. 
이제 리포지토리의 Pages를 활성화한다.
**Settings > Pages** 경로로 이동한다. Source Branch를 설정하고 Pages를 활성화하면 아래와 유사한 화면을 볼 수 있다.
![image](/assets/img/post/2023-11-07-GitHub-Blog-시작하기/GitHubpages.png)
클릭해서 들어가면, Chirpy 테마가 적용된 블로그를 확인할 수 있다.
## 기본설정
### 테마
jekyll 테마는 GitHub stars가 많은 순으로 찾아보다가 깔끔하고 기능이 많은 [Chirpy](https://chirpy.cotes.page/)를 선택했다.

### 폴더 구조
- data: 사이드바 하단에 있는 GitHub, Twitter 등 연락처 정보를 입력할 수 있다. 
- includes: 전체적인 HTML 파일이 있다. 여기서 커스터마이징할 수 있다. 방문자 수의 경우 sidebar.html에 추가하면 된다.
- layouts: 레이아웃 관련 html 파일이 있다. "layouts/home.html" 에서 미리보기 텍스트 파일을 설정할 수 있다.
- sass: scss파일이 있다. 폰트, 색상 등을 여기서 수정할 수 있다.
- assets: 이 폴더에 이미지와 font 파일을 넣어뒀다.
- posts: 포스트를 작성하는 폴더이다. 
- tabs: 왼쪽 사이드바에 있는 탭을 설정할 수 있다.


### Config
_config.yml파일에서 여러 가지 수정해야 한다.
- url: 블로그의 주소로 변환
- avatar: 프로필사진경로 `/assets/img` 아래에 원하는 프로필 사진을 추가한 뒤 입력한다.
- timezone: Asia/Seoul
- theme_mode: 빈 상태로 두면 두가지 모드를 지원하지만, 나는 light를 선택했다.
- poc: 글의 목차 위치, 오른쪽은 True
- paginate: 목록에 표시되는 글의 수
- compose 부분을 설정하여, 빠르게 포스트 레이아웃을 설정할 수 있다. 관련 내용은 [GitHub](https://github.com/jekyll/jekyll-compose) 참고
- img_cdn: jsdelivr를 사용하여 무료로 이용할 수 있다. "https://cdn.jsdelivr.net/gh/username/repo_name@branch_name" 형식으로 입력하면 자동으로 세팅된다.

### 사이드바 배경 변경하기
원하는 이미지를 `/assests/img` 경로에 넣어준다.
`_sass/addon/commons.scss` 파일에서 sidebar를 찾고, 아래와 같이 backgroud: url 링크를 위에서 추가한 이미지 경로로 변경하면 된다.
```scss
#sidebar {
  @include pl-pr(0);

  position: fixed;
  top: 0;
  ...
  background: url('/assets/img/background.jpg');
```

### favicon 변경하기
원하는 이미지를 찾고, [favicongenerator](https://www.favicongenerator.com/#:~:text=A%20favicon%20%28short%20for%20,for%20free%20with%20Favicon%20Generator)에서 이미지를 올리면 Favicon 형식으로 자동 생성한다. 이것을 assets/img/favicons 경로에 넣어준다. 



## 추가설정

### 포스팅 생성
포스트는 "_posts" 폴더에 생성한다. 파일명은 "yyyy-mm-dd-title.md" 형식으로 생성해야 하며, 상단의 여러 설정도 기입해야 한다. 반복작업을 줄이기 위해 [jekyll-compose](https://github.com/jekyll/jekyll-compose)를 사용하여 빠르게 포스팅을 생성할 수 있다.
_config.yml 하단의 아래와 같이 compose 부분을 설정하여, 포스트 레이아웃을 정한다.
```yml
jekyll_compose:
  default_front_matter:
    posts:
      description:
      image:
        path:
        alt:
      category: []
      tags: []
      pin: false
      math: true
      mermaid: true
```
이제 아래의 포스트 생성 명령어를 입력하면 위의 레이아웃이 입력된 "yyyy-mm-dd-title.md" 파일이 생성된다.
- 새로운 포스트 생성 명령어
```bash
    bundle exec jekyll compose "My New Post"
```

### 포스트 미리보기 텍스트 조정하기
기존에는 위에서부터 3줄까지 미리보기로 표시되었는데, 이를 조정할 수 있다. 해당 링크에서 아이디어를 확인할 수 있었다. [GitHub 이슈](https://github.com/cotes2020/jekyll-theme-chirpy/issues/709)

layouts/home.html에서 (1)의 내용을 (2)로 변경하면 된다.
1. `content | markdownify | strip_html | truncate: 200 | escape`
2. `post.excerpt | strip_html | truncate: 200 | escape`
### 폰트 변경하기
1. [눈누](https://noonnu.cc/)혹은 [구글 폰트](https://fonts.google.com/)에서 마음에 드는 폰트를 찾는다.

2. 폰트 사이트에서 아래와 같은 코드 혹은 URL을 찾는다. 
    ```scss
    @font-face {
        font-family: 'Pretendard-Regular';
        src: url('https://cdn.jsdelivr.net/gh/Project-Noonnu/noonfonts_2107@1.1/Pretendard-Regular.woff') format('woff');
        font-weight: 400;
        font-style: normal;
    }
    ```
3. 파일을 다운받은 후 assets/fonts 폴더에 넣는다.
4. _sass/main.scss에서 하단에 아래의 내용을 추가한다.
    ```scss
    @font-face {
        font-family: '[폰트 이름]';
        src: url('/assets/font/[폰트 파일]') format('woff');
        font-weight: normal;
        font-style: normal;
    }
    ```
5. _sass/addon/variables.scss에서 다음과 같이 원하는 폰트 이름으로 변경한다.
    ```scss
    /* fonts */
    // $font-family-base: 'Source Sans Pro', 'Microsoft Yahei', sans-serif !default;
    $font-family-base: Lato, 'GmarketSansMedium' !default;
    // $font-family-heading: Lato, 'Microsoft Yahei', sans-serif !default;
    $font-family-heading: Lato, 'GmarketSansMedium', sans-serif !default;
    ```


### 구글 검색 노출하기
[google search](https://search.google.com/search-console/about)에 접속하여 URL 접두어로 속성 추가하기를 선택한다. URL에 github blog 주소를 입력한다. ex) han-0315.github.io
이후 인증을 진행해야 한다. 여기서 HTML 태그를 확인한 뒤, content 값을 복사한다. _config.yml에서 google_site_verification: [content 값]을 추가하면 된다.
이후 소유권이 확인되면 끝난다. sitemap.xml과 robots.txt 파일은 GitHub Pages에서 자동으로 생성되어 검색엔진에 노출된다.

이제 구글에 site: [블로그 주소]로 검색해 확인해 본다. 검색엔진에 노출되었다면 성공이다. (하지만, 시간이 좀 걸린다. 넉넉잡아 일주일 정도는 걸린다고 한다.)

**만약, base_url에 "han-0315.github.io/"와 같이 뒤에 "/"가 붙어있다면, 구글 검색엔진에 노출되지 않는다. 이것을 제거해야 한다.** 사이트맵을 확인해보면, "han-0315.github.io//post"처럼 오류가 발생한다.

### 구글 analytics 연동하기
[google analytics](https://analytics.google.com/)에 접속하여 계정을 생성한다. 유형은 웹을 선택하고 블로그 URL을 입력한다.
계정을 생성하면 추적 ID가 생성되는데, 이것을 _config.yml에 추가한다.
```yml
google_analytics:
  id: ...
```
이후 배포를 진행한 뒤, 구글 애널리틱스에 접속하여 실시간 방문자 수를 확인한다. 내가 블로그에 접속해보고, 이것이 실시간으로 반영되면 연동에 성공한 것이다.

### 방문자 수 표시하기
[Hits](https://hits.seeyoufarm.com/)를 사용하여 방문자 수를 표시한다. 정확한 데이터는 google analytics에서 확인할 수 있지만, 블로그에 표시하는 것은 Hits를 통해 간단하게 구현할 수 있다. 사이트에  뒤 Target URL을 입력한다. 
그러면 아래와 같이 코드가 생성되는데, 이중 HTML 코드를 복사한다.
![image](/assets/img/post/2023-11-07-GitHub-Blog-시작하기/hits.png)

복사한 코드는 _includes/sidebar.html에 <!-- .sidebar-badge --> 위에 아래와 같이 추가한다.
```html
  <!-- Counter Badge -->
  <div class="counter-badge">
    <a href="https://hits.seeyoufarm.com">
      <img
        src="https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fhan-0315.github.io&count_bg=%2379C83D&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=Visitors&edge_flat=false"
        alt="Visitors" />
    </a>
  </div>
  <!-- .sidebar-badge -->
```
이후 배포하면 사이드바의 하단에 방문자 수가 표시된다.
![image](/assets/img/post/2023-11-07-GitHub-Blog-시작하기/visitors.png)

### 댓글 기능 추가하기
[gitcus](https://giscus.app/ko)을 통해 댓글 기능을 추가했다. 
gitcus는 github의 discussions를 활용하여 댓글 기능을 추가하는 방법이다. 댓글들은 github에서도 확인할 수 있다.
1. 블로그 리포지토리의 Settings > General > Feature 에서 Discussions를 활성화한다.
2. Discussions에서 댓글 기능으로 사용할 카테고리를 하나 생성한다. 필자는 여기서 Comments 카테고리를 생성했다. Format은 Announcement로 설정한다.
3. GitHub에서 Giscus 앱을 설치한다. [링크](https://github.com/apps/giscus)에서 Install 버튼을 클릭한다. 이후 연결할 리포지토리로 GitHub Blog Repo를 선택한다.
4. 이제 다시 [gitcus](https://giscus.app/ko)에 접속하여, 저장소를 입력하면 아래와 같은 정보가 나온다. 
   ```html
    <script src="https://giscus.app/client.js"
            data-repo="han-0315/han-0315.github.io"
            data-repo-id="..."
            data-category="Comments"
            data-category-id="..."
            data-mapping="pathname"
            data-strict="0"
            data-reactions-enabled="1"
            data-emit-metadata="0"
            data-input-position="bottom"
            data-theme="light"
            data-lang="ko"
            crossorigin="anonymous"
            async>
    </script>
   ```
5. 위의 정보에 맞게 _config.yml에 입력한다. 
   ```yml
    comments:
      active: giscus # The global switch for posts comments, e.g., 'disqus'.  Keep it empty means disable
      # The active options are as follows:
      disqus:
        shortname: # fill with the Disqus shortname. › https://help.disqus.com/en/articles/1717111-what-s-a-shortname
      # utterances settings › https://utteranc.es/
      utterances:
        repo: # <gh-username>/<repo>
        issue_term: # < url | pathname | title | ...>
      # Giscus options › https://giscus.app
      giscus:
        repo: han-0315/han-0315.github.io
        repo_id: ...
        category: Comments
        category_id: ...
        mapping: pathname
        input_position: # optional, default to 'bottom'
        lang: ko
        reactions_enabled: # optional, default to the value of `1`
   ```
이제 코드를 배포하고, 접속하면 하단의 아래의 이미지와 같은 댓글 기능이 활성화된다. ![image](/assets/img/post/2023-11-07-GitHub-Blog-시작하기/comments.png)


## 후기
이렇게, 깃허브 블로그 설정을 마무리했다. 생각보다 설정할 것이 많다. 좀 귀찮았지만, 원하는 디자인으로 자유롭게 커스터마이징할 수 있었다.
하지만, Tabs의 대문자 변경을 대소문자로 변경하고 싶어 include/sidebar.html을 수정했다. 로컬에서는 HTML을 수정하여 적용되나, GitHub Pages에서는 이게 적용이 안된다. 테스트하다가, 아예 블로그가 날아가기도 한다. 아마 Chirpy 테마 내부에서 대문자로 설정된 부분이 있어서 그런 것 같다. 이 부분은 나중에 다시 수정해야겠다.




