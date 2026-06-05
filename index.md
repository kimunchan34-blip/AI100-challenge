# AI 100일 챌린지 시스템 — 핵심 구현 코드

> 스택: NestJS + Supabase(PostgreSQL) + React + Tailwind CSS  
> MVP 기준 핵심 파일만 발췌

---

## 1. DB 마이그레이션 (Supabase SQL)

```sql
-- 사용자 테이블 (사번 기반 인증)
CREATE TABLE users (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  employee_id VARCHAR(20) UNIQUE NOT NULL,  -- 사번
  name        VARCHAR(50) NOT NULL,
  department  VARCHAR(50),
  role        VARCHAR(20) DEFAULT 'participant' CHECK (role IN ('participant','admin','judge')),
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- 챌린지 테이블
CREATE TABLE challenges (
  id           UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  seq          SMALLINT NOT NULL,            -- 순서 (1,2,3,4)
  title        VARCHAR(100) NOT NULL,
  description  TEXT,
  submit_type  VARCHAR(20) DEFAULT 'text',   -- text|url|file|mixed
  criteria     TEXT,                         -- 평가 기준
  start_date   DATE NOT NULL,
  end_date     DATE NOT NULL,
  status       VARCHAR(10) DEFAULT 'upcoming' CHECK (status IN ('upcoming','active','closed')),
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- 제출물 테이블
CREATE TABLE submissions (
  id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  challenge_id  UUID REFERENCES challenges(id) ON DELETE CASCADE,
  title         VARCHAR(200) NOT NULL,
  content       TEXT,
  url           TEXT,
  file_url      TEXT,
  tags          TEXT[],                      -- 키워드 배열
  is_featured   BOOLEAN DEFAULT false,       -- 우수 사례 여부
  is_editable   BOOLEAN DEFAULT true,        -- 수정 가능 여부
  submitted_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, challenge_id)              -- 챌린지당 1건 (필요시 제거)
);

-- 좋아요
CREATE TABLE likes (
  id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  submission_id UUID REFERENCES submissions(id) ON DELETE CASCADE,
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  created_at    TIMESTAMPTZ DEFAULT now(),
  UNIQUE(submission_id, user_id)
);

-- 댓글
CREATE TABLE comments (
  id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  submission_id UUID REFERENCES submissions(id) ON DELETE CASCADE,
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  body          TEXT NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now()
);

-- 심사 점수
CREATE TABLE scores (
  id             UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  submission_id  UUID REFERENCES submissions(id) ON DELETE CASCADE,
  judge_id       UUID REFERENCES users(id),
  creativity     SMALLINT CHECK (creativity BETWEEN 0 AND 10),   -- 창의성
  practicality   SMALLINT CHECK (practicality BETWEEN 0 AND 10), -- 실용성
  completeness   SMALLINT CHECK (completeness BETWEEN 0 AND 10), -- 완성도
  comment        TEXT,
  scored_at      TIMESTAMPTZ DEFAULT now(),
  UNIQUE(submission_id, judge_id)
);

-- 수상
CREATE TABLE awards (
  id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  submission_id UUID REFERENCES submissions(id),
  grade         VARCHAR(20) CHECK (grade IN ('대상','최우수상','우수상','장려상')),
  awarded_at    TIMESTAMPTZ DEFAULT now()
);

-- RLS (Row Level Security) 정책 예시
ALTER TABLE submissions ENABLE ROW LEVEL SECURITY;
-- 모든 인증 사용자는 조회 가능
CREATE POLICY "submissions_select" ON submissions FOR SELECT USING (auth.role() = 'authenticated');
-- 본인 제출물만 수정
CREATE POLICY "submissions_update" ON submissions FOR UPDATE USING (user_id = auth.uid());
```

---

## 2. NestJS — 인증 모듈 (JWT + 사번)

```typescript
// src/auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  /** 사번 + 이름으로 로그인 (MVP: 단순 조회 후 토큰 발급) */
  async login(employeeId: string, name: string) {
    // 실제 환경에서는 사내 AD/LDAP 연동
    const user = await this.usersService.findOrCreate({ employeeId, name });

    if (!user) throw new UnauthorizedException('사번을 확인해주세요');

    const payload = { sub: user.id, role: user.role, name: user.name };
    return {
      access_token: this.jwtService.sign(payload),
      user: { id: user.id, name: user.name, role: user.role, department: user.department },
    };
  }
}

// src/auth/jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, role: payload.role, name: payload.name };
  }
}
```

---

## 3. NestJS — 챌린지 API

```typescript
// src/challenges/challenges.service.ts
import { Injectable } from '@nestjs/common';
import { SupabaseService } from '../supabase/supabase.service';

@Injectable()
export class ChallengesService {
  constructor(private supabase: SupabaseService) {}

  /** 전체 챌린지 목록 조회 (상태 포함) */
  async findAll() {
    const { data, error } = await this.supabase.client
      .from('challenges')
      .select('*, submissions(count)')  // 제출 수 집계
      .order('seq');
    if (error) throw error;
    return data;
  }

  /** 챌린지 상세 + 제출물 목록 */
  async findOne(id: string) {
    const { data, error } = await this.supabase.client
      .from('challenges')
      .select(`
        *,
        submissions (
          id, title, content, url, tags, is_featured, submitted_at,
          users (name, department),
          likes (count),
          comments (count)
        )
      `)
      .eq('id', id)
      .single();
    if (error) throw error;
    return data;
  }

  /** 챌린지 상태 자동 업데이트 (Cron Job에서 호출) */
  async syncStatuses() {
    const today = new Date().toISOString().split('T')[0];
    await this.supabase.client
      .from('challenges')
      .update({ status: 'active' })
      .lte('start_date', today)
      .gte('end_date', today);
    await this.supabase.client
      .from('challenges')
      .update({ status: 'closed' })
      .lt('end_date', today);
  }
}

// src/challenges/challenges.controller.ts
import { Controller, Get, Param, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { ChallengesService } from './challenges.service';

@Controller('challenges')
@UseGuards(JwtAuthGuard)  // 모든 엔드포인트 인증 필요
export class ChallengesController {
  constructor(private readonly svc: ChallengesService) {}

  @Get()
  findAll() { return this.svc.findAll(); }

  @Get(':id')
  findOne(@Param('id') id: string) { return this.svc.findOne(id); }
}
```

---

## 4. NestJS — 제출물 API

```typescript
// src/submissions/submissions.service.ts
import { Injectable, ForbiddenException } from '@nestjs/common';
import { SupabaseService } from '../supabase/supabase.service';

interface CreateSubmissionDto {
  challengeId: string;
  title: string;
  content?: string;
  url?: string;
  fileUrl?: string;
  tags?: string[];
}

@Injectable()
export class SubmissionsService {
  constructor(private supabase: SupabaseService) {}

  /** 제출물 생성 */
  async create(userId: string, dto: CreateSubmissionDto) {
    const { data, error } = await this.supabase.client
      .from('submissions')
      .insert({
        user_id: userId,
        challenge_id: dto.challengeId,
        title: dto.title,
        content: dto.content,
        url: dto.url,
        file_url: dto.fileUrl,
        tags: dto.tags ?? [],
      })
      .select()
      .single();
    if (error) throw error;
    return data;
  }

  /** 갤러리 목록 (좋아요 수 포함) */
  async findAll(challengeId?: string, featuredOnly = false) {
    let query = this.supabase.client
      .from('submissions')
      .select(`
        id, title, content, url, tags, is_featured, submitted_at,
        users (name, department),
        likes (count),
        comments (count)
      `)
      .order('submitted_at', { ascending: false });

    if (challengeId) query = query.eq('challenge_id', challengeId);
    if (featuredOnly) query = query.eq('is_featured', true);

    const { data, error } = await query;
    if (error) throw error;
    return data;
  }

  /** 좋아요 토글 */
  async toggleLike(submissionId: string, userId: string) {
    // 이미 좋아요 눌렀는지 확인
    const { data: existing } = await this.supabase.client
      .from('likes')
      .select('id')
      .eq('submission_id', submissionId)
      .eq('user_id', userId)
      .single();

    if (existing) {
      // 좋아요 취소
      await this.supabase.client.from('likes').delete().eq('id', existing.id);
      return { liked: false };
    } else {
      // 좋아요 추가
      await this.supabase.client.from('likes').insert({ submission_id: submissionId, user_id: userId });
      return { liked: true };
    }
  }

  /** 운영자: 우수 사례 설정 */
  async setFeatured(id: string, isFeatured: boolean) {
    const { data, error } = await this.supabase.client
      .from('submissions')
      .update({ is_featured: isFeatured })
      .eq('id', id)
      .select()
      .single();
    if (error) throw error;
    return data;
  }
}
```

---

## 5. React — 메인 대시보드 컴포넌트

```tsx
// src/pages/Dashboard.tsx
import { useEffect, useState } from 'react';
import { api } from '../lib/api';

interface Stats {
  totalParticipants: number;
  totalSubmissions: number;
  participationRate: number;
  featuredCount: number;
}

interface Challenge {
  id: string;
  seq: number;
  title: string;
  status: 'upcoming' | 'active' | 'closed';
  start_date: string;
  end_date: string;
}

export default function Dashboard() {
  const [stats, setStats] = useState<Stats | null>(null);
  const [challenges, setChallenges] = useState<Challenge[]>([]);
  const [currentDay, setCurrentDay] = useState(0);

  useEffect(() => {
    // 100일 챌린지 시작일 기준 현재 일수 계산
    const START_DATE = new Date('2025-01-02');
    const today = new Date();
    const diff = Math.ceil((today.getTime() - START_DATE.getTime()) / 86400000);
    setCurrentDay(Math.max(1, Math.min(diff, 100)));

    // 데이터 로드
    api.get('/stats').then(r => setStats(r.data));
    api.get('/challenges').then(r => setChallenges(r.data));
  }, []);

  const progressPct = (currentDay / 100) * 100;

  return (
    <div className="p-6 max-w-5xl mx-auto">
      {/* 100일 타임라인 */}
      <div className="bg-white rounded-2xl p-5 shadow-sm border border-gray-100 mb-6">
        <div className="flex justify-between items-center mb-3">
          <h2 className="text-base font-semibold text-gray-800">전체 진행 현황</h2>
          <span className="text-sm text-gray-500">
            Day <strong className="text-blue-600">{currentDay}</strong> / 100
          </span>
        </div>
        {/* 진행 바 */}
        <div className="relative h-4 bg-gray-100 rounded-full overflow-visible mb-2">
          <div
            className="h-full bg-blue-500 rounded-full transition-all duration-700 relative"
            style={{ width: `${progressPct}%` }}
          >
            <span className="absolute -right-3 -top-1 w-6 h-6 bg-blue-600 rounded-full border-2 border-white flex items-center justify-center text-white text-xs font-bold">
              {currentDay}
            </span>
          </div>
        </div>
        {/* 구간 레이블 */}
        <div className="flex justify-between text-xs text-gray-400 mt-3">
          {['Ch1 프롬프트', 'Ch2 아이디어', 'Ch3 AI 앱', 'AI 페스티벌'].map((label, i) => {
            const isActive = i === 1; // Day 42 → Ch2 진행중
            return (
              <span key={i} className={isActive ? 'text-blue-600 font-medium' : ''}>
                {label}
              </span>
            );
          })}
        </div>
      </div>

      {/* 요약 카드 */}
      {stats && (
        <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
          {[
            { label: '전체 참여자', value: stats.totalParticipants.toLocaleString(), sub: '↑ 12 이번주', subColor: 'text-green-500' },
            { label: '총 제출물', value: stats.totalSubmissions.toLocaleString(), sub: '오늘 +38건', subColor: 'text-blue-500' },
            { label: '참여율', value: `${stats.participationRate}%`, sub: '목표 80% 달성', subColor: 'text-green-500' },
            { label: '우수 사례', value: stats.featuredCount.toString(), sub: '선정 완료', subColor: 'text-gray-400' },
          ].map((card, i) => (
            <div key={i} className="bg-white rounded-xl p-4 border border-gray-100 shadow-sm">
              <p className="text-xs text-gray-500 mb-1">{card.label}</p>
              <p className="text-2xl font-semibold text-gray-800">{card.value}</p>
              <p className={`text-xs mt-1 ${card.subColor}`}>{card.sub}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## 6. 환경 변수 설정 (.env.example)

```bash
# Supabase
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJI...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJI...

# JWT
JWT_SECRET=your-super-secret-key-change-in-production
JWT_EXPIRES_IN=7d

# App
PORT=3001
FRONTEND_URL=http://localhost:5173

# 파일 업로드 제한
MAX_FILE_SIZE_MB=20
ALLOWED_FILE_TYPES=pdf,pptx,docx,png,jpg,jpeg
```

---

## 7. GitHub Actions CI/CD (.github/workflows/deploy.yml)

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
      # Render 자동 배포 (Render의 deploy hook URL 사용)
      - name: Trigger Render Deploy
        run: curl -X POST ${{ secrets.RENDER_DEPLOY_HOOK_URL }}

  deploy-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
      # Vercel CLI로 프론트엔드 배포
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```
