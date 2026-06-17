# Workout Tracker

A personal strength training tracker. Built as a single HTML file, deployed to GitHub Pages, with Supabase for cross-device sync and authentication. Currently includes linear progression programs (4, 5, and 6 day splits), with plans to add more programs and custom workout creation.

## Features

- **Program variants** — 4, 5, and 6 day splits, including short variants and squat/deadlift focus options
- **Training Max tracking** — set TMs for bench, squat, deadlift, and OHP; auto-suggests increases based on AMRAP performance
- **Plate math** — shows exact plate configuration per side for each set
- **Rest timers** — live per-set timers that freeze when you move to the next set
- **TM history graphs** — SVG line charts showing your progression over weeks
- **Dark mode**
- **lbs / kg toggle**
- **Cross-device sync** — sign in to access your data on any device (Supabase auth + database)
- **AI weekly summary** — Gemini-generated recap after each week, reviewing progression, AMRAP performance, and days completed

## Stack

- Single HTML file (`public/index.html`) — all CSS and JS inline, no build step
- [Supabase](https://supabase.com) — auth and a single `user_state` JSONB column for syncing state
- Google Gemini API (via Supabase Edge Function) — weekly AI summary
- GitHub Pages — hosting

## Setup

### 1. Supabase

Create a project at [supabase.com](https://supabase.com), then run this in the SQL Editor:

```sql
CREATE TABLE user_state (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id),
  state JSONB
);

ALTER TABLE user_state ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own state" ON user_state
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own state" ON user_state
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own state" ON user_state
  FOR UPDATE USING (auth.uid() = user_id);

GRANT SELECT, INSERT, UPDATE ON user_state TO authenticated;
```

Update the credentials in `public/index.html`:

```js
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-publishable-key';
```

### 2. AI weekly summary (optional)

Get a free Gemini API key at [aistudio.google.com](https://aistudio.google.com).

In the Supabase dashboard, go to **Edge Functions → Secrets** and add:

```
GEMINI_API_KEY = your-key-here
```

Then create an Edge Function named `weekly-summary` with the following:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'

const cors = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

serve(async (req) => {
  if (req.method === 'OPTIONS') return new Response('ok', { headers: cors })

  const { weekData } = await req.json()
  const key = Deno.env.get('GEMINI_API_KEY')

  if (!key) {
    return new Response(JSON.stringify({ error: 'GEMINI_API_KEY secret not set' }), {
      headers: { ...cors, 'Content-Type': 'application/json' }
    })
  }

  const prompt = `You are a concise strength training coach. Summarize this training week in 2-3 short sentences. Be specific and encouraging. End with one actionable tip if relevant.

Program: ${weekData.program}
Completed week: ${weekData.completedWeek}
Days finished: ${weekData.daysCompleted} of ${weekData.totalDays}
TM changes this week (${weekData.unit}): ${weekData.tmChanges}
AMRAP reps logged: ${weekData.amraps}
Total weeks on program: ${weekData.totalWeeks}`

  const geminiRes = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${key}`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        contents: [{ parts: [{ text: prompt }] }],
        generationConfig: { maxOutputTokens: 1024, temperature: 0.7 }
      })
    }
  )

  const data = await geminiRes.json()

  if (!geminiRes.ok) {
    return new Response(JSON.stringify({ error: `Gemini error ${geminiRes.status}: ${data?.error?.message ?? JSON.stringify(data)}` }), {
      headers: { ...cors, 'Content-Type': 'application/json' }
    })
  }

  const summary = data.candidates?.[0]?.content?.parts?.[0]?.text ?? 'No response from Gemini.'
  return new Response(JSON.stringify({ summary }), {
    headers: { ...cors, 'Content-Type': 'application/json' }
  })
})
```

### 3. GitHub Pages

The repository deploys automatically via GitHub Actions on push to `main`. Enable Pages in your repository settings under **Settings → Pages → Source → GitHub Actions**.

## Program variants

| Key | Description |
|-----|-------------|
| `4day` | 4 day standard |
| `4day-short` | 4 day short (fewer T1 sets) |
| `5day` | 5 day standard |
| `5day-short` | 5 day short |
| `6day-squat` | 6 day squat focus |
| `6day-squat-short` | 6 day squat focus short |
| `6day-deadlift` | 6 day deadlift focus |
| `6day-deadlift-short` | 6 day deadlift focus short |
