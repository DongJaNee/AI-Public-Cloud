# Claude.ai MCP 설정
## 사전 설치 
### 1. Git install  
- https://git-scm.com/install/windows

### 2. Node.js install
- https://nodejs.org/ko

### 3. Claude Desktop install 
- https://claude.com/download'

---
## MCP 서버 설치 
### 1. AWS MCP 
1) 저장소 클론 
```
git clone https://github.com/RafalWilinski/aws-mcp
cd aws-mcp
```

2) 의존성 설치
```
npm install
# 또는 pnpm이 설치되어 있다면
pnpm install
```

<img width="1028" height="438" alt="image" src="https://github.com/user-attachments/assets/bf2796ab-f0d1-4c1f-8319-7bbd2bfef954" />

<br>
<br>


3) claude_desktop_config.json 설정 (C:\Users\%UserName\AppData\Roaming\claude_desktop_config.json)
```
{
  "mcpServers": {
    "aws": {
      "command": "C:\\Program Files\\nodejs\\node.exe",
      "args": [
        "C:/projects/aws-mcp/node_modules/tsx/dist/cli.mjs",
        "C:/projects/aws-mcp/index.ts"
      ],
      "env": {
        "AWS_ACCESS_KEY_ID": "YOUR_ACCESS_KEY",
        "AWS_SECRET_ACCESS_KEY": "YOUR_SECRET_KEY",
        "AWS_REGION": "ap-northeast-2",
        "APPDATA": "C:\\Users\\username\\AppData\\Roaming"
      }
    }
  }
}
```
<img width="1112" height="585" alt="image" src="https://github.com/user-attachments/assets/dae84215-d0c8-4fea-be5f-5327f35dc343" />

---


<img width="721" height="444" alt="image" src="https://github.com/user-attachments/assets/f59f7439-cd9d-4815-8ca0-eea971e816ac" />

---
### Filesystem MCP
#### Claude Desktop 연동 설정 
`cluade_destktop_config.json` 파일에 내용 추가 
```
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\Admin\\Documents",
        "C:\\Users\\Admin\\Desktop",
        "C:\\Projects"
      ]
    }
```

---
### 통합 설정 예시 (AWS, Filesystem, notion)
`cluade_destktop_config.json`
```

{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@suekou/mcp-notion-server"],
      "env": {
        "NOTION_API_TOKEN": "ntn_Y38328568259Blu8NvS1dLCcQX4CVZNSBzzYqEXPILD8QI",
        "APPDATA": "C:\\Users\\Admin\\AppData\\Roaming"
      }
    },
    "aws": {
      "command": "C:\\Program Files\\nodejs\\node.exe",
      "args": [
        "C:/projects/aws-mcp/node_modules/tsx/dist/cli.mjs",
        "C:/projects/aws-mcp/index.ts"
      ],
      "env": {
        "AWS_ACCESS_KEY_ID": "YOUR_ACCESS_KEY",
        "AWS_SECRET_ACCESS_KEY": "YOUR_SECRET_KEY",
        "AWS_REGION": "ap-northeast-2",
        "APPDATA": "C:\\Users\\Admin\\AppData\\Roaming"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\Admin\\Documents",
        "C:\\Users\\Admin\\Desktop",
        "C:\\Projects"
      ]
    }
  }
}
```

---
#### 설치 후 재시작
1. Claude Desktop 완전 종료
2. Claude Desktop 재시작

#### 확인 

<img width="739" height="766" alt="image" src="https://github.com/user-attachments/assets/f9c52350-37c7-410c-8959-67ef365c0668" />
