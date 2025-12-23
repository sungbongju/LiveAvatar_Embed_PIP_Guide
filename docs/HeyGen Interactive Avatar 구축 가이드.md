
# HeyGen Interactive Avatar 구축 가이드

> 차의과학대학교 경영학전공 AI 상담 아바타 구축 프로젝트
> 작성일: 2025년 12월

---

## 📋 목차

1. [[#1. 개요]]
2. [[#2. 아키텍처]]
3. [[#3. 환경 설정]]
4. [[#4. 프로젝트 구조]]
5. [[#5. UI 커스터마이징]]
6. [[#6. OpenAI GPT 연동]]
7. [[#7. GitHub Pages 임베딩]]
8. [[#8. 실시간 지식관리]]

---

## 1. 개요

### 프로젝트 목표
- AI 아바타를 통한 경영학전공 상담 서비스 제공
- 음성/텍스트 기반 실시간 대화
- OpenAI GPT와 연동하여 전공 지식 기반 응답

### 기술 스택
| 구분 | 기술 |
|------|------|
| 아바타 | HeyGen Interactive Avatar SDK |
| 프론트엔드 | Next.js 15, React, TypeScript |
| AI | OpenAI GPT-4o-mini |
| 배포 | Netlify |
| 호스팅 | GitHub Pages (임베딩) |

### 주요 URL
- **Netlify 앱**: https://dapper-moonbeam-54a024.netlify.app
- **GitHub Pages**: https://sdkparkforbi.github.io/d-id-agents/
- **GitHub 레포**: https://github.com/sdkparkforbi/InteractiveAvatarNextJSDemo

---

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Pages                              │
│                 (iframe 임베딩)                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Netlify                                   │
│              (Next.js 앱 호스팅)                             │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ /api/       │  │ /api/chat   │  │ Components  │         │
│  │ get-access- │  │ (OpenAI)    │  │ Interactive │         │
│  │ token       │  │             │  │ Avatar      │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ HeyGen   │    │ OpenAI   │    │ 사용자   │
    │ API      │    │ API      │    │ 브라우저 │
    └──────────┘    └──────────┘    └──────────┘
```

### 데이터 흐름
```
사용자 텍스트 입력
    ↓
OpenAI API (/api/chat)
    ↓
GPT-4o-mini + System Prompt (전공 지식)
    ↓
응답 생성
    ↓
HeyGen Avatar가 응답 말함 (avatar.speak)
```

---

## 3. 환경 설정

### 3.1 필요한 계정
- [HeyGen](https://app.heygen.com) - API Key 발급
- [OpenAI](https://platform.openai.com) - API Key 발급
- [GitHub](https://github.com) - 레포지토리
- [Netlify](https://netlify.com) - 배포

### 3.2 HeyGen API Key 발급
1. https://app.heygen.com 로그인
2. Settings → API 메뉴
3. API Key 복사

### 3.3 Netlify 환경변수 설정
| 변수명 | 설명 |
|--------|------|
| `HEYGEN_API_KEY` | HeyGen API 키 |
| `OPENAI_API_KEY` | OpenAI API 키 |

**설정 경로**: Netlify Dashboard → Site settings → Environment variables

### 3.4 GitHub-Netlify 연동
1. Netlify → "Add new site" → "Import an existing project"
2. GitHub 연결 → 레포지토리 선택
3. Build settings:
   - Build command: `pnpm run build`
   - Publish directory: `.next`

---

## 4. 프로젝트 구조

```
InteractiveAvatarNextJSDemo/
├── app/
│   ├── api/
│   │   ├── get-access-token/
│   │   │   └── route.ts          # HeyGen 토큰 발급
│   │   └── chat/
│   │       └── route.ts          # OpenAI 연동 ⭐
│   ├── layout.tsx                # 레이아웃 (NavBar 제거됨)
│   └── page.tsx                  # 메인 페이지
├── components/
│   └── InteractiveAvatar.tsx     # 아바타 컴포넌트 ⭐
├── .env                          # 환경변수 (로컬)
└── netlify.toml                  # Netlify 설정
```

---

## 5. UI 커스터마이징

### 5.1 제거된 요소
- ❌ NavBar ("SDK NextJS Demo", "Avatars" 메뉴)
- ❌ 설정 화면 (AvatarConfig)
- ❌ "Connection Quality" 표시
- ❌ 영어 버튼들

### 5.2 최종 UI
```
┌─────────────────────────┐
│                     [✕] │  ← 종료 버튼
│                         │
│     [아바타 영상]        │
│                         │
├─────────────────────────┤
│ [질문 입력창...] [전송]  │  ← 채팅 입력
└─────────────────────────┘
```

### 5.3 주요 파일 수정

#### `app/layout.tsx`
```typescript
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body className="h-screen w-screen overflow-hidden">
        {children}  {/* NavBar 제거됨 */}
      </body>
    </html>
  );
}
```

#### `app/page.tsx`
```typescript
"use client";
import InteractiveAvatar from "@/components/InteractiveAvatar";

export default function App() {
  return (
    <main className="w-screen h-screen">
      <InteractiveAvatar />
    </main>
  );
}
```

---

## 6. OpenAI GPT 연동

### 6.1 API Route 생성

**파일**: `app/api/chat/route.ts`

```typescript
import { NextRequest } from "next/server";
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

const SYSTEM_PROMPT = `당신은 차의과학대학교 경영학전공의 AI 상담사입니다.
학생들의 전공 관련 질문에 친절하고 정확하게 답변해주세요.
답변은 간결하게 2-3문장으로 해주세요.

## 경영학전공 지식
[여기에 지식 내용 추가]
`;

export async function POST(request: NextRequest) {
  const { message, history } = await request.json();

  const messages = [
    { role: "system", content: SYSTEM_PROMPT },
    ...history,
    { role: "user", content: message },
  ];

  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages,
    max_tokens: 300,
  });

  const reply = response.choices[0]?.message?.content;

  return new Response(JSON.stringify({ reply }), {
    headers: { "Content-Type": "application/json" },
  });
}
```

### 6.2 아바타 컴포넌트에서 호출

```typescript
// OpenAI API 호출
const response = await fetch("/api/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message: textToSend, history: chatHistory }),
});

const data = await response.json();

// 아바타가 응답 말하기
await avatarRef.current.speak({
  text: data.reply,
  taskType: TaskType.TALK,
});
```

---

## 7. GitHub Pages 임베딩

### 7.1 iframe 코드
```html
<iframe 
  id="heygen-pip" 
  src="https://dapper-moonbeam-54a024.netlify.app" 
  allow="camera; microphone; autoplay; clipboard-write">
</iframe>
```

### 7.2 CSS 스타일 (아바타 크기 맞춤)
```css
#heygen-pip {
    width: 380px;
    height: 214px;  /* 16:9 비율 */
    border: none;
    border-radius: 16px;
    box-shadow: 0 8px 32px rgba(0,0,0,0.3);
    background: transparent;
}

@media (max-width: 768px) {
    #heygen-pip {
        width: 300px;
        height: 169px;
    }
}
```

---

## 8. 실시간 지식관리

> 별도 문서: [[HeyGen 실시간 지식관리 가이드]]

---

## 🔧 트러블슈팅

### 빌드 에러: `avatar` does not exist
```typescript
// ❌ 잘못됨
const { avatar } = useStreamingAvatarSession();

// ✅ 올바름
const { avatarRef } = useStreamingAvatarSession();
avatarRef.current.speak(...)
```

### 빌드 에러: TaskType
```typescript
// ❌ 잘못됨
taskType: "talk"

// ✅ 올바름
import { TaskType } from "@heygen/streaming-avatar";
taskType: TaskType.TALK
```

### Netlify 배포 안 됨
- **캐시 문제**: "Deploy project without cache" 실행
- **환경변수 누락**: HEYGEN_API_KEY, OPENAI_API_KEY 확인

---

## 📚 참고 자료

- [HeyGen API Docs](https://docs.heygen.com)
- [HeyGen Labs](https://labs.heygen.com/interactive-avatar)
- [OpenAI API Docs](https://platform.openai.com/docs)
- [Next.js Docs](https://nextjs.org/docs)

---

#HeyGen #AI아바타 #OpenAI #NextJS #Netlify
