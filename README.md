# customhttp
사용법:<br />
  서버 모드(보낼 때):<br />
    customhttp -s -f <파일 또는 폴더 경로> [-t] [-g] [-tg] [-p 포트] [-h 호스트] <br />

      -t      : tar로 묶기 (압축 없음)
      -g      : gzip 압축 (파일이면 .gz, 폴더면 tar.gz 역할)
      -tg     : tar.gz (tar 후 gzip)
      (없음)  : 파일은 그대로, 폴더는 tar/gz 없이 폴더 구조 그대로 RAW 전송

  클라이언트 모드(받을 때):<br />
    customhttp -r [-f 저장폴더] [-b] [-c] [-norelease] [-p 포트] [-h 서버IP] <br />

      -b          : 진행률(progress bar) 표시
      -c          : Range 기반 chunk 다운로드 사용
      -norelease  : -t/-g/-tg 로 보낸 tar/gz/tar.gz 자동 해제하지 않고 그대로 보존

예:
  서버:<br />
    customhttp -s -f ./data <br />
    customhttp -s -f ./data -t <br />
    customhttp -s -f ./data -tg -p 8833 -h 0.0.0.0 <br />

  클라이언트: <br />
    customhttp -r -f ./downloads <br />
    customhttp -r -f ./downloads -b <br />
    customhttp -r -f ./downloads -b -c <br />
    customhttp -r -f ./downloads -b -c -norelease <br />

