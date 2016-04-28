---
layout: post
title:  "man bash by examples"
date:   2016-04-28 00:10:00 +0900
categories: bash
---
man bash의 내용 순서대로 정리를 합니다. Bash 4.3.xx 기준이기 때문에 Mac OS X 3.2.xx에서는 동작하지 않는 명령도 존재합니다.

# OPTIONS
* -c

      $ bash -c 'echo $0 $1' command arg1
      command arg1

# SHELL GRAMMAR 
* Simple commands
    - 환경 변수 설정 및 리디렉션까지를 포함한다.
    - `;` `;;` `&` `&&` `||` `|` `|&` `(` `)` `<newline>`으로 다른 command와 구분이 된다.

          $ JEKYLL_ENV=production jekyll build >output

* Pipelines
    - 여러 command를 pipe로 연결해서 실행할 수 있다.
    - exit status는 마지막 command의 exit status
        * 이 경우에는 도중에 에러가 발생하면 알 수 없으므로 pipefail 옵션을 이용해서 모두 0을 리턴한 경우에만 0을 리턴하도록 할 수 있다.
    - `time [-p]`을 앞에 붙이면 전체 실행 시간을 표시해 준다.
    - `2>&1 |` 와 같이 리디렉션이 함께 있을 경우에는 pipe에 의한 fd의 변경이 먼저 일어나고 그 후에 리디렉션이 일어난다.
    - `2>&1 |` 는 `|&` 와 같다.

          # 각 command에 대해서 stderr를 별도의 파일로 저장
          $ time ls 2>ls.error | sort 2>sort.error | uniq 2>uniq.error

          # ls의 stdout이 아닌 stderr를 pipe로 전달하기. fd 3은 1, 2 번의 swap에 사용하기 위한 임시 fd
          $ ls -z 3>&1 1>&2 2>&3 | tee ls.error

* Lists
    - 여러 pipeline의 모임
    - `;`, `&`, `&&`, `||`로 개별 pipeline이 구별된다. `&`, `;`, `<newline>`으로 list는 끝난다.
    - `&&`, `||`의 우선 순위가 `;`, `&` 보다 높다.
    - `&&`, `||`의 우선 순위는 동등하다. `&&`가 더 높지 않다는 것에 주위하라.

          # &&이 아닌 || 부터 실행이 되기 때문에 1, 3이 출력된다.
          $ echo "1" || echo "2" && echo "3" & echo "4"
          [1] 36066
          4
          1
          3
          [1]+  Done                    echo "1" || echo "2" && echo "3"

          # background 실행을 한 줄에 실행하는 것이 가능하다.
          $ find dir1 | sort >list1 & find dir2 | sort >list2 &

* Compound Commands
    - (list)
        * subshell로 실행
    - { list; }
        * 세미콜론이나 newline으로 list가 종료
        * 공백이 있어야 한다.
    - ((expression))
        * ARITHMETIC EVALUATION
        * 0이 아닌 경우의 return status가 0이 된다.

              $ if ((1+1));then echo "OK";fi
              OK

        * `let "expression"` 과 동일

    - [[ expression ]]
        * CONDITIONAL EXPRESSIONS
        * Tilde expansion, parameter and variable expansion, arithmetic expansion, command substitution, process substitution, quote removal은 적용된다.
        * Pathname expansion은 적용되지 않는다.

              # $a, *에 대해서 pathname expansion이 발생해서 [ 를 이용하는 경우에는 에러 발생
              $ a='*'; if [ $a = * ];then echo "YES";fi
              -bash: [: too many arguments

              # $a, *를 모두 quote를 해서 해결
              $ a='*'; if [ "$a" = "*" ];then echo "YES";fi
              YES

              # pathname expansion이 발생하지 않는다.
              $ a='*'; if [[ $a = * ]];then echo "YES";fi
              YES

        * Word splitting은 적용되지 않는다.

              # $a에 대해서 word splitting이 발생해서 [ 를 이용하는 경우에는 에러 발생
              $ a="abc def"; if [ $a = "abc def" ];then echo "YES";fi
              -bash: [: too many arguments

              $ a="abc def"; if [ "$a" = "abc def" ];then echo "YES";fi
              YES

              # Word splitting이 발생하지 않는다.
              $ a="abc def"; if [[ $a = "abc def" ]];then echo "YES";fi
              YES

    - for name [ [ in [ word ... ] ] ; ] do list ; done
        * in word가 생략되면 positional parameter($1, $2, ...)에서 읽어온다.
              
              # 아래의 두 명령을 같다.
              for i         do echo $i;done
              for i in "$@";do echo $i;done

    - for (( expr1 ; expr2 ; expr3 )) ; do list ; done 
        * expr이 없는 경우에는 1로 처리
              
              $ for ((i = 0; i < 3; i++));do echo $i;done
              0
              1
              2

    - select name [ in word ] ; do list ; done
        * 주어진 목록이 화면에 번호와 함께 표시되고 사용자가 선택시 name이 값이 저장된다.
        * PS3에 원하는 프롬프트를 지정할 수 있다.
        * EOF(CTRL+D)로 끝낼 수 있다.

              $ PS3='Which file to delete? ';select i in *;do rm $i;done
              1) BashPlayground    5) file1        9) list2           13) readnum.sh  17) uniq.error  21) xad     25) xxx
              2) a b               6) file2       10) ls.error        14) run_get.sh  18) xaa         22) xae
              3) cut.sh            7) input.txt   11) no_in_word.sh   15) sort.error  19) xab         23) xaf
              4) diff.sh           8) list1       12) outfile         16) test.sh     20) xac         24) xag
              Which file to delete? 25
              rm: xxx: is a directory
              Which file to delete? 21
              Which file to delete? ^D


    - case word in [ [(] pattern [ \| pattern ] ... ) list ;; ] ... esac
        * `;;` 는 C switch 문의 break와 같은 역할을 한다. 버전 4.X 부터 `;;&`, `;&` 추가된 것으로 보임.
        * ;; 대신에 `;&`를 사용하면 C switch 문에서 break가 없는 것과 같다. (pass through)
        * `;;&` 를 사용하면 나머지 pattern에 대해서도 비교를 계속 수행한다.
        * word에는 다음의 expansion이 적용된다.
            - tilde expansion, parameter and  variable  expansion, arithmetic substitution, command substitution, process substitution and quote removal
        * pattern에도 다음의 expansion이 적용된다.
            - tilde expansion, parameter and variable expansion, arithmetic substitution, command substitution, and process substitution
        * 비교는 파일 이름 비교(glob pattern) 방법을 사용한다.

              $ cat case.sh
              #!/bin/bash
              filename=$1
              case $filename in
                  * )
                      echo "File $filename:"
                      ;;&
                  *jpg | *png | *bmp )
                      echo "image file"
                      ;;&
                  *.* )
                      echo "extension exist"
                      ;;&
              esac

              $ ./case.sh a.bmp
              File a.bmp:
              image file
              extension exist

              $ ./case.sh a.txt
              File a.txt:
              extension exist

              $ ./case.sh a
              File a:

    - if list; then list; [ elif list; then list; ] ... [ else list; ] fi

    - while list-1; do list-2; done
    - until list-1; do list-2; done
        * until은 while의 조건에 not 을 한 것과 같다.

              $ x=3;while ((x--));do echo $x;done
              2
              1
              0

              $ x=3;until ! ((x--));do echo $x;done
              2
              1
              0







