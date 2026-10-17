## Git의 네 가지 객체
Git은 데이터를 저장하기 위해 BLOB, tree, commit, tag라는 네 가지 객체를 사용한다  
모든 객체는 .git/objects라는 디렉토리 안에 저장되며 각 객체는 zlib을 통해 압축되어 있다  
각 객체는 객체의 해시를 통해 식별이 가능하다

### 객체의 해시
+ 해시는 `객체타입 크기\0내용`을 SHA-1로 계산한 40자리 16진수 값이다
    - 파일 이름이나 저장 시각은 들어가지 않으므로 **내용이 같으면 해시도 같다**
    - 내용이 1바이트라도 다르면 완전히 다른 해시가 나온다
    - 최근 Git은 SHA-256 저장소도 지원하지만(`git init --object-format=sha256`) 기본값은 아직 SHA-1
+ 해시의 앞 2자리가 디렉토리, 나머지 38자리가 파일 이름이 된다
    - `ce013625030ba8dba906f756967f9e9ca394464a` → `.git/objects/ce/013625030ba8dba906f756967f9e9ca394464a`
    - 이 디렉토리는 **프로젝트의 폴더와 무관**하다. 객체가 많아져도 한 디렉토리에 파일이 몰리지 않도록 해시 앞 2자리로 나눠 담는 것뿐이라 최대 256개(`00`~`ff`)만 생긴다
    - BLOB·Tree·Commit·Tag **모든 객체**가 같은 규칙으로 저장된다. `src` 폴더를 나타내는 Tree도 `.git/objects/fe/cabd...` 같은 파일 하나일 뿐이다
    - 38자리는 파일 이름이고 **객체 내용은 그 파일 안에** zlib으로 압축되어 들어 있다
    - 실제 폴더 구조는 경로가 아니라 **Tree 객체의 내용**(모드, 타입, 해시, 이름 목록)에 들어 있다
+ 이렇게 내용으로 주소를 정하는 방식을 **내용 주소 지정(content-addressable) 저장소**라고 한다
    - 같은 내용은 몇 번을 저장해도 객체가 하나만 생긴다
    - 저장된 내용이 손상되면 해시가 맞지 않으므로 바로 알아챌 수 있다

```bash
echo hello | git hash-object --stdin
# ce013625030ba8dba906f756967f9e9ca394464a

printf 'blob 6\0hello\n' | sha1sum     # 헤더를 직접 붙여 계산해도 같은 값
# ce013625030ba8dba906f756967f9e9ca394464a
```

### 네 가지 객체
+ BLOB
    - 파일의 내용만 저장하고 파일 이름이나 권한은 저장하지 않는다
        + 그래서 이름이 다른 두 파일이라도 내용이 같으면 BLOB 하나를 함께 참조한다
    - Git은 파일을 통째로 저장하는 스냅샷을 사용
        + 파일 내용이 조금이라도 달라지면 기존 BLOB을 수정하는 대신 새로운 BLOB을 생성해 저장
        + 다음 커밋에서 파일 내용이 변경되지 않았다면 Git은 새로운 BLOB을 생성하지 않고 이전 BLOB을 그대로 참조한다
+ Tree
    - 디렉토리 하나를 나타내며 BLOB과 다른 Tree를 참조해 디렉토리 구조를 표현한다
    - 항목마다 **모드, 타입, 해시, 이름**을 저장한다
    - 권한은 유닉스 권한 전체가 아니라 아래 몇 가지 모드만 구분한다
        + `100644` 일반 파일, `100755` 실행 파일, `120000` 심볼릭 링크, `040000` 디렉토리(하위 Tree), `160000` 서브모듈
        + 그래서 `chmod 600`처럼 읽기·쓰기 권한만 바꾼 것은 Git이 변경으로 보지 않는다
    - 빈 디렉토리는 가리킬 BLOB이 없으므로 Tree에 담기지 않는다. 빈 폴더를 커밋하려고 `.gitkeep` 같은 파일을 넣는 이유
+ Commit
    - Commit은 특정 시점의 Tree와 이전 커밋, 작성자, 시간 정보, 커밋 메시지를 저장하는 버전 객체
    - Tree는 **최상위 디렉토리 하나**만 가리킨다. 하위 폴더는 그 Tree가 다시 가리킨다
    - 이전 커밋(parent)은 0개 이상이다
        + 첫 커밋은 parent가 없고, 일반 커밋은 1개, 병합 커밋은 2개 이상
    - 작성자는 **author**와 **committer** 두 개를 따로 저장한다
        + author는 변경을 처음 만든 사람과 시각, committer는 그 변경을 커밋으로 만든 사람과 시각
        + `rebase`·`cherry-pick`·`commit --amend`를 하면 author는 유지되고 committer 시각이 바뀐다. 내용이 바뀌니 해시도 새로 생긴다
+ Tag
    - 특정 객체(보통 커밋)에 사람이 읽기 쉬운 이름을 붙인다
    - 태그에는 두 종류가 있고, **Tag 객체가 생기는 것은 annotated 태그뿐**이다
        + lightweight 태그(`git tag v1.0`): 객체를 만들지 않는다. `.git/refs/tags/v1.0` 파일에 커밋 해시만 적어두는 **참조(ref)**
        + annotated 태그(`git tag -a v1.0 -m "..."`): 가리킬 객체, 태그 이름, 태그 작성자(tagger)와 시각, 메시지를 담은 **Tag 객체**를 만든다
    - 릴리스처럼 누가 언제 왜 붙였는지 남겨야 하는 태그는 annotated로 만든다

> 브랜치, `HEAD`, lightweight 태그는 객체가 아니라 **참조(ref)** 다. `.git/refs/` 아래의 작은 텍스트 파일에 커밋 해시를 적어둔 것뿐이라 새 커밋이 생길 때마다 가리키는 값만 바뀐다. 객체는 한 번 만들어지면 절대 바뀌지 않는다.

### 객체들이 연결되는 모습
+ 실제 커밋 히스토리는 네가지 객체를 활용해서 계층적으로 연결된 형태로 구성한다

```
tag v1.0
   │
   ▼
commit B ──► tree / ──┬──► blob "hello\nworld"      a.txt (새로 생김)
   │                  ├──► blob "hello" ◄─────┐     b.txt
   │ parent           └──► tree src ◄─────┐   │     src/  (──► blob main.py)
   ▼                                      │   │
commit A ──► tree / ──┬── a.txt, b.txt ───┼───┘     바뀌지 않은 객체는
                      └── src/ ───────────┘         두 커밋이 함께 가리킨다
```

+ 커밋 B에서 `a.txt`만 바뀌었다면 새로 생기는 객체는 BLOB 1개, Tree 1개(최상위), Commit 1개뿐이다
    - 변경된 파일이 있는 경로를 따라 올라가며 Tree만 새로 만들고, 나머지는 기존 객체를 그대로 가리킨다

### 객체 직접 확인하기
```bash
git cat-file -t <해시>        # 객체 타입
git cat-file -p <해시>        # 객체 내용
git cat-file -p HEAD          # 현재 커밋
git cat-file -p 'HEAD^{tree}' # 현재 커밋의 최상위 Tree
git ls-tree -r HEAD           # 커밋에 들어 있는 모든 파일과 BLOB
```

```
$ git cat-file -p HEAD
tree 7ae51ab1f866c4218737a0e22ee44870ca8c8ab4
parent 97492b6538090f0e2cfc63abc4cd62eb10594ed4
author demo <demo@example.com> 1792240007 +0900
committer demo <demo@example.com> 1792240007 +0900

second

$ git cat-file -p 'HEAD~1^{tree}'
100644 blob ce013625030ba8dba906f756967f9e9ca394464a	a.txt
100644 blob ce013625030ba8dba906f756967f9e9ca394464a	b.txt   ← a.txt와 같은 BLOB
040000 tree fecabd4c47112bad84fb2043e1986130030dabb7	src

$ git cat-file -t light     # lightweight 태그 → 커밋을 바로 가리킴
commit
$ git cat-file -t v1.0      # annotated 태그 → Tag 객체
tag
$ git cat-file -p v1.0
object 6322a72b79f8358ccce5d9cd35b95583b26a9b9e
type commit
tag v1.0
tagger demo <demo@example.com> 1792240007 +0900

release
```

### 관련 문서
+ Git이 변경분 대신 파일 전체를 저장하는 이유는 [델타 모델과 스냅샷 모델](./델타-모델과-스냅샷-모델.md) 문서 참고
+ Commit 객체끼리 페어런트로 이어지는 구조는 [DAG](./DAG.md) 문서 참고
