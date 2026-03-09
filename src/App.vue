<template>
  <div class="app">
    <h1>💪 Weekly Workout</h1>
    <p class="subtitle">
      Track your progress · Stay consistent · Crush your goals
    </p>

    <!-- WARM UP BANNER -->
    <div class="warmup-banner">
      <h3>
        🔥 Daily Warm-up
        <span class="wu-note">(5–7 min · included in every day)</span>
      </h3>
      <div class="wu-items">
        <span class="wu-tag" v-for="w in warmupPreview" :key="w">{{ w }}</span>
      </div>
    </div>

    <!-- DAYS -->
    <div
      class="day-card"
      v-for="(day, di) in days"
      :key="di"
      :class="{ open: openDay === di }"
    >
      <div class="day-header" @click="toggleDay(di)">
        <div class="day-num">{{ di + 1 }}</div>
        <div class="day-info">
          <div class="day-title">{{ day.title }}</div>
          <div class="day-sub">{{ day.subtitle }}</div>
        </div>
        <div class="prog-ring">
          <svg width="40" height="40" viewBox="0 0 40 40">
            <circle
              cx="20"
              cy="20"
              r="16"
              fill="none"
              stroke="#2e2e3e"
              stroke-width="3.5"
            />
            <circle
              cx="20"
              cy="20"
              r="16"
              fill="none"
              :stroke="dayPct(di) === 100 ? '#34d399' : '#a78bfa'"
              stroke-width="3.5"
              stroke-dasharray="100.5"
              :stroke-dashoffset="100.5 * (1 - dayPct(di) / 100)"
              stroke-linecap="round"
            />
          </svg>
          <div class="pct">{{ dayPct(di) }}%</div>
        </div>
        <div class="chevron">▾</div>
      </div>

      <transition name="scroll-fade">
        <div class="day-body" v-if="openDay === di">
          <template v-for="(sec, si) in day.sections" :key="si">
            <div class="section-label" :class="{ 'warmup-lbl': sec.warmup }">
              {{ sec.label }}
            </div>

            <div
              class="ex-row"
              :class="{ 'warmup-ex': sec.warmup }"
              v-for="(ex, ei) in sec.exercises"
              :key="ei"
            >
              <div class="ex-main">
                <!-- Checkbox -->
                <div
                  class="ex-check"
                  :class="{ done: isExDone(di, si, ei) }"
                  @click="toggleDone(di, si, ei)"
                >
                  <span v-if="isExDone(di, si, ei)" class="check-icon">✓</span>
                </div>

                <!-- Name + badge -->
                <div class="ex-name" :class="{ done: isExDone(di, si, ei) }">
                  {{ ex.name }}
                  <span class="warmup-tag" v-if="sec.warmup">Warm-up</span>
                  <span class="finisher-tag" v-else-if="sec.finisher"
                    >Finisher</span
                  >
                </div>

                <!-- Set pips -->
                <div class="ex-sets" v-if="ex.sets > 1">
                  <div
                    class="set-pip"
                    v-for="s in ex.sets"
                    :key="s"
                    :class="{
                      active:
                        getState(di, si, ei).currentSet === s &&
                        getState(di, si, ei).running,
                      complete: s <= getState(di, si, ei).completedSets,
                    }"
                  >
                    {{ s }}
                  </div>
                </div>

                <!-- Buttons -->
                <div class="ex-btns">
                  <button
                    class="btn btn-detail"
                    :class="{ active: detailOpen(di, si, ei) }"
                    @click.stop="toggleDetail(di, si, ei)"
                  >
                    {{ detailOpen(di, si, ei) ? '▴ hide' : 'ℹ detail' }}
                  </button>
                  <button
                    class="btn btn-stop"
                    v-if="getState(di, si, ei).running"
                    @click="stopSet(di, si, ei)"
                  >
                    ✓ Done
                  </button>
                  <button
                    class="btn btn-start"
                    v-else-if="!isExDone(di, si, ei)"
                    @click="startSet(di, si, ei)"
                  >
                    ▶ Start
                  </button>
                  <span v-else class="done-icon">✅</span>
                </div>
              </div>

              <!-- Timer bar -->
              <div class="timer-bar" v-if="getState(di, si, ei).running">
                <div class="timer-val">
                  {{ fmtTime(getState(di, si, ei).elapsed) }}
                </div>
                <div class="timer-label">
                  <template v-if="ex.sets > 1"
                    >Set {{ getState(di, si, ei).currentSet }} /
                    {{ ex.sets }} &nbsp;·&nbsp;</template
                  >
                  <span v-if="ex.reps">{{ ex.reps }}</span>
                </div>
                <div class="timer-prog-wrap" v-if="ex.targetSec">
                  <div
                    class="timer-prog-bar"
                    :style="{
                      width:
                        Math.min(
                          100,
                          (getState(di, si, ei).elapsed / ex.targetSec) * 100
                        ) + '%',
                    }"
                  ></div>
                </div>
              </div>

              <!-- Detail panel -->
              <transition name="fade">
                <div class="detail-panel" v-if="detailOpen(di, si, ei)">
                  <strong>{{ ex.name }}</strong>
                  {{ ex.detail }}
                  <ul class="detail-tips" v-if="ex.tips && ex.tips.length">
                    <li v-for="t in ex.tips" :key="t">{{ t }}</li>
                  </ul>
                </div>
              </transition>
            </div>

            <!-- Rest indicator -->
            <div
              class="rest-box"
              v-if="sec.rest && si < day.sections.length - 1"
            >
              <div>
                <div class="r-label">Rest between rounds</div>
                <div class="r-sub">Take a breather before the next round</div>
              </div>
              <div class="rest-val">{{ sec.rest }}</div>
            </div>
          </template>

          <!-- Completion banner -->
          <div class="complete-banner" v-if="dayPct(di) === 100">
            <div>🎉</div>
            <div>Day {{ di + 1 }} Complete! Amazing work!</div>
          </div>
        </div>
      </transition>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref } from 'vue';

// ─── Exercise Detail Library ───────────────────────────────────────────────
const exDetails = {
  'March in place': {
    detail:
      'Lift your knees to hip height while swinging arms naturally. Keep your core engaged and maintain an upright posture.',
    tips: ['Pump arms for extra warm-up', 'Breathe rhythmically'],
  },
  'Arm circles': {
    detail:
      'Extend arms to sides at shoulder height and make slow controlled circles, then reverse direction.',
    tips: [
      'Start small, gradually increase circle size',
      'Keep shoulders relaxed, not shrugged',
    ],
  },
  'Bodyweight squats': {
    detail:
      'Stand feet shoulder-width apart, lower until thighs are parallel to the floor, then drive through heels to stand.',
    tips: [
      'Keep chest upright',
      'Knees track over toes',
      'Engage core throughout',
    ],
  },
  'Glute bridges': {
    detail:
      'Lie on back, feet flat on floor hip-width apart. Drive hips up by squeezing glutes until body forms a straight line from shoulders to knees.',
    tips: [
      'Hold at the top for 1 sec',
      'Press heels into floor',
      "Don't hyperextend lower back",
    ],
  },
  'Hip circles': {
    detail:
      'Stand with hands on hips and make large slow circles with your hips, engaging the hip flexors and glutes.',
    tips: ['Move slowly and deliberately', 'Alternate directions each set'],
  },
  'Goblet squats': {
    detail:
      'Hold a dumbbell vertically at chest height, feet slightly wider than shoulder-width. Squat deep, keeping elbows inside knees at the bottom.',
    tips: [
      'Use the weight as a counterbalance',
      "Sit into the squat, don't just bend knees",
      'Drive elbows up on the way up',
    ],
  },
  'Dumbbell Romanian deadlift': {
    detail:
      'Hold dumbbells in front of thighs, hinge at hips with slight knee bend, lower weights along legs until you feel a hamstring stretch, then drive hips forward.',
    tips: [
      'Keep back flat, not rounded',
      'Feel the stretch in hamstrings',
      "Think 'push hips back' not 'bend over'",
    ],
  },
  'Band side steps': {
    detail:
      'Place resistance band above knees, slight squat position, step sideways keeping constant tension on the band.',
    tips: [
      "Don't let knees cave inward",
      'Keep steps controlled and even',
      'Maintain squat depth throughout',
    ],
  },
  'Wall sit': {
    detail:
      'Back flat against wall, slide down until thighs are parallel to floor. Hold this isometric position.',
    tips: [
      'Feet shoulder-width, 2 feet from wall',
      "Don't let knees go past toes",
      "Breathe steadily — don't hold your breath",
    ],
  },
  'Slow mountain climbers': {
    detail:
      'Plank position, slowly drive one knee toward chest then return, alternating sides in a controlled manner.',
    tips: [
      "Keep hips level — don't pike up",
      'Count each knee drive as one rep',
      'Engage core the entire time',
    ],
  },
  'Dumbbell shoulder press': {
    detail:
      'Sit or stand, dumbbells at shoulder height, palms forward. Press overhead until arms are nearly straight, then lower with control.',
    tips: [
      "Don't arch your lower back",
      'Exhale on the press',
      'Keep core braced',
    ],
  },
  'Dumbbell rows': {
    detail:
      'Hinge forward at hips, dumbbell in each hand hanging down. Pull elbows back and up, squeezing shoulder blades together at the top.',
    tips: [
      'Keep back flat',
      'Lead with the elbow, not the hand',
      'Full range of motion',
    ],
  },
  'Chest press (floor)': {
    detail:
      'Lie on back, knees bent, dumbbells at chest level with elbows at 45°. Press up until arms are extended, then lower slowly.',
    tips: [
      'The floor limits range of motion safely',
      'Keep wrists straight',
      'Control the lowering phase',
    ],
  },
  'Lateral raises': {
    detail:
      'Stand with dumbbells at sides, slightly bend elbows. Raise arms out to the sides to shoulder height, then lower slowly.',
    tips: [
      "Don't swing the weights",
      "Think 'pour water from a pitcher' — thumbs slightly down",
      '2 sec up, 3 sec down',
    ],
  },
  Plank: {
    detail:
      'Forearms on floor, body in a straight line from head to heels. Hold this position while breathing steadily.',
    tips: [
      "Don't let hips sag or pike up",
      'Squeeze glutes and abs simultaneously',
      'Focus on a spot on the floor',
    ],
  },
  'Dead bug': {
    detail:
      'Lie on back, arms pointing up, knees bent at 90° in the air. Slowly lower opposite arm and leg toward floor, return, repeat.',
    tips: [
      'Press lower back into the floor throughout',
      'Move slowly — about 3 sec each way',
      'Exhale as you lower',
    ],
  },
  'Sumo squats': {
    detail:
      'Stand with feet wider than shoulder-width, toes turned out 45°. Squat down keeping chest up and knees tracking over toes.',
    tips: [
      'Greater inner thigh activation than regular squats',
      'Drive knees out on the way up',
      'Keep torso upright',
    ],
  },
  'Hip thrust (sofa)': {
    detail:
      'Upper back on sofa edge, feet flat on floor, dumbbell or hands on hips. Drive hips up squeezing glutes, hold, lower slowly.',
    tips: [
      'Chin tucked to chest throughout',
      'Squeeze hard at the top',
      "Don't let hips drop to the side",
    ],
  },
  'Glute bridge hold': {
    detail:
      'Same as glute bridge but hold the top position for the full duration without lowering.',
    tips: [
      'Squeeze glutes as hard as possible',
      'Breathe in steady rhythm',
      'Press knees out slightly if using a band',
    ],
  },
  'Band side walks': {
    detail:
      'Band above knees, slight squat stance. Walk sideways for the prescribed steps, keeping tension on the band throughout.',
    tips: [
      'Take full, deliberate steps',
      "Don't let feet come together completely",
      'Keep low the entire time',
    ],
  },
  'Plank shoulder taps': {
    detail:
      'High plank position. Lift one hand to tap the opposite shoulder, return, alternate sides. Keep hips as still as possible.',
    tips: [
      'Widen your feet slightly for stability',
      'Brace core before each tap',
      'Count each tap as one rep',
    ],
  },
  'Standing knee raises': {
    detail:
      'Stand tall, alternate driving knees up to hip height or higher. Pump opposite arm for balance and heart rate elevation.',
    tips: [
      'Maintain upright posture',
      "Drive knees up — don't lean forward",
      'Add arm pumps for intensity',
    ],
  },
  'Wall push-ups': {
    detail:
      "Stand facing wall at arm's length. Place hands on wall at shoulder height. Bend elbows to bring chest toward wall, then push back.",
    tips: [
      'Keep body straight like a plank',
      'Elbows at 45° angle to body',
      'Great low-impact upper body option',
    ],
  },
  'Brisk marching': {
    detail:
      'March in place or across the room at a brisk pace, lifting knees high and pumping arms energetically.',
    tips: [
      'Aim for 100–120 steps per minute',
      'Stay light on your feet',
      'Focus on breathing rhythm',
    ],
  },
  'Step-touch side steps': {
    detail:
      'Step one foot to the side, bring the other to meet it, rhythmically alternating sides. Add arms for extra movement.',
    tips: [
      'Keep movement smooth and rhythmic',
      'Add a gentle clap or arm raise',
      'Good low-impact cardio option',
    ],
  },
  Stretch: {
    detail:
      'Spend 10–12 minutes doing full-body static stretches. Focus on quads, hamstrings, hip flexors, glutes, chest and shoulders.',
    tips: [
      'Hold each stretch 20–30 sec',
      'Breathe deeply and relax into each stretch',
      'Never stretch to the point of pain',
    ],
  },
};

// ─── Helpers ───────────────────────────────────────────────────────────────
function makeEx(name, sets, reps, targetSec = null) {
  const d = exDetails[name] || {
    detail: 'A great exercise for strength and muscle activation.',
    tips: [],
  };
  return {
    name,
    sets: sets || 1,
    reps,
    targetSec,
    detail: d.detail,
    tips: d.tips,
  };
}

// ─── Warm-up (shared across all days) ──────────────────────────────────────
const warmupSection = {
  label: '🔥 Daily Warm-up',
  warmup: true,
  exercises: [
    makeEx('March in place', 1, '2 min', 120),
    makeEx('Arm circles', 1, '1 min', 60),
    makeEx('Bodyweight squats', 1, '15 reps'),
    makeEx('Glute bridges', 1, '15 reps'),
    makeEx('Hip circles', 1, '1 min', 60),
  ],
};

const warmupPreview = [
  'March in place – 2 min',
  'Arm circles – 1 min',
  'Bodyweight squats – 15',
  'Glute bridges – 15',
  'Hip circles – 1 min',
];

// ─── Weekly Plan ───────────────────────────────────────────────────────────
const days = [
  {
    title: 'Day 1 – Glutes + Legs',
    subtitle: 'Warm-up + 4 supersets · ~45 min',
    sections: [
      warmupSection,
      {
        label: 'Superset A',
        exercises: [
          makeEx('Goblet squats', 4, '12 reps'),
          makeEx('Glute bridges', 4, '20 reps'),
        ],
      },
      {
        label: 'Superset B',
        exercises: [
          makeEx('Dumbbell Romanian deadlift', 4, '12 reps'),
          makeEx('Band side steps', 4, '20 reps'),
        ],
      },
      {
        label: 'Finisher 🏁',
        finisher: true,
        exercises: [
          makeEx('Wall sit', 3, '45 sec', 45),
          makeEx('Slow mountain climbers', 3, '30 sec', 30),
        ],
      },
    ],
  },
  {
    title: 'Day 2 – Upper Body + Core',
    subtitle: 'Warm-up + 3 supersets + core',
    sections: [
      warmupSection,
      {
        label: 'Superset A',
        exercises: [
          makeEx('Dumbbell shoulder press', 3, '12 reps'),
          makeEx('Dumbbell rows', 3, '12 reps'),
        ],
      },
      {
        label: 'Superset B',
        exercises: [
          makeEx('Chest press (floor)', 3, '12 reps'),
          makeEx('Lateral raises', 3, '15 reps'),
        ],
      },
      {
        label: 'Core',
        exercises: [
          makeEx('Plank', 3, '40 sec', 40),
          makeEx('Dead bug', 3, '12 each side'),
        ],
      },
      {
        label: 'Finisher 🏁',
        finisher: true,
        exercises: [makeEx('Brisk marching', 1, '5–7 min')],
      },
    ],
  },
  {
    title: 'Day 3 – Fat-Burn Conditioning',
    subtitle: 'Warm-up + 4 rounds · Low impact',
    sections: [
      warmupSection,
      {
        label: 'Circuit × 4 Rounds',
        rest: '60 sec',
        exercises: [
          makeEx('Bodyweight squats', 4, '20 reps'),
          makeEx('Glute bridges', 4, '20 reps'),
          makeEx('Standing knee raises', 4, '30 reps'),
          makeEx('Wall push-ups', 4, '15 reps'),
          makeEx('Plank', 4, '30 sec', 30),
        ],
      },
    ],
  },
  {
    title: 'Day 4 – Glute Focus',
    subtitle: 'Warm-up + 2 supersets + finisher',
    sections: [
      warmupSection,
      {
        label: 'Superset A',
        exercises: [
          makeEx('Sumo squats', 4, '12 reps'),
          makeEx('Hip thrust (sofa)', 4, '15 reps'),
        ],
      },
      {
        label: 'Superset B',
        exercises: [
          makeEx('Dumbbell Romanian deadlift', 3, '12 reps'),
          makeEx('Glute bridge hold', 3, '40 sec', 40),
        ],
      },
      {
        label: 'Finisher 🏁',
        finisher: true,
        exercises: [makeEx('Band side walks', 3, '25 steps')],
      },
    ],
  },
  {
    title: 'Day 5 – Full Body Strength',
    subtitle: 'Warm-up + Circuit × 4 rounds',
    sections: [
      warmupSection,
      {
        label: 'Circuit × 4 Rounds',
        rest: '45 sec',
        exercises: [
          makeEx('Goblet squats', 4, '12 reps'),
          makeEx('Dumbbell rows', 4, '12 reps'),
          makeEx('Dumbbell shoulder press', 4, '12 reps'),
          makeEx('Glute bridges', 4, '20 reps'),
          makeEx('Plank shoulder taps', 4, '20 reps'),
        ],
      },
    ],
  },
  {
    title: 'Day 6 – Low-Cortisol Fat Burn',
    subtitle: 'Warm-up + 30 min continuous + stretch',
    sections: [
      warmupSection,
      {
        label: 'Continuous 30 min · Alternate every minute',
        exercises: [
          makeEx('Brisk marching', 1, '1 min', 60),
          makeEx('Bodyweight squats', 1, '12 reps'),
          makeEx('Step-touch side steps', 1, '1 min', 60),
          makeEx('Standing knee raises', 1, '1 min', 60),
        ],
      },
      { label: 'Cool Down', exercises: [makeEx('Stretch', 1, '10–12 min')] },
    ],
  },
  {
    title: 'Day 7 – Recovery',
    subtitle: 'Warm-up + light yoga + walking',
    sections: [
      warmupSection,
      {
        label: 'Recovery',
        exercises: [
          makeEx('Brisk marching', 1, '20–30 min'),
          makeEx('Hip circles', 1, '5 min', 300),
          makeEx('Arm circles', 1, '2 min', 120),
        ],
      },
    ],
  },
];

// ─── State ─────────────────────────────────────────────────────────────────
const openDay = ref(null);
const exState = reactive({}); // keyed by "di-si-ei"
const detailKeys = reactive({});
const timers = {};

function key(di, si, ei) {
  return `${di}-${si}-${ei}`;
}

function getState(di, si, ei) {
  const k = key(di, si, ei);
  if (!exState[k])
    exState[k] = {
      running: false,
      elapsed: 0,
      currentSet: 0,
      completedSets: 0,
      done: false,
    };
  return exState[k];
}

function detailOpen(di, si, ei) {
  return !!detailKeys[key(di, si, ei)];
}
function toggleDetail(di, si, ei) {
  const k = key(di, si, ei);
  detailKeys[k] = !detailKeys[k];
}
function toggleDay(di) {
  openDay.value = openDay.value === di ? null : di;
}

function startSet(di, si, ei) {
  const s = getState(di, si, ei);
  if (s.running) return;
  s.running = true;
  s.currentSet = s.completedSets + 1;
  s.elapsed = 0;
  const k = key(di, si, ei);
  clearInterval(timers[k]);
  timers[k] = setInterval(() => {
    s.elapsed++;
  }, 1000);
}

function stopSet(di, si, ei) {
  const s = getState(di, si, ei);
  s.running = false;
  clearInterval(timers[key(di, si, ei)]);
  s.completedSets++;
  const ex = days[di].sections[si].exercises[ei];
  if (s.completedSets >= ex.sets) s.done = true;
}

function toggleDone(di, si, ei) {
  const s = getState(di, si, ei);
  if (s.running) stopSet(di, si, ei);
  else s.done = !s.done;
}

function isExDone(di, si, ei) {
  return !!getState(di, si, ei).done;
}

function dayPct(di) {
  let total = 0,
    done = 0;
  days[di].sections.forEach((sec, si) => {
    sec.exercises.forEach((_, ei) => {
      total++;
      if (isExDone(di, si, ei)) done++;
    });
  });
  return total === 0 ? 0 : Math.round((done / total) * 100);
}

function fmtTime(s) {
  return `${String(Math.floor(s / 60)).padStart(2, '0')}:${String(
    s % 60
  ).padStart(2, '0')}`;
}
</script>

<style scoped>
:root {
  --bg: #0f0f13;
  --card: #1a1a24;
  --card2: #22222f;
  --accent: #a78bfa;
  --accent2: #7c3aed;
  --green: #34d399;
  --red: #f87171;
  --yellow: #fbbf24;
  --text: #e2e8f0;
  --muted: #94a3b8;
  --border: #2e2e3e;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.app {
  max-width: 780px;
  margin: 0 auto;
  padding: 20px 16px 60px;
  background: #0f0f13;
  min-height: 100vh;
  color: #e2e8f0;
  font-family: 'Segoe UI', sans-serif;
}

h1 {
  text-align: center;
  font-size: 1.8rem;
  font-weight: 800;
  background: linear-gradient(135deg, #a78bfa, #60a5fa);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  margin-bottom: 4px;
}
.subtitle {
  text-align: center;
  color: #94a3b8;
  font-size: 0.85rem;
  margin-bottom: 28px;
}

/* WARM UP BANNER */
.warmup-banner {
  background: linear-gradient(135deg, #1e1b4b, #312e81);
  border: 1px solid #4338ca;
  border-radius: 16px;
  padding: 16px 20px;
  margin-bottom: 24px;
}
.warmup-banner h3 {
  color: #a5b4fc;
  font-size: 0.95rem;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 8px;
}
.wu-note {
  color: #6366f1;
  font-size: 0.75rem;
  font-weight: 400;
}
.wu-items {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.wu-tag {
  background: #312e81;
  border: 1px solid #4338ca;
  border-radius: 20px;
  padding: 4px 12px;
  font-size: 0.78rem;
  color: #c7d2fe;
}

/* DAY CARD */
.day-card {
  background: #1a1a24;
  border: 1px solid #2e2e3e;
  border-radius: 16px;
  margin-bottom: 12px;
  overflow: hidden;
  transition: box-shadow 0.3s;
}
.day-card.open {
  box-shadow: 0 0 0 1px #7c3aed, 0 8px 32px #7c3aed22;
}
.day-header {
  display: flex;
  align-items: center;
  padding: 16px 20px;
  cursor: pointer;
  gap: 14px;
  user-select: none;
}
.day-num {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: linear-gradient(135deg, #7c3aed, #4f46e5);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  font-size: 0.9rem;
  flex-shrink: 0;
}
.day-info {
  flex: 1;
}
.day-title {
  font-weight: 700;
  font-size: 1rem;
}
.day-sub {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 2px;
}
.prog-ring {
  position: relative;
  width: 40px;
  height: 40px;
}
.prog-ring svg {
  transform: rotate(-90deg);
}
.pct {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.65rem;
  font-weight: 700;
  color: #a78bfa;
}
.chevron {
  color: #94a3b8;
  transition: transform 0.3s;
  font-size: 1.1rem;
}
.open .chevron {
  transform: rotate(180deg);
}

/* BODY */
.day-body {
  padding: 0 16px 16px;
}

/* SECTION LABEL */
.section-label {
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  color: #a78bfa;
  margin: 14px 0 8px;
  padding-left: 4px;
}
.section-label.warmup-lbl {
  color: #818cf8;
}

/* EXERCISE ROW */
.ex-row {
  background: #22222f;
  border: 1px solid #2e2e3e;
  border-radius: 12px;
  margin-bottom: 8px;
  overflow: hidden;
}
.ex-row.warmup-ex {
  border-color: #3730a380;
}
.ex-main {
  display: flex;
  align-items: center;
  padding: 12px 14px;
  gap: 10px;
}

.ex-check {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  border: 2px solid #2e2e3e;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  cursor: pointer;
  transition: all 0.2s;
}
.ex-check.done {
  background: #34d399;
  border-color: #34d399;
}
.check-icon {
  color: #fff;
  font-size: 0.8rem;
}

.ex-name {
  flex: 1;
  font-size: 0.9rem;
  font-weight: 600;
}
.ex-name.done {
  text-decoration: line-through;
  color: #94a3b8;
}

.warmup-tag {
  display: inline-block;
  font-size: 0.65rem;
  font-weight: 700;
  letter-spacing: 1px;
  text-transform: uppercase;
  background: #3730a330;
  color: #818cf8;
  border: 1px solid #3730a360;
  border-radius: 6px;
  padding: 1px 7px;
  margin-left: 8px;
  vertical-align: middle;
}
.finisher-tag {
  display: inline-block;
  font-size: 0.65rem;
  font-weight: 700;
  letter-spacing: 1px;
  text-transform: uppercase;
  background: #fbbf2420;
  color: #fbbf24;
  border: 1px solid #fbbf2440;
  border-radius: 6px;
  padding: 1px 7px;
  margin-left: 8px;
  vertical-align: middle;
}

.ex-sets {
  display: flex;
  gap: 5px;
  flex-wrap: wrap;
}
.set-pip {
  width: 22px;
  height: 22px;
  border-radius: 6px;
  border: 2px solid #2e2e3e;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.62rem;
  font-weight: 700;
  color: #94a3b8;
  transition: all 0.2s;
}
.set-pip.active {
  border-color: #fbbf24;
  color: #fbbf24;
  background: #fbbf2420;
  animation: pulse 1s infinite;
}
.set-pip.complete {
  background: #34d399;
  border-color: #34d399;
  color: #fff;
}
@keyframes pulse {
  0%,
  100% {
    box-shadow: 0 0 0 0 #fbbf2455;
  }
  50% {
    box-shadow: 0 0 0 5px #fbbf2400;
  }
}

.ex-btns {
  display: flex;
  gap: 6px;
  align-items: center;
}
.btn {
  border: none;
  border-radius: 8px;
  padding: 6px 11px;
  font-size: 0.75rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.15s;
}
.btn-start {
  background: linear-gradient(135deg, #7c3aed, #4f46e5);
  color: #fff;
}
.btn-start:hover {
  filter: brightness(1.15);
}
.btn-stop {
  background: linear-gradient(135deg, #f87171, #dc2626);
  color: #fff;
}
.btn-stop:hover {
  filter: brightness(1.15);
}
.btn-detail {
  background: transparent;
  border: 1px solid #2e2e3e;
  color: #94a3b8;
  padding: 5px 9px;
  font-size: 0.72rem;
}
.btn-detail:hover,
.btn-detail.active {
  border-color: #a78bfa;
  color: #a78bfa;
}
.done-icon {
  font-size: 1.1rem;
}

/* TIMER */
.timer-bar {
  display: flex;
  align-items: center;
  gap: 10px;
  background: #fbbf2410;
  border-top: 1px solid #fbbf2430;
  padding: 8px 14px;
}
.timer-val {
  font-size: 1.2rem;
  font-weight: 800;
  color: #fbbf24;
  font-variant-numeric: tabular-nums;
  min-width: 54px;
}
.timer-label {
  font-size: 0.75rem;
  color: #94a3b8;
  flex: 1;
}
.timer-prog-wrap {
  flex: 1;
  height: 4px;
  background: #2e2e3e;
  border-radius: 2px;
}
.timer-prog-bar {
  height: 100%;
  background: #fbbf24;
  border-radius: 2px;
  transition: width 0.5s linear;
}

/* DETAIL */
.detail-panel {
  border-top: 1px solid #2e2e3e;
  padding: 12px 14px;
  background: #151520;
  font-size: 0.82rem;
  color: #94a3b8;
  line-height: 1.6;
}
.detail-panel strong {
  color: #a78bfa;
  display: block;
  margin-bottom: 4px;
}
.detail-tips {
  margin-top: 8px;
}
.detail-tips li {
  margin-left: 16px;
  margin-bottom: 2px;
}

/* REST */
.rest-box {
  background: #0f2820;
  border: 1px solid #34d39940;
  border-radius: 12px;
  padding: 12px 16px;
  margin-top: 10px;
  display: flex;
  align-items: center;
  gap: 12px;
}
.r-label {
  font-size: 0.8rem;
  color: #34d399;
  font-weight: 700;
}
.r-sub {
  font-size: 0.75rem;
  color: #94a3b8;
}
.rest-val {
  font-size: 1.4rem;
  font-weight: 800;
  color: #34d399;
}

/* COMPLETE */
.complete-banner {
  text-align: center;
  padding: 18px;
  background: linear-gradient(135deg, #052e16, #14532d);
  border: 1px solid #34d399;
  border-radius: 12px;
  margin-top: 12px;
}
.complete-banner div:first-child {
  font-size: 2rem;
}
.complete-banner div:last-child {
  color: #34d399;
  font-weight: 700;
  font-size: 1rem;
  margin-top: 4px;
}

/* TRANSITIONS */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.25s, transform 0.25s;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translateY(-6px);
}

.scroll-fade-enter-active {
  transition: all 0.3s ease;
}
.scroll-fade-enter-from {
  opacity: 0;
  max-height: 0;
}
.scroll-fade-enter-to {
  opacity: 1;
  max-height: 9999px;
}
.scroll-fade-leave-active {
  transition: all 0.25s ease;
}
.scroll-fade-leave-from {
  opacity: 1;
  max-height: 9999px;
}
.scroll-fade-leave-to {
  opacity: 0;
  max-height: 0;
}
</style>
