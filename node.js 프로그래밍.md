## 준비 사항 
### 1. Node.js.20.x 설치 

<img width="869" height="699" alt="image" src="https://github.com/user-attachments/assets/16585620-c2d5-4280-8aa2-bf83de2e18c3" />

설치 URL : https://nodejs.org/ko/download

### 2. 작업용 폴더 생성
```
mkdir nodejs-labs
cd nodejs-labs
```

### 3. 프로젝트 초기화
```
npm init -y
```

---

### 실습 1 : Hello HTTP 서버 
```
npm init -y
npm install express
```

**코드(lab01-hello-server.js)**
```
// lab01-hello-server.js
const http = require("http");

const server = http.createServer((req, res) => {
  if (req.url === "/") {
    res.writeHead(200, { "Content-Type": "text/plain; charset=utf-8" });
    res.end("Hello, Node.js HTTP Server!");
  } else {
    res.writeHead(404, { "Content-Type": "text/plain; charset=utf-8" });
    res.end("Not Found");
  }
});

server.listen(3001, () => {
  console.log("🚀 Lab01 Hello server running on http://localhost:3001");
});
```

**실행**
```
node lab01-hello-server.js

curl -i http://localhost:3001/
```

### 실습 2 : Express 기본 라우팅 
**코드(lab02-express-basic.js)**
```
// lab02-express-basic.js
const express = require("express");
const app = express();
app.use(express.json());

app.get("/", (req, res) => res.send("Welcome to Lab02"));

app.get("/greet/:name", (req, res) => {
  res.json({ message: `Hello, ${req.params.name}!` });
});

app.post("/echo", (req, res) => {
  // 요청 바디를 그대로 반환
  res.json({ received: req.body });
});

app.listen(3002, () => console.log("🚀 Lab02 running on http://localhost:3002"));
```

**실행**
```
node lab02-express-basic.js

# GET 루트
curl http://localhost:3002/

# 경로 파라미터
curl http://localhost:3002/greet/Alice

# POST JSON
curl -X POST http://localhost:3002/echo -H "Content-Type: application/json" -d '{"foo":"bar"}'

# 또는 PowerShell POST JSON
Invoke-RestMethod -Uri http://localhost:3002/echo -Method POST -Headers @{ "Content-Type" = "application/json" } -Body '{"foo":"bar"}'
```

### 실습 3 : 미들웨어 이해(로깅 + 응답시간)
**코드(lab03-middleware.js)**
```
// lab03-middleware.js
const express = require("express");
const app = express();

app.use((req, res, next) => {
  req.startAt = Date.now();
  console.log(`[REQ] ${req.method} ${req.url}`);
  next();
});

app.get("/time", (req, res) => {
  const took = Date.now() - req.startAt;
  res.json({ message: "OK", tookMs: took });
});

// 글로벌 에러 핸들러 예시
app.use((err, req, res, next) => {
  console.error("Global Error:", err);
  res.status(500).json({ error: "Internal Server Error" });
});

app.listen(3003, () => console.log("🚀 Lab03 running on http://localhost:3003"));
```

**실행**
```
node lab03-middleware.js

curl http://localhost:3003/time
# 터미널에 [REQ] GET /time 로그 출력, 응답에 tookMs 표시
```

### 실습 4 : 정적 파일 제공 및 파일 구조 
#### 파일 구조 
```
lab04/
  public/
    index.html
  lab04-static.js
```

`public/index.html`
```
<!-- public/index.html -->
<!doctype html>
<html>
  <body>
    <h1>Lab04 Static File</h1>
    <p id="time"></p>
    <script>
      fetch("/api/now").then(r=>r.json()).then(j=>document.getElementById("time").innerText = j.now);
    </script>
  </body>
</html>
```

`lab04-static.js`
```
// lab04-static.js
const path = require("path");
const express = require("express");
const app = express();

app.use(express.static(path.join(__dirname, "public")));

app.get("/api/now", (req, res) => {
  res.json({ now: new Date().toISOString() });
});

app.listen(3004, () => console.log("🚀 Lab04 running on http://localhost:3004"));
```

**실행**
```
node lab04-static.js

curl http://localhost:3004/api/now
```

### 실습 5 : 쿼리 파라미터 및 폼 처리 
**코드(lab05-params-forms.js)**
```
// lab05-params-forms.js
const express = require("express");
const app = express();

// urlencoded 파싱 (폼)
app.use(express.urlencoded({ extended: false }));
app.use(express.json());

app.get("/search", (req, res) => {
  // 예: /search?q=node&page=2
  const { q = "", page = "1" } = req.query;
  res.json({ query: q, page: Number(page) });
});

app.post("/submit-form", (req, res) => {
  // form fields: name, comment
  res.json({ received: req.body });
});

app.listen(3005, () => console.log("🚀 Lab05 running on http://localhost:3005"));
```

**실행**
```
node lab05-params-forms.js

curl "http://localhost:3005/search?q=express&page=3"
curl -X POST http://localhost:3005/submit-form -d "name=Tom&comment=Hi"
# 또는 PowerShell:
Invoke-RestMethod -Uri http://localhost:3005/submit-form -Method POST -Body @{name="Tom";comment="Hi"}
```

### 실습 5 : 환경변수 사용 
#### 파일
`.env`
```
PORT=3006
GREETING=HelloFromEnv
```

**코드(lab06-env.js)**
```
// lab06-env.js
require("dotenv").config();
const express = require("express");
const app = express();

const PORT = process.env.PORT || 3006;
const GREETING = process.env.GREETING || "Hello";

app.get("/", (req, res) => res.send(`${GREETING} - port:${PORT}`));

app.listen(PORT, () => console.log(`🚀 Lab06 running on http://localhost:${PORT}`));
```

**실행**
```
node lab06-env.js

curl http://localhost:3006/
```

### 실습 7 : 파일 업로드
**코드(lab07-file-upload.js)**
```
// lab07-file-upload.js
const express = require("express");
const multer = require("multer");
const path = require("path");
const fs = require("fs");

const uploadDir = path.join(__dirname, "uploads");
if (!fs.existsSync(uploadDir)) fs.mkdirSync(uploadDir);

const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, uploadDir),
  filename: (req, file, cb) => cb(null, Date.now() + "-" + file.originalname)
});
const upload = multer({ storage });

const app = express();

app.post("/upload", upload.single("file"), (req, res) => {
  if (!req.file) return res.status(400).json({ error: "No file" });
  res.json({ filename: req.file.filename, path: `/uploads/${req.file.filename}` });
});

app.use("/uploads", express.static(uploadDir)); // 업로드 파일 제공

app.listen(3007, () => console.log("🚀 Lab07 running on http://localhost:3007"));

```

**실행**
```
node lab07-file-upload.js

curl -F "file=@./some-image.png" http://localhost:3007/upload
# 응답에 path가 반환되며 브라우저에서 http://localhost:3007/uploads/<파일명>으로 확인 가능

# 또는 PowerShell 
Invoke-WebRequest -Uri http://localhost:3007/upload -Method POST -Form @{ file = Get-Item "./some-image.png" }
```

### 실습 8 : 스트리밍 
**코드(lab08-stream-file.js)**
```
// lab08-stream-file.js
const express = require("express");
const fs = require("fs");
const path = require("path");
const app = express();

const bigFile = path.join(__dirname, "sample-large-file.bin");

// 샘플 대용량 파일 생성(없으면 생성)
if (!fs.existsSync(bigFile)) {
  // 10MB 임시 파일 생성 (실습용)
  const buf = Buffer.alloc(10 * 1024 * 1024, 0);
  fs.writeFileSync(bigFile, buf);
}

app.get("/download", (req, res) => {
  const stat = fs.statSync(bigFile);
  res.writeHead(200, {
    "Content-Type": "application/octet-stream",
    "Content-Length": stat.size,
    "Content-Disposition": 'attachment; filename="sample-large-file.bin"'
  });
  const readStream = fs.createReadStream(bigFile);
  readStream.pipe(res);
});

app.listen(3008, () => console.log("🚀 Lab08 running on http://localhost:3008"));
```

**실행**
```
node lab08-stream-file.js

curl -O http://localhost:3008/download
# 또는 PowerShell
Invoke-WebRequest -Uri http://localhost:3008/download -OutFile "download"
```

### 실습 9 : 타이머/스케줄
**코드(lab09-timer.js)**
```
// lab09-timer.js
const express = require("express");
const app = express();

let counter = 0;

// 5초마다 counter 증가 (실습용)
setInterval(() => {
  counter++;
  console.log(`[TIMER] counter=${counter} at ${new Date().toISOString()}`);
}, 5000);

app.get("/counter", (req, res) => {
  res.json({ counter, updatedAt: new Date().toISOString() });
});

app.listen(3009, () => console.log("🚀 Lab09 running on http://localhost:3009"));
```

**실행**
```
node lab09-timer.js

curl http://localhost:3009/counter
```

### 실습 10 : 단위 테스트 
**설치**
```
npm init -y
npm install express
npm install --save-dev jest supertest
# package.json에 test 스크립트 추가: "test": "jest"
```

**코드(lab10-app.js)**
```
// lab10-app.js
const express = require("express");
const app = express();
app.use(express.json());

app.get("/ping", (req, res) => res.json({ pong: true }));
app.post("/add", (req, res) => {
  const { a = 0, b = 0 } = req.body;
  res.json({ result: a + b });
});

module.exports = app;
```

**테스트(lab10-app.test.js)**
```
// lab10-app.test.js
const request = require("supertest");
const app = require("./lab10-app");

describe("Lab10 API", () => {
  test("GET /ping returns pong", async () => {
    const res = await request(app).get("/ping");
    expect(res.statusCode).toBe(200);
    expect(res.body).toEqual({ pong: true });
  });

  test("POST /add sums numbers", async () => {
    const res = await request(app).post("/add").send({ a: 2, b: 3 });
    expect(res.statusCode).toBe(200);
    expect(res.body.result).toBe(5);
  });
});
```

**실행**
```
npx jest --init
npm test
```





