# Bifrost MCP Gateway

여러 MCP 서버를 하나의 엔드포인트 뒤에 묶어, 사내 클라이언트(Claude Code 등)에
**단일 주소**로 노출하는 게이트웨이. [bifrost](https://github.com/maximhq/bifrost)를
사용한다. BigQuery는 "동작하는 예시" upstream 하나로 포함돼 있다.

```
[Claude Code 등]  ──►  bifrost (:8080)  ──►  toolbox (BigQuery MCP)
   클라이언트            게이트웨이              ├─ (추가) 다른 MCP 서버
                      웹 UI/인증/관측성         └─ ...
```

## 빠른 시작

```bash
cp .env.example .env          # 값 채우기 (BIGQUERY_PROJECT 등)
# BigQuery 예시를 쓸 경우에만: GCP 서비스 계정 키를 넣는다
#   secrets/gcp-sa.json  (BigQuery Data Viewer + Job User 권한)

docker compose up -d
docker compose logs -f bifrost
```

- 게이트웨이 / 웹 UI: http://localhost:8080
- MCP 도구는 bifrost가 부팅 시 upstream에 접속해 자동으로 discovery → sync 한다.

## MCP 서버 추가 / 교체 (핵심)

`data/config.json`의 `mcp.client_configs` 배열에 entry를 추가하면 끝이다.
세 가지 transport를 지원한다.

```jsonc
{
  "mcp": {
    "client_configs": [
      // 1) 원격 HTTP MCP 서버 (예시의 BigQuery toolbox가 이 방식)
      {
        "name": "bigquery",
        "connection_type": "http",
        "connection_string": "http://toolbox:5000/mcp",
        "is_ping_available": true,
        "tools_to_execute": ["*"]          // ["*"]=전체, []=차단, 특정 도구만 화이트리스트 가능
      },

      // 2) 로컬 프로세스 (STDIO) — npx/CLI 도구
      {
        "name": "filesystem",
        "connection_type": "stdio",
        "stdio_config": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data"],
          "envs": ["HOME", "PATH"]
        },
        "tools_to_execute": ["*"]
      },

      // 3) SSE 스트리밍 원격 서버
      {
        "name": "some-remote",
        "connection_type": "sse",
        "connection_string": "https://mcp.example.com/sse",
        "is_ping_available": true,
        "tools_to_execute": []
      }
    ]
  }
}
```

> STDIO 방식을 Docker bifrost 컨테이너에서 쓰려면 컨테이너 안에 해당 런타임(node 등)이
> 있어야 한다. 멀티유저 운영에서는 각 MCP 서버를 **별도 서비스(HTTP)** 로 띄우고
> bifrost가 `http://<service>:<port>/mcp`로 붙는 패턴(예시의 toolbox 방식)이 가장 견고하다.

새 MCP 서버를 docker-compose 서비스로 추가하는 경우, `docker-compose.yml`의 `toolbox`
블록을 템플릿 삼아 서비스를 하나 더 정의하고 `config.json`의 entry만 맞추면 된다.

## 인증 (HTTP/SSE upstream)

`connection_type`이 http/sse면 upstream 인증을 붙일 수 있다:
`none`, `headers`, `oauth`, `per_user_oauth`, `per_user_headers`. 사내 사용자별 권한
분리가 필요하면 `per_user_*` 모드를 검토한다.

## BigQuery 예시 자격증명

`secrets/gcp-sa.json`에 서비스 계정 키를 두고 `.env`의 `BIGQUERY_PROJECT`를 채운다.
필요 권한(최소): `roles/bigquery.dataViewer`, `roles/bigquery.jobUser`.
BigQuery가 필요 없으면 `docker-compose.yml`의 `toolbox` 서비스와 `config.json`의
`bigquery` entry를 지우면 된다.

## 참고

- bifrost: https://github.com/maximhq/bifrost · 문서 https://docs.getbifrost.ai
- MCP 연결 설정: https://docs.getbifrost.ai/mcp/connecting-to-servers
- BigQuery MCP (Google MCP Toolbox): https://github.com/googleapis/mcp-toolbox
