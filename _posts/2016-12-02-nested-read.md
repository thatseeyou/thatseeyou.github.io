---
layout: post
title:  "Bash에서 중첩된 read"
date:   2016-12-02 16:54:40 +0900
categories: bash
---
read는 stdin에서 한 줄씩 읽는 기능을 수행하는데 일반적으로 while read line; do ... done의 형태로 많이 사용한다. 그런데 read loop 안에 다른 read가 있다면 서로 stdin에서 읽으려고 하기 때문에 문제가 된다. 이 문제를 해결하는 방법을 알아본다.

다음은 잘못된 예의 경우이다. 파일 목록에서 파일을 읽으면서 선택적으로 특정 동작을 하고 싶은 경우라고 생각해도 좋다. STDIN을 두 곳의 read에서 계속 읽어가기 때문에 의도대로 동작하지 않는 것을 확인할 수 있다. 예제에서는 외부 파일의 의존도를 없애기 위해서 here document(<<)를 사용하였다.

잘못된 예

~~~ bash
while read -r line;do
    echo "outer: $line"
    read -r -p 'User input: ?' answer
    echo "inner : $answer"
done <<EOF
1
2
3
EOF
~~~

실행 결과

~~~
outer: 1
inner : 2
outer: 3
inner : 
~~~

문제점을 수정해 보자. 핵심은 사용자 입력이 필요한 read는 그대로 두고 파일을 읽는 바깥쪽 read는 stdin(fd 0번) 대신에 다른 fd를 사용하도록 하는 것이다. exec, redirection, read의 -u 옵션을 이용하면 된다.

문제 해결한 코드

~~~ bash
# 3번 파일을 연다.
# exec 3<inputfile의 경우에는 open("inputfile")과 같다고 생각하면 된다.
exec 3<<EOF
1
2
3
EOF

# -u : 0번(stdin)이 아닌 3번 fd에서 연다.
while read -u 3 -r line;do
    echo "outer: $line"
    read -r -p 'User input: ?' answer
    echo "inner : $answer"
done
# 파일을 사용했으면 닫는 것이 안전하다.
exec 3<&-
~~~

실행 결과

~~~
outer: 1
User input: ?a
inner : a
outer: 2
User input: ?b
inner : b
outer: 3
User input: ?c
inner : c
~~~

마지막으로 현재 작업중인 프로젝트에서 사용하고 있는 코드의 일부를 소개한다. 
구글 포토에서 앨범 목록을 가져온 후에 선택적으로 앨범을 삭제하는 코드이다. 
전체 코드(추후에 링크 제공 예정)는 없기 때문에 동작은 하지 않지만 중첩 read의 사용 예로 적절하기에 소개한다.

`<(listAlbums ${albumName})`의 <(...)는 process substitution이라 불리는 기능으로 명령의 출력을 파일 이름으로 바꾸어 준다. 정확하게는 named pipe로 바꾸어 준다.
<(...)의 실행 결과는 named pipe(fifo)로 전달이 되고 이 파일(fifo)을 3번 fd로 열게 되는 것이다. 
결과적으로 read -u 3는 listAlbums의 실행 결과에서 한 줄씩 읽게 되는 것이다.

~~~ bash
#
# Delete albums interactively
# Usage:
#     deleteAlbums [albumName]
#     if albumName is null, delete all albums
deleteAlbums() {
    local albumName=$1

    # outer read from fd 3
    # inner read from fd 0 (stdin)
    exec 3< <(listAlbums ${albumName})
    IFS=$'\t'
    while read -r -u 3 id numphotos published title link _; do
        read -r -n 1 -s -p "Are you sure to delete '$title'(num of photos = $numphotos) ? (y/N/q) " decision
        if [[ $decision = 'q' ]];then
            echo "Quit"
            exit 0
        elif [[ $decision != 'y' ]];then
            echo "No"
            continue
        fi
        echo "Yes"

        "${cURL[@]}" \
            --header 'If-Match: *' \
            --request DELETE \
            "${link}"
    done
    IFS=$' \t\n'
    exec 3<&-
}
~~~
