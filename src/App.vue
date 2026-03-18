<template>
  <div class="app">
    <h1>🌸 My Workout Plan</h1>
    <p class="subtitle">Flat belly · Fat loss · Knock-knee friendly · 45 min/day</p>

    <div class="warmup-banner">
      <h3>🔥 Daily Warm-up <span class="wu-note">(5 min · included in every day)</span></h3>
      <div class="wu-items">
        <span class="wu-tag" v-for="w in warmupPreview" :key="w">{{ w }}</span>
      </div>
    </div>

    <div class="day-card" v-for="(day, di) in days" :key="di" :class="{ open: openDay === di }">
      <div class="day-header" @click="toggleDay(di)">
        <div class="day-num">{{ di + 1 }}</div>
        <div class="day-info">
          <div class="day-title">{{ day.title }}</div>
          <div class="day-sub">{{ day.subtitle }}</div>
        </div>
        <div class="prog-ring">
          <svg width="40" height="40" viewBox="0 0 40 40">
            <circle cx="20" cy="20" r="16" fill="none" stroke="#2e2e3e" stroke-width="3.5" />
            <circle cx="20" cy="20" r="16" fill="none"
              :stroke="dayPct(di) === 100 ? '#34d399' : '#f472b6'"
              stroke-width="3.5" stroke-dasharray="100.5"
              :stroke-dashoffset="100.5 * (1 - dayPct(di) / 100)"
              stroke-linecap="round" />
          </svg>
          <div class="pct">{{ dayPct(di) }}%</div>
        </div>
        <div class="chevron">▾</div>
      </div>

      <transition name="scroll-fade">
        <div class="day-body" v-if="openDay === di">
          <div class="day-note" v-if="day.note">💡 {{ day.note }}</div>

          <template v-for="(sec, si) in day.sections" :key="si">
            <div class="section-label" :class="{ 'warmup-lbl': sec.warmup, 'cooldown-lbl': sec.cooldown }">
              {{ sec.label }}
            </div>

            <div class="ex-row" :class="{ 'warmup-ex': sec.warmup, 'cooldown-ex': sec.cooldown }"
              v-for="(ex, ei) in sec.exercises" :key="ei">
              <div class="ex-main">
                <div class="ex-check" :class="{ done: isExDone(di, si, ei) }" @click="toggleDone(di, si, ei)">
                  <span v-if="isExDone(di, si, ei)" class="check-icon">✓</span>
                </div>
                <div class="ex-name" :class="{ done: isExDone(di, si, ei) }">
                  {{ ex.name }}
                  <span class="warmup-tag"   v-if="sec.warmup">Warm-up</span>
                  <span class="cooldown-tag" v-else-if="sec.cooldown">Cool-down</span>
                  <span class="kk-tag"       v-if="ex.kkFocus">🦵 KK</span>
                </div>
                <div class="ex-sets" v-if="ex.sets > 1">
                  <div class="set-pip" v-for="s in ex.sets" :key="s"
                    :class="{
                      active:   getState(di,si,ei).currentSet === s && getState(di,si,ei).running,
                      complete: s <= getState(di,si,ei).completedSets
                    }">{{ s }}</div>
                </div>
                <div class="ex-btns">
                  <button class="btn btn-detail" :class="{ active: detailOpen(di,si,ei) }" @click.stop="toggleDetail(di,si,ei)">
                    {{ detailOpen(di,si,ei) ? '▴ hide' : 'ℹ detail' }}
                  </button>
                  <button class="btn btn-stop"  v-if="getState(di,si,ei).running"          @click="stopSet(di,si,ei)">✓ Done</button>
                  <button class="btn btn-start" v-else-if="!isExDone(di,si,ei)"            @click="startSet(di,si,ei)">▶ Start</button>
                  <span v-else class="done-icon">✅</span>
                </div>
              </div>

              <div class="timer-bar" v-if="getState(di,si,ei).running">
                <div class="timer-val">{{ fmtTime(getState(di,si,ei).elapsed) }}</div>
                <div class="timer-label">
                  <template v-if="ex.sets > 1">Set {{ getState(di,si,ei).currentSet }}/{{ ex.sets }} &nbsp;·&nbsp;</template>
                  {{ ex.reps }}
                </div>
                <div class="timer-prog-wrap" v-if="ex.targetSec">
                  <div class="timer-prog-bar" :style="{ width: Math.min(100, getState(di,si,ei).elapsed / ex.targetSec * 100) + '%' }"></div>
                </div>
              </div>

              <transition name="fade">
                <div class="detail-panel" v-if="detailOpen(di,si,ei)">
                  <strong>{{ ex.name }}</strong>
                  {{ ex.detail }}
                  <ul class="detail-tips" v-if="ex.tips && ex.tips.length">
                    <li v-for="t in ex.tips" :key="t">{{ t }}</li>
                  </ul>
                  <div class="kk-note" v-if="ex.kkNote">
                    <span>🦵</span> <span><strong>Knock-knee tip:</strong> {{ ex.kkNote }}</span>
                  </div>
                </div>
              </transition>
            </div>

            <div class="rest-box" v-if="sec.rest">
              <div><div class="r-label">Rest</div><div class="r-sub">Breathe, hydrate, reset</div></div>
              <div class="rest-val">{{ sec.rest }}</div>
            </div>
          </template>

          <div class="complete-banner" v-if="dayPct(di) === 100">
            <div>🌸</div>
            <div>Day {{ di + 1 }} done! You showed up — that's everything.</div>
          </div>
        </div>
      </transition>
    </div>

    <div class="period-note">
      <span>🌙</span>
      <div>
        <strong>On period days:</strong> Replace any workout with a 20-min walk + cool-down stretches only. Rest is productive.
      </div>
    </div>

    <div class="kk-legend">
      <span>🦵 KK</span> = exercises that specifically help correct knock knees by strengthening hip abductors & glute medius
    </div>
  </div>
</template>

<script setup>
import { reactive, ref } from 'vue'

// ─── Exercise library ─────────────────────────────────────────────────────
const EX = {
  // Warm-up
  'March in place':              { detail: 'Lift knees to hip height, swing arms naturally. Gentle full-body activation.', tips: ['Keep posture upright', 'Breathe slowly and deeply'], kkNote: null },
  'Hip circles':                 { detail: 'Hands on hips, feet shoulder-width. Make large slow circles — feel the hip joint open.', tips: ['Both directions equally', '10 circles each way'], kkNote: 'Loosens the hip joint before exercises that require knee tracking.' },
  'Shoulder rolls':              { detail: 'Roll shoulders slowly forward 5×, then backward 5×. Releases upper-body tension.', tips: ['Big, slow circles', 'Drop shoulders away from ears'], kkNote: null },
  'Cat-cow stretch':             { detail: 'On hands and knees: inhale, drop belly (cow); exhale, round spine (cat). Warms up the entire spine.', tips: ['One breath per movement', 'Never force range of motion'], kkNote: null },
  'Clamshells (band)':           { detail: 'Lie on side, knees bent, band above knees. Keep feet together and lift top knee like a clamshell opening. Lower with control.', tips: ['Hips stay stacked — don\'t roll back', 'Slow lift AND slow lower — both matter', 'Feel the outer glute/hip working'], kkFocus: true, kkNote: 'One of the best exercises for strengthening the glute medius — directly counters knock knees by pulling the knee outward.' },
  'Glute bridge with band':      { detail: 'Lie on back, band above knees, feet flat. Push knees OUT against the band as you drive hips up. Squeeze glutes at top, lower slowly.', tips: ['Press knees outward the entire time', 'Squeeze glutes — not lower back', 'Hold 1 sec at the top'], kkFocus: true, kkNote: 'The band forces your hip abductors to activate throughout — directly trains the muscles that pull knees into correct alignment.' },
  'Standing hip abduction':      { detail: 'Stand tall, band above knees. Lift one leg directly out to the side, hold 1 sec, lower with control. Alternate.', tips: ['Keep pelvis level — don\'t lean sideways', 'Hold a wall for balance if needed', 'Slow lifts are more effective'], kkFocus: true, kkNote: 'Isolates the glute medius — the key muscle that stops knees from caving in during walking and exercise.' },
  'Sumo squat':                  { detail: 'Wide stance, toes turned out 45°. Lower straight down keeping chest tall. Push knees OUT over toes throughout.', tips: ['Push knees outward consciously — especially on the way up', 'Sit tall, don\'t lean forward', 'Pause at the bottom for extra glute activation'], kkFocus: true, kkNote: 'Wide stance + toes out naturally externalises the hip, reducing inward knee pressure. Always use a band above knees to reinforce outward tracking.' },
  'Resistance band side step':   { detail: 'Band above knees, slight squat stance. Step sideways 15 steps each way. Keep tension on the band throughout — don\'t let feet fully meet.', tips: ['Stay low throughout — don\'t stand up between steps', 'Take deliberate, full steps', 'Feet never fully together — keep tension'], kkFocus: true, kkNote: 'Directly works the hip abductors in a functional way. Builds the strength needed to keep knees from caving during daily activities.' },
  'Glute kickback (band)':       { detail: 'Band above ankles, hands on wall for balance. Kick one leg straight back, squeeze glute at top. Return with control.', tips: ['Don\'t swing — slow and controlled', 'Slight forward hip hinge is fine', 'Feel it in the glute, not the lower back'], kkNote: 'Strengthens the glute max, which works together with glute medius to maintain proper knee alignment.' },
  'Sumo squat pulse':            { detail: 'Get into sumo squat position and do small, controlled pulses (about 3–4 inch range). Knees pushed out, chest up.', tips: ['Stay low — don\'t come all the way up', 'Push knees out on every single pulse', '20 pulses = 1 set'], kkFocus: true, kkNote: 'Sustained time under tension with knees actively pushed out trains your body to hold correct knee position automatically.' },
  'Wall sit with band':          { detail: 'Band above knees, back flat against wall, slide down until thighs are parallel. Push knees outward against the band and hold.', tips: ['Feet shoulder-width, slightly forward', 'Actively push knees OUT into the band the whole time', 'Breathe steadily — don\'t hold breath'], kkFocus: true, kkNote: 'Isometric hold with active knee-out pressure builds endurance in the hip abductors — exactly what knock knees need.' },
  'Bird dog':                    { detail: 'On hands and knees, extend one arm + opposite leg simultaneously. Hold 2 sec, return, alternate.', tips: ['Keep hips level — no rotation', 'Look down, not forward', 'Core braced throughout'], kkNote: null },
  'Plank (on knees)':            { detail: 'Forearms on mat, knees down, body straight from knees to shoulders. Hold, breathe.', tips: ['Don\'t let hips sag', 'Squeeze glutes and belly together', 'This is just as effective as a full plank'], kkNote: null },
  'Dumbbell shoulder press':     { detail: 'Seated or standing, dumbbells at shoulder height. Press overhead until nearly straight, lower with control.', tips: ["Don't arch your lower back", 'Keep core gently engaged', 'Light-to-medium weight'], kkNote: null },
  'Dumbbell bent-over row':      { detail: 'Hinge at hips, dumbbells hanging. Pull elbows back and up, squeeze shoulder blades. Lower with control.', tips: ['Back flat throughout', 'Lead with the elbow', 'No need for heavy weights'], kkNote: null },
  'Dumbbell bicep curl':         { detail: 'Arms at sides, palms forward. Curl dumbbells up, squeeze at top, lower slowly (3 sec down).', tips: ['Elbows pinned to sides', 'Slow lower = more toning', "Don't swing your body"], kkNote: null },
  'Dumbbell lateral raise':      { detail: 'Arms at sides, slight bend in elbows. Raise to shoulder height, lower slowly.', tips: ["Don't shrug", 'Thumbs slightly down', 'Very light weight works best'], kkNote: null },
  'Wall push-up':                { detail: 'Stand arm\'s length from wall, hands at shoulder height. Bend elbows to bring chest to wall, push back.', tips: ['Keep body straight like a plank', 'Elbows at 45° — not flared wide', 'Slow for maximum benefit'], kkNote: null },
  'Dumbbell chest fly (floor)':  { detail: 'Lie on back, knees bent, dumbbells above chest with slight elbow bend. Open arms wide toward floor, return.', tips: ['Keep elbow bend fixed throughout', 'Go only as wide as comfortable', 'Very light weight recommended'], kkNote: null },
  'Standing torso twist':        { detail: 'Feet hip-width, arms at shoulder height or holding a light dumbbell. Rotate torso left and right. Oblique activation.', tips: ['Move from waist, not hips', 'Slow and deliberate', 'Can hold a light dumbbell for extra work'], kkNote: null },
  'Side plank (on knees)':       { detail: 'Lie on side, bottom knee bent for support, top leg straight or stacked. Lift hips off floor, hold. Works obliques and outer hip.', tips: ['Keep hips lifted — don\'t let them sag', 'Free hand on hip or reaching up', 'Feel the side waist working'], kkNote: 'Also activates the hip abductor of the top leg — a bonus for knock-knee correction.' },
  'Step touch':                  { detail: 'Step one foot to the side, bring the other to meet it, rhythmically alternating. Add arm swings for heart rate.', tips: ['Add bigger arm movements to raise intensity', 'Stay light on your feet', 'Easy to sustain for 10+ minutes'], kkNote: null },
  'Slow march with arms':        { detail: 'March in place at a steady pace, lifting knees and pumping arms. Gentle heart rate elevation.', tips: ['Breathe rhythmically', 'Increase knee height to increase intensity', 'Great on lower-energy days'], kkNote: null },
  'Child\'s pose':               { detail: 'Kneel, sit back on heels, arms extended forward on the mat, forehead down. Full back and hip release.', tips: ['Breathe deeply into your back', 'Hold the full time — this is earned rest', 'Walk hands further forward for a deeper stretch'], kkNote: null },
  'Supine spinal twist':         { detail: 'Lie on back, hug one knee to chest then let it fall across your body. Arms wide, look the other way.', tips: ['Let gravity do the work', 'Keep both shoulders on the mat', 'Breathe slowly and deeply'], kkNote: null },
  'Lying glute stretch':         { detail: 'Lie on back, cross one ankle over the opposite thigh (figure-4). Gently pull both legs toward your chest.', tips: ['The closer you pull, the deeper the stretch', 'Let the hip relax — don\'t force it', 'Great after glute work'], kkNote: 'Stretches the piriformis and glute medius — helps the hip abductors recover properly after working them.' },
  'Seated hamstring stretch':    { detail: 'Sit on mat, legs extended. Hinge forward at the hips, reaching toward toes as far as comfortable.', tips: ['Flex feet for a deeper stretch', 'Hinge from hips — don\'t just round your back', 'Hold 30 sec each side'], kkNote: null },
  'Deep breathing':              { detail: 'Sit comfortably, eyes closed. Inhale 4 counts, hold 2, exhale 6 counts. Activates the parasympathetic nervous system.', tips: ['Longer exhale than inhale is the key', 'Lowers cortisol — don\'t skip this', 'End every workout here'], kkNote: null },
}

function makeEx(name, sets, reps, targetSec = null) {
  const d = EX[name] || { detail: 'A great exercise for your goals.', tips: [] }
  return { name, sets: sets || 1, reps, targetSec, detail: d.detail, tips: d.tips, kkFocus: !!d.kkFocus, kkNote: d.kkNote || null }
}

// ─── Warm-up ──────────────────────────────────────────────────────────────
const warmupSection = {
  label: '🔥 Warm-up (5 min)',
  warmup: true,
  exercises: [
    makeEx('March in place',   1, '2 min', 120),
    makeEx('Hip circles',      1, '1 min', 60),
    makeEx('Shoulder rolls',   1, '30 sec', 30),
    makeEx('Cat-cow stretch',  1, '1 min', 60),
    makeEx('Clamshells (band)',1, '10 reps each side'),
  ],
}

// ─── Cool-down ────────────────────────────────────────────────────────────
const cooldownSection = {
  label: '🌙 Cool-down (5 min)',
  cooldown: true,
  exercises: [
    makeEx('Child\'s pose',            1, '1 min', 60),
    makeEx('Lying glute stretch',      1, '1 min each side', 120),
    makeEx('Supine spinal twist',      1, '30 sec each side', 60),
    makeEx('Deep breathing',           1, '1 min', 60),
  ],
}

const warmupPreview = [
  'March in place – 2 min',
  'Hip circles – 1 min',
  'Shoulder rolls – 30 sec',
  'Cat-cow stretch – 1 min',
  'Clamshells (band) – 10 each side',
]

// ─── Weekly plan ──────────────────────────────────────────────────────────
const days = [
  {
    title: 'Day 1 – Glutes + Hip Abductors',
    subtitle: 'Warm-up · Glute & hip work · Cool-down',
    note: 'This is your most important day. Strong glutes and hip abductors are the #1 fix for knock knees. Use your resistance band for every exercise that calls for it.',
    sections: [
      warmupSection,
      {
        label: '🍑 Glutes & Hip Abductors (30 min)',
        rest: '45 sec',
        exercises: [
          makeEx('Glute bridge with band',    4, '15 reps'),
          makeEx('Sumo squat',                4, '12 reps'),
          makeEx('Resistance band side step', 3, '15 steps each way'),
          makeEx('Standing hip abduction',    3, '15 reps each side'),
          makeEx('Glute kickback (band)',      3, '12 reps each side'),
          makeEx('Wall sit with band',        3, '30 sec', 30),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 2 – Upper Body + Core',
    subtitle: 'Warm-up · Upper body · Core · Cool-down',
    note: 'Light-to-medium dumbbells. Toning the upper body and core helps create the hourglass shape and keeps your posture strong — which also reduces knee stress.',
    sections: [
      warmupSection,
      {
        label: '💪 Upper Body (18 min)',
        rest: '45 sec',
        exercises: [
          makeEx('Wall push-up',              3, '12 reps'),
          makeEx('Dumbbell shoulder press',   3, '12 reps'),
          makeEx('Dumbbell bent-over row',    3, '12 reps'),
          makeEx('Dumbbell bicep curl',       3, '12 reps'),
          makeEx('Dumbbell lateral raise',    3, '12 reps'),
          makeEx('Dumbbell chest fly (floor)',3, '12 reps'),
        ],
      },
      {
        label: '🎯 Core (10 min)',
        rest: '30 sec',
        exercises: [
          makeEx('Bird dog',            3, '10 reps each side'),
          makeEx('Plank (on knees)',    3, '30 sec', 30),
          makeEx('Side plank (on knees)',2, '25 sec each side', 25),
          makeEx('Standing torso twist',2, '30 sec', 30),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 3 – Cardio + Full Body',
    subtitle: 'Warm-up · Steady cardio · Light circuit · Cool-down',
    note: 'Keep effort at a comfortable 5/10 — you should be able to hold a conversation. This is your fat-burn day, not a push day.',
    sections: [
      warmupSection,
      {
        label: '🚶 Steady Cardio (15 min)',
        exercises: [
          makeEx('Step touch',         1, '5 min', 300),
          makeEx('Slow march with arms',1,'5 min', 300),
          makeEx('Step touch',         1, '5 min', 300),
        ],
      },
      {
        label: '🔄 Light Full-Body Circuit × 3 (12 min)',
        rest: '60 sec between rounds',
        exercises: [
          makeEx('Sumo squat',              3, '12 reps'),
          makeEx('Wall push-up',            3, '10 reps'),
          makeEx('Glute bridge with band',  3, '15 reps'),
          makeEx('Dumbbell bent-over row',  3, '10 reps'),
          makeEx('Standing hip abduction',  3, '10 reps each side'),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 4 – Lower Body Sculpt',
    subtitle: 'Warm-up · Legs & glutes · Cool-down',
    note: 'All lower-body exercises today use a wide stance or band to keep knees in safe alignment. Focus on pushing knees OUT on every single rep.',
    sections: [
      warmupSection,
      {
        label: '🦵 Lower Body Sculpt (30 min)',
        rest: '45 sec',
        exercises: [
          makeEx('Sumo squat',               4, '15 reps'),
          makeEx('Sumo squat pulse',         3, '20 pulses'),
          makeEx('Glute bridge with band',   4, '20 reps'),
          makeEx('Resistance band side step',3, '20 steps each way'),
          makeEx('Clamshells (band)',         3, '15 reps each side'),
          makeEx('Glute kickback (band)',     3, '15 reps each side'),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 5 – Core + Upper Body',
    subtitle: 'Warm-up · Core focus · Upper body · Cool-down',
    note: 'A strong core reduces pressure on your knees during daily movement. Every core exercise here is active and engaging — no boring holds.',
    sections: [
      warmupSection,
      {
        label: '🎯 Core Focus (18 min)',
        rest: '30 sec',
        exercises: [
          makeEx('Bird dog',             3, '12 reps each side'),
          makeEx('Plank (on knees)',     3, '35 sec', 35),
          makeEx('Side plank (on knees)',3, '30 sec each side', 30),
          makeEx('Standing torso twist', 3, '40 sec', 40),
          makeEx('Glute bridge with band',3,'15 reps'),
        ],
      },
      {
        label: '💪 Upper Body (10 min)',
        rest: '45 sec',
        exercises: [
          makeEx('Dumbbell shoulder press',  2, '12 reps'),
          makeEx('Dumbbell bicep curl',      2, '12 reps'),
          makeEx('Dumbbell bent-over row',   2, '12 reps'),
          makeEx('Wall push-up',             2, '12 reps'),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 6 – Active Recovery',
    subtitle: 'Gentle movement · Full-body stretch · No intensity',
    note: 'Walk outside if you can — 25–30 min at a comfortable pace. If indoors, use the march. This day helps you recover so Days 1 and 4 can actually work.',
    sections: [
      {
        label: '🚶 Easy Movement (25 min)',
        exercises: [
          makeEx('Slow march with arms', 1, '10 min', 600),
          makeEx('Step touch',           1, '8 min',  480),
          makeEx('Slow march with arms', 1, '7 min',  420),
        ],
      },
      {
        label: '🧘 Full-Body Stretch (15 min)',
        cooldown: true,
        exercises: [
          makeEx('Cat-cow stretch',         1, '2 min', 120),
          makeEx('Child\'s pose',           1, '2 min', 120),
          makeEx('Lying glute stretch',     1, '2 min each side', 240),
          makeEx('Seated hamstring stretch',1, '1 min each side', 120),
          makeEx('Supine spinal twist',     1, '1 min each side', 120),
          makeEx('Deep breathing',          1, '2 min', 120),
        ],
      },
    ],
  },
  {
    title: 'Day 7 – Rest',
    subtitle: 'Full rest · No workout · Recover & reset',
    note: 'Rest is where your body actually changes. Your muscles repair, your hormones reset, and tomorrow you\'ll be stronger. Eat well, sleep well.',
    sections: [
      {
        label: '💤 Gentle Rest',
        cooldown: true,
        exercises: [
          makeEx('Deep breathing',   1, '5 min', 300),
          makeEx('Child\'s pose',    1, '2 min', 120),
          makeEx('Supine spinal twist', 1, '1 min each side', 120),
        ],
      },
    ],
  },
]

// ─── State ────────────────────────────────────────────────────────────────
const openDay   = ref(null)
const exState   = reactive({})
const detailKeys = reactive({})
const timers    = {}

function key(di, si, ei) { return `${di}-${si}-${ei}` }

function getState(di, si, ei) {
  const k = key(di, si, ei)
  if (!exState[k]) exState[k] = { running: false, elapsed: 0, currentSet: 0, completedSets: 0, done: false }
  return exState[k]
}

function detailOpen(di, si, ei) { return !!detailKeys[key(di, si, ei)] }
function toggleDetail(di, si, ei) { const k = key(di, si, ei); detailKeys[k] = !detailKeys[k] }
function toggleDay(di) { openDay.value = openDay.value === di ? null : di }

function startSet(di, si, ei) {
  const s = getState(di, si, ei)
  if (s.running) return
  s.running = true
  s.currentSet = s.completedSets + 1
  s.elapsed = 0
  const k = key(di, si, ei)
  clearInterval(timers[k])
  timers[k] = setInterval(() => { s.elapsed++ }, 1000)
}

function stopSet(di, si, ei) {
  const s = getState(di, si, ei)
  s.running = false
  clearInterval(timers[key(di, si, ei)])
  s.completedSets++
  if (s.completedSets >= days[di].sections[si].exercises[ei].sets) s.done = true
}

function toggleDone(di, si, ei) {
  const s = getState(di, si, ei)
  if (s.running) stopSet(di, si, ei)
  else s.done = !s.done
}

function isExDone(di, si, ei) { return !!getState(di, si, ei).done }

function dayPct(di) {
  let total = 0, done = 0
  days[di].sections.forEach((sec, si) => {
    sec.exercises.forEach((_, ei) => { total++; if (isExDone(di, si, ei)) done++ })
  })
  return total === 0 ? 0 : Math.round(done / total * 100)
}

function fmtTime(s) {
  return `${String(Math.floor(s / 60)).padStart(2, '0')}:${String(s % 60).padStart(2, '0')}`
}
</script>

<style scoped>
* { box-sizing: border-box; margin: 0; padding: 0; }
.app {
  max-width: 780px; margin: 0 auto; padding: 20px 16px 60px;
  background: #0f0f13; min-height: 100vh;
  color: #e2e8f0; font-family: 'Segoe UI', sans-serif;
}
h1 {
  text-align: center; font-size: 1.8rem; font-weight: 800;
  background: linear-gradient(135deg, #f472b6, #a78bfa);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent; margin-bottom: 4px;
}
.subtitle { text-align: center; color: #94a3b8; font-size: .85rem; margin-bottom: 28px; }

.warmup-banner {
  background: linear-gradient(135deg, #1e1b4b, #312e81);
  border: 1px solid #4338ca; border-radius: 16px; padding: 16px 20px; margin-bottom: 24px;
}
.warmup-banner h3 { color: #a5b4fc; font-size: .95rem; margin-bottom: 10px; }
.wu-note { color: #6366f1; font-size: .75rem; font-weight: 400; }
.wu-items { display: flex; flex-wrap: wrap; gap: 8px; }
.wu-tag { background: #312e81; border: 1px solid #4338ca; border-radius: 20px; padding: 4px 12px; font-size: .78rem; color: #c7d2fe; }

.day-card {
  background: #1a1a24; border: 1px solid #2e2e3e;
  border-radius: 16px; margin-bottom: 12px; overflow: hidden; transition: box-shadow .3s;
}
.day-card.open { box-shadow: 0 0 0 1px #db2777, 0 8px 32px #db277722; }
.day-header { display: flex; align-items: center; padding: 16px 20px; cursor: pointer; gap: 14px; user-select: none; }
.day-num {
  width: 38px; height: 38px; border-radius: 50%;
  background: linear-gradient(135deg, #db2777, #9333ea);
  display: flex; align-items: center; justify-content: center; font-weight: 800; font-size: .9rem; flex-shrink: 0;
}
.day-info { flex: 1; }
.day-title { font-weight: 700; font-size: 1rem; }
.day-sub { font-size: .75rem; color: #94a3b8; margin-top: 2px; }
.prog-ring { position: relative; width: 40px; height: 40px; }
.prog-ring svg { transform: rotate(-90deg); }
.pct { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: .65rem; font-weight: 700; color: #f472b6; }
.chevron { color: #94a3b8; transition: transform .3s; font-size: 1.1rem; }
.open .chevron { transform: rotate(180deg); }
.day-body { padding: 0 16px 16px; }
.day-note {
  background: #1e1a2e; border: 1px solid #4c1d95; border-radius: 10px;
  padding: 10px 14px; font-size: .82rem; color: #c4b5fd; margin-bottom: 4px; line-height: 1.5;
}
.section-label {
  font-size: .72rem; font-weight: 700; letter-spacing: 1.5px; text-transform: uppercase;
  color: #f472b6; margin: 14px 0 8px; padding-left: 4px;
}
.section-label.warmup-lbl { color: #818cf8; }
.section-label.cooldown-lbl { color: #34d399; }

.ex-row { background: #22222f; border: 1px solid #2e2e3e; border-radius: 12px; margin-bottom: 8px; overflow: hidden; }
.ex-row.warmup-ex  { border-color: #3730a380; }
.ex-row.cooldown-ex { border-color: #065f4680; }
.ex-main { display: flex; align-items: center; padding: 12px 14px; gap: 10px; }
.ex-check {
  width: 22px; height: 22px; border-radius: 50%; border: 2px solid #2e2e3e;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0; cursor: pointer; transition: all .2s;
}
.ex-check.done { background: #34d399; border-color: #34d399; }
.check-icon { color: #fff; font-size: .8rem; }
.ex-name { flex: 1; font-size: .9rem; font-weight: 600; }
.ex-name.done { text-decoration: line-through; color: #94a3b8; }

.warmup-tag   { display: inline-block; font-size: .62rem; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; background: #3730a330; color: #818cf8; border: 1px solid #3730a360; border-radius: 6px; padding: 1px 6px; margin-left: 6px; vertical-align: middle; }
.cooldown-tag { display: inline-block; font-size: .62rem; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; background: #06403020; color: #34d399; border: 1px solid #34d39940; border-radius: 6px; padding: 1px 6px; margin-left: 6px; vertical-align: middle; }
.kk-tag       { display: inline-block; font-size: .62rem; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; background: #78350f30; color: #fbbf24; border: 1px solid #fbbf2440; border-radius: 6px; padding: 1px 6px; margin-left: 6px; vertical-align: middle; }

.ex-sets { display: flex; gap: 5px; flex-wrap: wrap; }
.set-pip {
  width: 22px; height: 22px; border-radius: 6px; border: 2px solid #2e2e3e;
  display: flex; align-items: center; justify-content: center;
  font-size: .62rem; font-weight: 700; color: #94a3b8; transition: all .2s;
}
.set-pip.active   { border-color: #fbbf24; color: #fbbf24; background: #fbbf2420; animation: pulse 1s infinite; }
.set-pip.complete { background: #34d399; border-color: #34d399; color: #fff; }
@keyframes pulse {
  0%,100% { box-shadow: 0 0 0 0 #fbbf2455; }
  50%      { box-shadow: 0 0 0 5px #fbbf2400; }
}

.ex-btns { display: flex; gap: 6px; align-items: center; }
.btn { border: none; border-radius: 8px; padding: 6px 11px; font-size: .75rem; font-weight: 700; cursor: pointer; transition: all .15s; }
.btn-start { background: linear-gradient(135deg, #db2777, #9333ea); color: #fff; }
.btn-start:hover { filter: brightness(1.15); }
.btn-stop  { background: linear-gradient(135deg, #f87171, #dc2626); color: #fff; }
.btn-stop:hover  { filter: brightness(1.15); }
.btn-detail { background: transparent; border: 1px solid #2e2e3e; color: #94a3b8; padding: 5px 9px; font-size: .72rem; }
.btn-detail:hover, .btn-detail.active { border-color: #f472b6; color: #f472b6; }
.done-icon { font-size: 1.1rem; }

.timer-bar {
  display: flex; align-items: center; gap: 10px;
  background: #fbbf2410; border-top: 1px solid #fbbf2430; padding: 8px 14px;
}
.timer-val { font-size: 1.2rem; font-weight: 800; color: #fbbf24; font-variant-numeric: tabular-nums; min-width: 54px; }
.timer-label { font-size: .75rem; color: #94a3b8; flex: 1; }
.timer-prog-wrap { flex: 1; height: 4px; background: #2e2e3e; border-radius: 2px; }
.timer-prog-bar  { height: 100%; background: #fbbf24; border-radius: 2px; transition: width .5s linear; }

.detail-panel { border-top: 1px solid #2e2e3e; padding: 12px 14px; background: #151520; font-size: .82rem; color: #94a3b8; line-height: 1.6; }
.detail-panel strong { color: #f472b6; display: block; margin-bottom: 4px; }
.detail-tips { margin-top: 8px; }
.detail-tips li { margin-left: 16px; margin-bottom: 2px; }
.kk-note {
  display: flex; gap: 8px; align-items: flex-start;
  margin-top: 10px; background: #78350f20; border: 1px solid #fbbf2430;
  border-radius: 8px; padding: 8px 10px; font-size: .8rem; color: #fde68a; line-height: 1.5;
}
.kk-note strong { color: #fbbf24; font-size: .8rem; }

.rest-box {
  background: #0f2820; border: 1px solid #34d39940; border-radius: 12px;
  padding: 10px 14px; margin: 6px 0 10px; display: flex; align-items: center; gap: 12px;
}
.r-label { font-size: .8rem; color: #34d399; font-weight: 700; }
.r-sub   { font-size: .72rem; color: #94a3b8; }
.rest-val { font-size: 1.1rem; font-weight: 800; color: #34d399; margin-left: auto; }

.period-note {
  display: flex; gap: 12px; align-items: flex-start;
  background: #1a0a2e; border: 1px solid #7c3aed60;
  border-radius: 14px; padding: 14px 18px; margin-top: 20px;
  font-size: .83rem; color: #c4b5fd; line-height: 1.6;
}
.period-note span { font-size: 1.3rem; flex-shrink: 0; }
.period-note strong { color: #f472b6; }

.kk-legend {
  text-align: center; margin-top: 12px; font-size: .78rem;
  color: #94a3b8; padding: 8px;
}
.kk-legend span { background: #78350f30; color: #fbbf24; border: 1px solid #fbbf2440; border-radius: 6px; padding: 2px 7px; font-size: .72rem; font-weight: 700; margin-right: 6px; }

.complete-banner {
  text-align: center; padding: 18px;
  background: linear-gradient(135deg, #1a0030, #3b0764);
  border: 1px solid #f472b6; border-radius: 12px; margin-top: 12px;
}
.complete-banner div:first-child { font-size: 2rem; }
.complete-banner div:last-child  { color: #f9a8d4; font-weight: 700; font-size: 1rem; margin-top: 4px; }

.fade-enter-active, .fade-leave-active { transition: opacity .25s, transform .25s; }
.fade-enter-from, .fade-leave-to { opacity: 0; transform: translateY(-6px); }
.scroll-fade-enter-active { transition: all .3s ease; }
.scroll-fade-enter-from { opacity: 0; max-height: 0; }
.scroll-fade-enter-to   { opacity: 1; max-height: 9999px; }
.scroll-fade-leave-active { transition: all .25s ease; }
.scroll-fade-leave-from   { opacity: 1; max-height: 9999px; }
.scroll-fade-leave-to     { opacity: 0; max-height: 0; }
</style>