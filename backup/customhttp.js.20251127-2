#!/usr/bin/env node

// customhttp.js
// 서버(-s): 파일/폴더를 다운로드용으로 제공
// 클라이언트(-r): 서버에서 파일 다운로드 (+옵션에 따라 압축해제, 진행률, 해시검증 등)

const express = require('express');
const fs = require('fs');
const path = require('path');
const axios = require('axios');
const crypto = require('crypto');
const os = require('os');
const zlib = require('zlib');
const tar = require('tar');

// -------------------------
// 옵션 파싱
// -------------------------
const args = process.argv.slice(2);

const hasSend = args.includes('-s');
const hasRecv = args.includes('-r');
const fIndex = args.indexOf('-f');
const pIndex = args.indexOf('-p');
const hIndex = args.indexOf('-h');
const hasB = args.includes('-b');            // progress bar
const hasNoRelease = args.includes('-norelease');
const hasC = args.includes('-c');            // chunk(range) 다운로드

const hasTG = args.includes('-tg');
const hasT = args.includes('-t');
const hasG = args.includes('-g');

const targetPath = fIndex !== -1 ? args[fIndex + 1] : null;
const port = pIndex !== -1 ? parseInt(args[pIndex + 1]) : 80;
const host = hIndex !== -1 ? args[hIndex + 1] : 'localhost';

// 패킹 모드 결정: none | tar | gz | targz
let packMode = 'none';
if (hasTG || (hasT && hasG)) {
    packMode = 'targz';
} else if (hasT) {
    packMode = 'tar';
} else if (hasG) {
    packMode = 'gz';
}

// -------------------------
// 공용 유틸
// -------------------------
function fileExists(p) {
    try {
        fs.accessSync(p, fs.constants.F_OK);
        return true;
    } catch {
        return false;
    }
}

function ensureDir(dirPath) {
    if (!fileExists(dirPath)) {
        fs.mkdirSync(dirPath, { recursive: true });
    }
}

function computeSha256(filePath) {
    return new Promise((resolve, reject) => {
        const hash = crypto.createHash('sha256');
        const rs = fs.createReadStream(filePath);
        rs.on('error', reject);
        rs.on('data', chunk => hash.update(chunk));
        rs.on('end', () => resolve(hash.digest('hex')));
    });
}

function drawProgressBar(downloaded, total) {
    if (!total || total <= 0) return;
    const ratio = downloaded / total;
    const percent = Math.floor(ratio * 100);
    const width = 40;
    const filled = Math.floor(width * ratio);
    const bar = '█'.repeat(filled) + '-'.repeat(width - filled);
    process.stdout.write(`\r[${bar}] ${percent}% (${downloaded}/${total} bytes)`);
    if (downloaded >= total) {
        process.stdout.write('\n');
    }
}

// 자동 압축 해제
async function autoExtractIfNeeded(filePath, noRelease) {
    if (noRelease) {
        console.log(`������ -norelease 옵션으로 인해 압축 해제하지 않습니다: ${filePath}`);
        return;
    }

    const dir = path.dirname(filePath);
    const base = path.basename(filePath);

    if (base.endsWith('.tar.gz') || base.endsWith('.tgz')) {
        console.log(`������ tar.gz 해제 중: ${filePath}`);
        await tar.x({
            file: filePath,
            cwd: dir,
            gzip: true
        });
        fs.unlinkSync(filePath);
        console.log(`✅ 해제 완료 (tar.gz): ${dir}`);
    } else if (base.endsWith('.tar')) {
        console.log(`������ tar 해제 중: ${filePath}`);
        await tar.x({
            file: filePath,
            cwd: dir,
            gzip: false
        });
        fs.unlinkSync(filePath);
        console.log(`✅ 해제 완료 (tar): ${dir}`);
    } else if (base.endsWith('.gz')) {
        console.log(`������ gz 해제 중: ${filePath}`);
        const destPath = filePath.replace(/\.gz$/, '');
        await new Promise((resolve, reject) => {
            const rs = fs.createReadStream(filePath);
            const ws = fs.createWriteStream(destPath);
            const gunzip = zlib.createGunzip();
            rs.pipe(gunzip).pipe(ws);
            ws.on('finish', resolve);
            ws.on('error', reject);
            rs.on('error', reject);
            gunzip.on('error', reject);
        });
        fs.unlinkSync(filePath);
        console.log(`✅ 해제 완료 (gz): ${destPath}`);
    } else {
        // 일반 파일: 아무것도 안 함
    }
}

// -------------------------
// 패킹 파일 준비 (서버 측)
// -------------------------
async function prepareSendFile(inputPath, packMode) {
    const stat = fs.statSync(inputPath);
    const isDir = stat.isDirectory();

    // 파일 & no pack → 원본 그대로
    if (!isDir && packMode === 'none') {
        return {
            filePath: inputPath,
            fileName: path.basename(inputPath),
            cleanup: false
        };
    }

    const baseName = path.basename(inputPath);
    const tmpDir = os.tmpdir();
    let outPath;

    if (isDir) {
        // 폴더
        if (packMode === 'none' || packMode === 'tar') {
            // tar (무압축)
            outPath = path.join(tmpDir, `${baseName}.tar`);
            console.log(`������ 폴더 tar 생성 중: ${outPath}`);
            await tar.c(
                {
                    file: outPath,
                    cwd: path.dirname(inputPath),
                    gzip: false
                },
                [baseName]
            );
        } else if (packMode === 'targz' || packMode === 'gz') {
            // tar.gz
            outPath = path.join(tmpDir, `${baseName}.tar.gz`);
            console.log(`������ 폴더 tar.gz 생성 중: ${outPath}`);
            await tar.c(
                {
                    file: outPath,
                    cwd: path.dirname(inputPath),
                    gzip: true
                },
                [baseName]
            );
        }
    } else {
        // 단일 파일
        if (packMode === 'tar') {
            outPath = path.join(tmpDir, `${baseName}.tar`);
            console.log(`������ 파일 tar 생성 중: ${outPath}`);
            await tar.c(
                {
                    file: outPath,
                    cwd: path.dirname(inputPath),
                    gzip: false
                },
                [baseName]
            );
        } else if (packMode === 'targz') {
            outPath = path.join(tmpDir, `${baseName}.tar.gz`);
            console.log(`������ 파일 tar.gz 생성 중: ${outPath}`);
            await tar.c(
                {
                    file: outPath,
                    cwd: path.dirname(inputPath),
                    gzip: true
                },
                [baseName]
            );
        } else if (packMode === 'gz') {
            outPath = path.join(tmpDir, `${baseName}.gz`);
            console.log(`������ 파일 gz 생성 중: ${outPath}`);
            await new Promise((resolve, reject) => {
                const rs = fs.createReadStream(inputPath);
                const ws = fs.createWriteStream(outPath);
                const gz = zlib.createGzip();
                rs.pipe(gz).pipe(ws);
                ws.on('finish', resolve);
                ws.on('error', reject);
                rs.on('error', reject);
                gz.on('error', reject);
            });
        } else {
            // packMode === 'none' 이지만 dir은 위에서 이미 처리됨, 파일인데 여기 들어오면 이상
            return {
                filePath: inputPath,
                fileName: baseName,
                cleanup: false
            };
        }
    }

    return {
        filePath: outPath,
        fileName: path.basename(outPath),
        cleanup: true
    };
}

// -------------------------
// 서버 모드
// -------------------------
async function startServer() {
    if (!targetPath) {
        console.error('Error: -s 모드에서는 -f <보낼 파일/폴더> 가 필요합니다.');
        process.exit(1);
    }

    const resolved = path.resolve(targetPath);

    if (!fileExists(resolved)) {
        console.error(`Error: 대상 경로가 존재하지 않습니다 → ${resolved}`);
        process.exit(1);
    }

    console.log(`������ 원본 경로: ${resolved}`);
    console.log(`������ 패킹 모드: ${packMode}`);

    // 보낼 파일 준비
    const prep = await prepareSendFile(resolved, packMode);
    const sendFilePath = prep.filePath;
    const sendFileName = prep.fileName;
    const cleanup = prep.cleanup;

    console.log(`������ 전송 대상 파일: ${sendFilePath} (name: ${sendFileName})`);

    const stat = fs.statSync(sendFilePath);
    const totalSize = stat.size;
    const sha256 = await computeSha256(sendFilePath);

    console.log(`������ 파일 크기: ${totalSize} bytes`);
    console.log(`������ SHA-256: ${sha256}`);

    const app = express();

    function setCommonHeaders(res, isPartial, start, end) {
        res.setHeader('Content-Type', 'application/octet-stream');
        res.setHeader(
            'Content-Disposition',
            `attachment; filename="${sendFileName}"`
        );
        res.setHeader('X-File-Sha256', sha256);
        if (!isPartial) {
            res.setHeader('Content-Length', totalSize);
        } else {
            res.setHeader('Accept-Ranges', 'bytes');
            res.setHeader('Content-Range', `bytes ${start}-${end}/${totalSize}`);
            res.setHeader('Content-Length', end - start + 1);
        }
    }

    app.get('/download', (req, res) => {
        const range = req.headers.range;

        if (range) {
            // Range 요청 처리 (206 Partial Content)
            const match = range.match(/bytes=(\d+)-(\d+)?/);
            if (!match) {
                res.status(416).send('Invalid Range');
                return;
            }
            const start = parseInt(match[1], 10);
            let end = match[2] ? parseInt(match[2], 10) : totalSize - 1;

            if (start >= totalSize || end >= totalSize) {
                res.status(416).send('Requested Range Not Satisfiable');
                return;
            }

            setCommonHeaders(res, true, start, end);
            res.status(206);

            const rs = fs.createReadStream(sendFilePath, { start, end });
            rs.on('error', (err) => {
                console.error('ReadStream Error:', err.message);
                res.destroy(err);
            });
            rs.pipe(res);
        } else {
            // 전체 파일 전송
            setCommonHeaders(res, false);
            res.status(200);
            const rs = fs.createReadStream(sendFilePath);
            rs.on('error', (err) => {
                console.error('ReadStream Error:', err.message);
                res.destroy(err);
            });
            rs.pipe(res);
        }
    });

    app.listen(port, host, () => {
        console.log('������ customhttp 서버 모드');
        console.log(`➡ Host: ${host}`);
        console.log(`➡ Port: ${port}`);
        console.log(`➡ URL: http://${host}:${port}/download`);
        console.log(`➡ PackMode: ${packMode}  (tar/gz/targz/none)`);
    });

    // cleanup 파일은 서버가 끝날 때 정리하는 게 맞지만,
    // 여기서는 서버가 계속 떠 있을 것이므로 별도 삭제는 하지 않음.
}

// -------------------------
// 클라이언트 모드: 단일 요청 다운로드
// -------------------------
async function simpleDownload(url, saveDir, showBar, noRelease) {
    const res = await axios.get(url, { responseType: 'stream' });

    // 파일명 추출
    let fileName = 'downloaded.bin';
    const disp = res.headers['content-disposition'];
    if (disp && disp.includes('filename=')) {
        fileName = disp.split('filename=')[1].replace(/"/g, '');
    }

    const filePath = path.join(saveDir, fileName);
    const total = parseInt(res.headers['content-length'] || '0', 10);
    const serverHash = res.headers['x-file-sha256'];

    console.log(`������ 저장 경로: ${filePath}`);
    if (total) console.log(`������ 예상 크기: ${total} bytes`);

    let downloaded = 0;
    const ws = fs.createWriteStream(filePath);

    await new Promise((resolve, reject) => {
        res.data.on('data', (chunk) => {
            downloaded += chunk.length;
            if (showBar && total) {
                drawProgressBar(downloaded, total);
            }
        });
        res.data.on('error', reject);
        ws.on('error', reject);
        ws.on('finish', resolve);

        res.data.pipe(ws);
    });

    console.log(`✅ 다운로드 완료: ${filePath}`);

    // 해시 검증
    if (serverHash) {
        const localHash = await computeSha256(filePath);
        if (localHash === serverHash) {
            console.log(`������ 해시 일치 (SHA-256): ${localHash}`);
        } else {
            console.warn(`⚠ 해시 불일치!\n  server: ${serverHash}\n  local : ${localHash}`);
        }
    }

    // 자동 해제
    await autoExtractIfNeeded(filePath, noRelease);
}

// -------------------------
// 클라이언트 모드: Range 기반 chunk 다운로드 (-c)
// -------------------------
async function rangeDownload(url, saveDir, showBar, noRelease) {
    const CHUNK_SIZE = 10 * 1024 * 1024; // 10MB

    // 첫 번째 Range 요청으로 파일명/크기/해시 획득
    const firstEnd = CHUNK_SIZE - 1;
    const firstRes = await axios.get(url, {
        responseType: 'arraybuffer',
        headers: {
            Range: `bytes=0-${firstEnd}`
        },
        validateStatus: (status) => status === 206 || status === 200
    });

    // 파일명
    let fileName = 'downloaded.bin';
    const disp = firstRes.headers['content-disposition'];
    if (disp && disp.includes('filename=')) {
        fileName = disp.split('filename=')[1].replace(/"/g, '');
    }

    const serverHash = firstRes.headers['x-file-sha256'];

    // total 크기
    let total = 0;
    const cr = firstRes.headers['content-range'];
    if (cr && cr.includes('/')) {
        total = parseInt(cr.split('/')[1], 10);
    } else {
        total = firstRes.data.byteLength;
    }

    const filePath = path.join(saveDir, fileName);
    console.log(`������ 저장 경로: ${filePath}`);
    console.log(`������ 예상 크기: ${total} bytes`);

    const fd = fs.openSync(filePath, 'w');
    let downloaded = 0;

    // 첫 chunk 쓰기
    let buffer = Buffer.from(firstRes.data);
    fs.writeSync(fd, buffer, 0, buffer.length, 0);
    downloaded += buffer.length;
    if (showBar && total) {
        drawProgressBar(downloaded, total);
    }

    // 나머지 chunk 순차 다운로드
    let position = buffer.length;
    while (position < total) {
        const start = position;
        const end = Math.min(start + CHUNK_SIZE - 1, total - 1);

        const res = await axios.get(url, {
            responseType: 'arraybuffer',
            headers: {
                Range: `bytes=${start}-${end}`
            },
            validateStatus: (status) => status === 206 || status === 200
        });

        buffer = Buffer.from(res.data);
        fs.writeSync(fd, buffer, 0, buffer.length, start);
        position += buffer.length;
        downloaded += buffer.length;

        if (showBar && total) {
            drawProgressBar(downloaded, total);
        }
    }

    fs.closeSync(fd);
    console.log(`✅ 다운로드 완료: ${filePath}`);

    // 해시 검증
    if (serverHash) {
        const localHash = await computeSha256(filePath);
        if (localHash === serverHash) {
            console.log(`������ 해시 일치 (SHA-256): ${localHash}`);
        } else {
            console.warn(`⚠ 해시 불일치!\n  server: ${serverHash}\n  local : ${localHash}`);
        }
    }

    // 자동 해제
    await autoExtractIfNeeded(filePath, noRelease);
}

// -------------------------
// 클라이언트 모드 시작
// -------------------------
async function startClient() {
    const saveDir = targetPath ? path.resolve(targetPath) : process.cwd();
    ensureDir(saveDir);

    const url = `http://${host}:${port}/download`;
    console.log(`������ 서버: ${url}`);
    console.log(`������ 저장 디렉터리: ${saveDir}`);
    console.log(`������ progress bar: ${hasB ? 'ON (-b)' : 'OFF'}`);
    console.log(`������ auto release: ${hasNoRelease ? 'OFF (-norelease)' : 'ON'}`);
    console.log(`������ chunk(range) 모드: ${hasC ? 'ON (-c)' : 'OFF'}`);

    if (hasC) {
        await rangeDownload(url, saveDir, hasB, hasNoRelease);
    } else {
        await simpleDownload(url, saveDir, hasB, hasNoRelease);
    }
}

// -------------------------
// 엔트리 포인트
// -------------------------
(async () => {
    if (hasSend) {
        await startServer();
    } else if (hasRecv) {
        await startClient();
    } else {
        console.log(`
사용법:
  서버 모드(보낼 때):
    customhttp -s -f <파일 또는 폴더 경로> [-t] [-g] [-tg] [-p 포트] [-h 호스트]

      -t      : tar로 묶기 (압축 없음)
      -g      : gzip 압축 (파일이면 .gz, 폴더면 tar.gz 역할)
      -tg     : tar.gz (tar 후 gzip)
      (없음)  : 파일은 그대로, 폴더는 tar(무압축)로 묶어서 전송

  클라이언트 모드(받을 때):
    customhttp -r [-f 저장폴더] [-b] [-c] [-norelease] [-p 포트] [-h 서버IP]

      -b          : 진행률(progress bar) 표시
      -c          : Range 기반 chunk 다운로드 사용
      -norelease  : tar/gz/tar.gz 자동 해제하지 않고 그대로 보존

예:
  서버:
    customhttp -s -f ./data
    customhttp -s -f ./data -t
    customhttp -s -f ./data -tg -p 9000 -h 0.0.0.0

  클라이언트:
    customhttp -r -f ./downloads
    customhttp -r -f ./downloads -b
    customhttp -r -f ./downloads -b -c -norelease
`);
    }
})();

