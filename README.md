# input
<h2>문제 소스와 <br></h2>

input 문제에서는 실행파일을 통해서 풀기가 어렵기 때문에 파이썬 pwn 모듈을 이용하여 페이로드를 작성하여
문제를 해결하는데 접근하려고 한다.<br>
먼저 코드부터 보면 <br>

<img width="600" alt="input1" src="https://user-images.githubusercontent.com/107084512/206091654-1d0ce44e-5f99-4e92-a85d-81d8368361c3.png">
<p>첫 번째로 stage1을 통과하기 위해서는 들어가는 인자의 개수가 100개에 argv['A']를 해석하면 argv[65]
(ASCII코드에 기반) 똑같이 argv['B']를 해석하면 argv[66]가 된다. 
strcmp 함수: strcmp(x,y) x와 y라는 문자열을 비교한다. x와 y가 같다면 0을 반환하고, 같지 않다면 음수나 양수를 반환하는 함수이다.
즉 strcmp(argv['A'],"\x00")은 65번째 들어오는 인자값이 \x00이 되어야한다는 것이다.(argv['B']도 동일)</p>

<p>두 번째로 buf라는 배열을 선언한다-> buf={0,0,0,0,0}<br>
read함수: read(fd,읽어들일 버퍼, 버퍼의 크기)<br>
파일 디스크립터:<br>
0-> stdin(입력 스트림)<br>
1-> stdout(출력 스트림)<br>
2-> stderr(에러 출력 스트림)<br>
나머지 (3 이후부터의 숫자) : 파일의 책갈피같은 역할을 한다.<br></p>
<img width="407" alt="표준 스트림 " src="https://user-images.githubusercontent.com/107084512/206097147-354e915f-676d-494a-b644-116403fb66ae.png">
<p>memcmp 함수 : memcmp(src1,src2,size)<br>
src1: 비교할 값1<br>
src2: 비교할 값2<br>
size: 비교할 크기<br>
*정확히 말하자면 주소값을 비교하는 것이 아니라, 주소에 든 문자열을 비교하는 것이다.
<img width="600" alt="예시" src="https://user-images.githubusercontent.com/107084512/206097607-88cc8724-5746-4bea-adae-a856631ce40b.png">

이와 같이 구조체가 같은 문자열을 가지고 있기 때문에 0을 반환한다. 만약 같지 않을 때는 1 or -1을 반환한다.</p>

<p>세 번째는 예시로 먼저 확인한다.
<img width="600" alt="env예시" src="https://user-images.githubusercontent.com/107084512/206097981-5675ac4e-0a44-43d2-954b-1d5f0f91493c.png">
<img width="600" alt="env예시 결과" src="https://user-images.githubusercontent.com/107084512/206098146-d0425e14-03ac-40d6-814e-fb5db0255541.png"><br>
위의 예시를 보면 <br>
getenv()함수에 인자값을 넣으면 해당 인자의 경로를 보여준다.<br>
putenv()함수의 형식은 putenv("PATH:c:~") 형태로 인자에 문자열 형식으로 환경 변수의 이름과 경로를 함께 
적어줘야한다.<br>
이것을 기반으로 문제 코드를 보면 "\xde\xad\xbe\xef"라는 환경 변수에 "\xca\xfe\xba\xbe" 값이 들어가야한다는 것을 알 수 있다.<br></p>

<img width="600" alt="input2" src="https://user-images.githubusercontent.com/107084512/206091674-7a611324-f0fd-4406-b4da-ba4434e3e3b3.png">
<p>네 번째로 
fopen 함수 FILE* fopen(const char*a1,const char*a2)
<br>a1: 파일의 이름
a2: 파일 처리의 종류
fread 함수 (*ptr, size, count, FILE * stream)
<img width="628" alt="파일 처리 종류" src="https://user-images.githubusercontent.com/107084512/206098589-674059a8-12df-4c9e-9aeb-6a46985105d4.png">
첫 번째 줄부터 해석하면 <br>
FILE* fp = fopen("\x0a","r");- => \x0a라는 파일을 읽기모드로 연다<br>
if(!fp) return 0; => 파일 정상적으로 열리지않으면 0을 반환하면서 프로그램이 종료된다.<br>
if(fread(buf,4,1,fp)!=1) return 0; => fp에서 4만큼을 읽고 buf에 저장하고 정상적으로 실행되지 않으면 0을 반환하면서 프로그램 종료된다. 정확히는 4byte가 1개인 buf라는 메모리를 할당하고 거기에 읽은 fp를 저장한다.<br>
그래서 예시에 나오는 fread는 fp에서 총 4*1=4byte 만큼을 읽어서 buf 에 저장한다.<br>
if(memcmp(buf,"\x00\x00\x00\x00",4)) return 0; => buf주소에 있는 문자열과 "\x00\x00\x00\x00"을 4byte만큼 비교한다. 두 값이 같아야 문제를 해결할 수 있음<br></p>

<img width="600" alt="input3" src="https://user-images.githubusercontent.com/107084512/206091684-a5e826b3-5a3c-4141-8cde-384d42c44302.png">
우선 소켓 통신의 흐름을 간단하게 보자면
<img width="440" alt="소켓 통신 과정" src="https://user-images.githubusercontent.com/107084512/206098967-90b09ccc-6bd5-4505-8d73-abc8d0ef12df.png">

<br>마지막으로 네트워크와 관련된 소켓 생성 함수들을 다룬다.
<br>코드를 처음부터 해석하자면
<br>1.int sd,cd; => sd,cd라는 변수 4byte 만큼 선언
<br>2.struct sockaddr_in saddr, caddr;-> 여기서부터는 처음보는 구조체 개념으로 들어가는데 우선 sockaddr_in 라는 이름을 가진 구조체와 그 안에는 sin_family,sin_port 들과 in_addr, sin_addr이라는 구조체를 포함하고 있다.
<br>sin_family : 항상 AF_INET을 설정한다.
<br>sin_port : 포트번호를 가진다(bytes). 포트번호는 0~65535의 범위를 갖는 숫자 값이다. 이 변수에 저장되는 값은 network byte order이어야 한다.
<br>sin_addr : 호스트 ip 주소이다.
<br>3.  htons() 함수 : 2bytes 데이터를 network byte order로 변경한다.
<br>4. atoi() 함수 : 문자열을 정수 타입으로 변경한다. 오직 정수값들만 다룬다. 인자로 들어온 문자열을 앞ㅇ서부터 어서 "공백" 이나 "숫자가 아닌 문자"가 올 때까지 숫자로 변환해주는 원리의 함수이다.
<br>*5. 여기서 argv['C'] 우리는 여기서 집중해야한다. 직접으로 코드에 영향을 줄 수 있는 부분이라서 본인이 주는 
입력값이 해당 코드에서 어떤 함수들을 통해 관통하는지 알아야한다. 해석하면 우리는 포트번호에 영향을 줄 것이다.
<br>6. bind() 함수 : 이 함수 이전까지는 비어있는 소켓을 생성한 것 뿐이고 bind함수를 통해서 이전에 선언한 
포트번호, 호스트 ip 주소, AF_INET 들을 비어있는 소켓 sd에 묶어주는 것이다. 만약 오류가 생긴다면 0을 반환하고 프로그램을 종료한다.
<br>bind(int sockfd, struct sockaddr* addr, socklen_t addrlen);
<br>sockfd : socket() 함수를 통해 배정받은 디스크립터 번호. 이 문제에서는 sd이다.
<br>*addr : IP주소와 PORT 번호를 지정한 sockaddr_in 구조체. 이 문제에서는 (struct sockaddr*)&saddr
<br>addrlen : 주소정보를 담은 변수의 길이. sizeof(saddr)
<br>7.listen () 함수: 
<br>listen(int sock, int backlog); 
<br>sock : 소켓 디스크립터 번호
<br>backlog : 연결요청을 대기하는 큐의 크기**
<br>8. accept()함수 : 
<br>accept(int sock, struct sockaddr*addr, socklen_t *addrlen);
<br>sock : 서버 소켓의 디스크립터 번호
<br>addr : 대기 큐를 참조해 얻은 클라이언트의 주소정보
<br>addrlen : addr 변수의 크기
<br>9. if(cd<0)~ : 0보다 작으면 client socket이 정상적으로 만들어지지 않은 것이기 때문에 0을 반환하고 프로그램을종료한다.
<br>10. if(recv(cd,buf,4,0)!=4) return 0; <-> send 함수 [send와 recv를 통해서 통신한다]
<br>문법 : recv(int sockfd, void * buff, size_t len, int flags)
<br><인자>
<br>int sockfd : 소켓 디스크립터
<br>void *buff : 수신할 버퍼 포인터
<br>size_t len : 버퍼의 바이트 단위 길이
<br>int flags : 아래와 같은 옵션을 사용할 수 있습니다.
<br>11. if(memcmp(buf,"\xde\xad\xbe\xef",4))return 0; 
수신받은 데이터의 4byte가 \xde\xad\xbe\xef 가 되야한다.


