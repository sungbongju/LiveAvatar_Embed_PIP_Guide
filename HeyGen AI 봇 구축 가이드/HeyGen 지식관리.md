# HeyGen 실시간 지식관리 가이드

> AI 아바타의 지식을 실시간으로 업데이트하고 관리하는 방법  
> 관련 문서: [[HeyGen Interactive Avatar 구축 가이드]]

---

## 📋 목차

1. [[#1. 지식관리 방식 비교]]
2. [[#2. System Prompt 방식]]
3. [[#3. 외부 파일 방식]]
4. [[#4. 데이터베이스 연동]]
5. [[#5. 벡터 DB + RAG]]
6. [[#6. 관리자 페이지 구축]]

---

## 1. 지식관리 방식 비교

| 방식 | 난이도 | 실시간 수정 | 대용량 | 추천 상황 |
|------|--------|------------|--------|----------|
| **System Prompt** | ⭐ | ❌ 배포 필요 | ❌ ~8K 토큰 | 소규모, 고정 지식 |
| **외부 파일** | ⭐⭐ | ✅ 파일만 수정 | ❌ ~50KB | 중규모, 가끔 수정 |
| **데이터베이스** | ⭐⭐⭐ | ✅ DB 수정 | ✅ 무제한 | 대규모, 자주 수정 |
| **벡터 DB + RAG** | ⭐⭐⭐⭐ | ✅ 자동 검색 | ✅ 무제한 | 대규모, 의미 검색 |

---

## 2. System Prompt 방식 (현재 사용 중)

### 구조
app/api/chat/route.ts  
└── SYSTEM_PROMPT 상수에 지식 포함


### 장점
- 구현이 가장 간단
- 추가 인프라 불필요
- 응답 속도 빠름

### 단점
- 지식 수정 시 코드 수정 + 재배포 필요
- 토큰 제한 (~8,000 토큰)

### 지식 수정 방법
1. `app/api/chat/route.ts` 파일 열기
2. `SYSTEM_PROMPT` 상수 내용 수정
3. GitHub에 커밋
4. Netlify 자동 재배포

---

## 3. 외부 파일 방식 ⭐추천

### 구조
public/  
└── knowledge/  
└── business_admin.json # 지식 파일  
app/api/chat/  
└── route.ts # 파일 읽어서 사용


### 구현

#### 3.1 지식 파일 생성
**파일**: `public/knowledge/business_admin.json`

```
json
{
  "version": "1.0",
  "updated": "2024-12-23",
  "topics": [
    {
      "title": "연구분야",
      "content": "경영학전공에서는 경영기획, 마케팅, 회계재무의 핵심 이론을 다룹니다..."
    },
    {
      "title": "취업률",
      "content": "경영학전공의 취업률은 88.7%로 전국 평균 대비 우수합니다..."
    }
  ]
}
```

#### 3.2 API Route 수정
**파일**: `app/api/chat/route.ts`

```import { NextRequest } from "next/server";
import OpenAI from "openai";
import fs from "fs";
import path from "path";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// 지식 파일 읽기
function loadKnowledge(): string {
  const filePath = path.join(process.cwd(), "public/knowledge/business_admin.json");
  const data = JSON.parse(fs.readFileSync(filePath, "utf-8"));
  
  return data.topics
    .map((t: { title: string; content: string }) => `### ${t.title}\n${t.content}`)
    .join("\n\n");
}

export async function POST(request: NextRequest) {
  const { message, history } = await request.json();
  
  const knowledge = loadKnowledge();
  
  const systemPrompt = `당신은 차의과학대학교 경영학전공의 AI 상담사입니다.
답변은 간결하게 2-3문장으로 해주세요.

## 경영학전공 지식
${knowledge}
`;

  const messages = [
    { role: "system" as const, content: systemPrompt },
    ...history,
    { role: "user" as const, content: message },
  ];

  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages,
    max_tokens: 300,
  });

  return new Response(
    JSON.stringify({ reply: response.choices[0]?.message?.content }),
    { headers: { "Content-Type": "application/json" } }
  );
}

```

### 지식 수정 방법

1. `public/knowledge/business_admin.json` 파일만 수정
    
2. GitHub에 커밋
    
3. Netlify 자동 재배포 (코드 변경 없이 데이터만 변경)


## 4. 데이터베이스 연동

|서비스|특징|무료 티어|
|---|---|---|
|**Supabase**|PostgreSQL, 실시간|500MB|
|**PlanetScale**|MySQL, 서버리스|5GB|
|**MongoDB Atlas**|NoSQL, 유연함|512MB|

### Supabase 구현 예시

#### 4.1 테이블 생성

```
CREATE TABLE knowledge (
  id SERIAL PRIMARY KEY,
  topic VARCHAR(100),
  content TEXT,
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 4.2 API Route

```
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_KEY!
);

async function loadKnowledge(): Promise<string> {
  const { data } = await supabase
    .from("knowledge")
    .select("topic, content");
  
  return data
    ?.map((row) => `### ${row.topic}\n${row.content}`)
    .join("\n\n") || "";
}
```

### 지식 수정 방법

1. Supabase 대시보드 접속
    
2. 테이블에서 직접 수정
    
3. **재배포 없이 즉시 반영!** ✅


## 5. 벡터 DB + RAG

> 대용량 지식을 의미 기반으로 검색하는 고급 방식


### 추천 서비스

|서비스|특징|무료 티어|
|---|---|---|
|**Pinecone**|전용 벡터 DB|1개 인덱스|
|**Supabase pgvector**|PostgreSQL 확장|500MB|
|**Chroma**|오픈소스|무제한|

### 구조

사용자 질문
    ↓
임베딩 생성 (OpenAI text-embedding-3-small)
    ↓
벡터 DB에서 유사 문서 검색
    ↓
검색된 문서 + 질문 → GPT
    ↓
응답 생성


### Pinecone 구현 예시

#### 5.1 환경변수 추가

```
PINECONE_API_KEY=xxx
PINECONE_INDEX=business-admin
```

#### 5.2 지식 임베딩 및 저장 (1회 실행)

```
import { Pinecone } from "@pinecone-database/pinecone";
import OpenAI from "openai";

const pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY! });

async function indexKnowledge(documents: { id: string; text: string }[]) {
  const index = pinecone.index("business-admin");
  
  for (const doc of documents) {
    const embedding = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: doc.text,
    });
    
    await index.upsert([{
      id: doc.id,
      values: embedding.data[0].embedding,
      metadata: { text: doc.text },
    }]);
  }
}
```

#### 5.3 RAG 검색

```
async function searchKnowledge(query: string): Promise<string> {
  const index = pinecone.index("business-admin");
  
  // 질문 임베딩
  const queryEmbedding = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  
  // 유사 문서 검색
  const results = await index.query({
    vector: queryEmbedding.data[0].embedding,
    topK: 3,
    includeMetadata: true,
  });
  
  // 검색된 문서 텍스트 결합
  return results.matches
    ?.map((m) => m.metadata?.text)
    .join("\n\n") || "";
}
```

## 6. 관리자 페이지 구축

### 간단한 관리 페이지 예시

#### 6.1 페이지 생성
**파일**: `app/admin/page.tsx`

```
"use client";
import { useState, useEffect } from "react";

export default function AdminPage() {
  const [knowledge, setKnowledge] = useState("");
  
  const handleSave = async () => {
    await fetch("/api/knowledge", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ content: knowledge }),
    });
    alert("저장되었습니다!");
  };
  
  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-4">지식 관리</h1>
      <textarea
        className="w-full h-96 p-4 border rounded"
        value={knowledge}
        onChange={(e) => setKnowledge(e.target.value)}
      />
      <button
        className="mt-4 px-6 py-2 bg-blue-600 text-white rounded"
        onClick={handleSave}
      >
        저장
      </button>
    </div>
  );
}
```

#### 6.2 접근 보호 (선택)

```
// 간단한 비밀번호 보호
const PASSWORD = process.env.ADMIN_PASSWORD;

if (request.headers.get("x-admin-key") !== PASSWORD) {
  return new Response("Unauthorized", { status: 401 });
}
```

### 📋 추천 구현 순서

### 1단계: 외부 파일 방식 (즉시 적용 가능)

- JSON 파일로 지식 분리
    
- 코드 수정 없이 데이터만 업데이트
    

### 2단계: 데이터베이스 연동 (확장 시)

- Supabase 무료 티어 활용
    
- 관리자 페이지 구축
    

### 3단계: 벡터 DB + RAG (대규모 시)

- 지식이 많아지면 의미 검색 도입
    
- Pinecone 또는 Supabase pgvector