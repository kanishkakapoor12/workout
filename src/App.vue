<template>
  <div class="app">
    <h1>🌸 My Workout Plan</h1>
    <p class="subtitle">5 days · 25 min · High impact · Desk-job optimised</p>

    <!-- CYCLE TOGGLE -->
    <div class="cycle-toggle">
      <button :class="['cycle-btn', { active: cycle === 'high' }]" @click="setCycle('high')">
        🔥 High Impact
        <span class="cycle-sub">Start of month</span>
      </button>
      <button :class="['cycle-btn', cycle === 'low' ? 'active-low' : '']" @click="setCycle('low')">
        🌿 Low Impact
        <span class="cycle-sub">10 days before period</span>
      </button>
    </div>

    <div :class="['cycle-banner', cycle === 'low' ? 'low' : '']">
      <span>{{ cycle === 'high' ? '🔥' : '🌿' }}</span>
      <div v-if="cycle === 'high'">
        <strong>High Impact mode.</strong> Expect to sweat hard in 15 minutes. If a jump feels like too much on any day, step instead — the circuit still works.
      </div>
      <div v-else>
        <strong>Low Impact mode.</strong> No jumping. Still intense via slower reps, longer holds, and resistance. Your body still changes — cortisol just stays low.
      </div>
    </div>

    <!-- DESK NOTE -->
    <div class="desk-banner">
      <div class="desk-title">🪑 You have a desk job — here's what that means for your workouts</div>
      <div class="desk-grid">
        <div class="desk-item"><span>😴</span><span>Glutes shut off after 30 min of sitting. Every session starts with glute activation.</span></div>
        <div class="desk-item"><span>🔒</span><span>Hip flexors tighten from sitting all day. Cool-down stretches every session are non-negotiable.</span></div>
        <div class="desk-item"><span>🐢</span><span>Metabolism drops 90% while seated. Morning workouts keep it elevated all day at your desk.</span></div>
        <div class="desk-item"><span>📌</span><span>Also try: stand up every 45 min at work, do 10 calf raises while waiting for anything.</span></div>
      </div>
    </div>

    <!-- DAYS -->
    <div class="day-card" v-for="(day, di) in activeDays" :key="di + cycle" :class="{ open: openDay === di }">
      <div class="day-header" @click="toggleDay(di)">
        <div class="day-num">D{{ di + 1 }}</div>
        <div class="day-info">
          <div class="day-title">{{ day.title }}</div>
          <div class="day-sub">{{ day.subtitle }}</div>
        </div>
        <div class="day-meta">
          <div class="cal-badge">🔥 {{ day.calories }}</div>
          <div class="time-badge">⏱ {{ day.duration }}</div>
        </div>
        <div class="prog-wrap">
          <div class="prog-ring">
            <svg width="38" height="38" viewBox="0 0 40 40">
              <circle cx="20" cy="20" r="16" fill="none" stroke="#2e2e3e" stroke-width="3.5"/>
              <circle cx="20" cy="20" r="16" fill="none"
                :stroke="dayPct(di) === 100 ? '#34d399' : (cycle==='high' ? '#f472b6' : '#4ade80')"
                stroke-width="3.5" stroke-dasharray="100.5"
                :stroke-dashoffset="100.5*(1-dayPct(di)/100)"
                stroke-linecap="round"/>
            </svg>
            <div class="pct">{{ dayPct(di) }}%</div>
          </div>
        </div>
        <div class="chevron">▾</div>
      </div>

      <transition name="scroll-fade">
        <div class="day-body" v-if="openDay === di">
          <div class="day-note">💡 {{ day.note }}</div>

          <template v-for="(sec, si) in day.sections" :key="si">
            <div class="section-label" :class="{
              'warmup-lbl':   sec.warmup,
              'cooldown-lbl': sec.cooldown,
              'tabata-lbl':   sec.tabata
            }">
              {{ sec.label }}
              <span class="tabata-pill" v-if="sec.tabata">40s ON · 20s REST</span>
            </div>

            <div class="circuit-meta" v-if="sec.rounds">
              🔄 {{ sec.rounds }} round{{ sec.rounds > 1 ? 's' : '' }}
              <template v-if="sec.rest"> · {{ sec.rest }} rest between rounds</template>
            </div>

            <div class="ex-row"
              :class="{ 'warmup-ex': sec.warmup, 'cooldown-ex': sec.cooldown, 'tabata-ex': sec.tabata }"
              v-for="(ex, ei) in sec.exercises" :key="ei">
              <div class="ex-main">
                <div class="ex-check" :class="{ done: isExDone(di,si,ei) }" @click="toggleDone(di,si,ei)">
                  <span v-if="isExDone(di,si,ei)" class="check-icon">✓</span>
                </div>
                <div class="ex-col">
                  <div class="ex-name" :class="{ done: isExDone(di,si,ei) }">
                    {{ ex.name }}
                    <span class="tag warmup-tag"   v-if="sec.warmup">Warm-up</span>
                    <span class="tag cooldown-tag" v-else-if="sec.cooldown">Cool-down</span>
                  </div>
                  <div class="ex-reps-line">{{ ex.reps }}</div>
                </div>
                <div class="ex-sets" v-if="ex.sets > 1">
                  <div class="set-pip" v-for="s in ex.sets" :key="s"
                    :class="{
                      active:   getState(di,si,ei).currentSet===s && getState(di,si,ei).running,
                      complete: s <= getState(di,si,ei).completedSets
                    }">{{ s }}</div>
                </div>
                <div class="ex-btns">
                  <button class="btn btn-detail" :class="{ active: detailOpen(di,si,ei) }" @click.stop="toggleDetail(di,si,ei)">ℹ</button>
                  <button class="btn btn-stop"  v-if="getState(di,si,ei).running"  @click="stopSet(di,si,ei)">✓</button>
                  <button class="btn btn-start" v-else-if="!isExDone(di,si,ei)"   @click="startSet(di,si,ei)">▶</button>
                  <span v-else class="done-icon">✅</span>
                </div>
              </div>

              <!-- TIMER -->
              <div class="timer-bar" v-if="getState(di,si,ei).running">
                <div class="timer-val">{{ fmtTime(getState(di,si,ei).elapsed) }}</div>
                <div class="timer-label">
                  <template v-if="ex.sets>1">Round {{ getState(di,si,ei).currentSet }}/{{ ex.sets }} ·&nbsp;</template>
                  {{ ex.reps }}
                </div>
                <div class="timer-prog-wrap" v-if="ex.targetSec">
                  <div class="timer-prog-bar" :style="{ width: timerPct(di,si,ei) + '%' }"></div>
                </div>
              </div>

              <!-- DETAIL -->
              <transition name="fade">
                <div class="detail-panel" v-if="detailOpen(di,si,ei)">
                  <strong>{{ ex.name }}</strong>
                  <p>{{ ex.detail }}</p>
                  <ul class="detail-tips" v-if="ex.tips && ex.tips.length">
                    <li v-for="t in ex.tips" :key="t">{{ t }}</li>
                  </ul>
                  <div class="kk-note" v-if="ex.kkNote">🦵 <strong>Knock-knee:</strong> {{ ex.kkNote }}</div>
                  <div class="desk-tip" v-if="ex.deskNote">🪑 <strong>Desk tip:</strong> {{ ex.deskNote }}</div>
                </div>
              </transition>
            </div>
          </template>

          <div class="complete-banner" v-if="dayPct(di) === 100">
            <div>🌸</div>
            <div>Day {{ di+1 }} crushed! {{ day.completeMsg }}</div>
          </div>
        </div>
      </transition>
    </div>

    <!-- RESULTS TIP -->
    <div class="results-box">
      <div class="results-title">📊 What actually moves the scale</div>
      <div class="results-row"><span class="r-icon">🥗</span><div><strong>Protein every meal</strong> — eggs, dal, paneer, curd, chicken. Protein burns 25% of its own calories during digestion and prevents muscle loss.</div></div>
      <div class="results-row"><span class="r-icon">🚫</span><div><strong>Cut liquid calories</strong> — chai with sugar, juices, sodas. These are the #1 hidden reason the scale doesn't move for desk workers.</div></div>
      <div class="results-row"><span class="r-icon">💧</span><div><strong>3L water daily</strong> — dehydration mimics hunger and stalls fat loss. Set a reminder every hour at your desk.</div></div>
      <div class="results-row"><span class="r-icon">😴</span><div><strong>7–8 hrs sleep</strong> — one night of poor sleep raises ghrelin (hunger hormone) by 24%. You can't out-train bad sleep.</div></div>
      <div class="results-row"><span class="r-icon">📅</span><div><strong>Weigh weekly, not daily</strong> — same day, same time, after bathroom. Pre-period you'll gain 1–3 kg of water. It's not fat. It drops after.</div></div>
      <div class="results-row"><span class="r-icon">🪑</span><div><strong>Stand/walk every 45 min at work</strong> — even 2 min of walking burns 22% more than sitting. Use a phone timer.</div></div>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref, computed } from 'vue'

// ─── Exercise Library ─────────────────────────────────────────────────────
const EX = {
  // ── Warm-up
  'March in place':             { detail: 'Lift knees to hip height, pump arms. Gentle full-body activation and hip flexor wake-up.', tips: ['Upright posture', 'Deep rhythmic breathing'], deskNote: 'Counteracts the hip-flexor shortening from sitting all morning.' },
  'Hip circles':                { detail: 'Hands on hips, make large slow circles both directions. Opens the hip joint before squats.', tips: ['10 each direction', 'Move slowly and intentionally'], deskNote: 'One of the best ways to undo hip tightness from a chair.' },
  'Cat-cow stretch':            { detail: 'On hands and knees: inhale drop belly (cow), exhale round spine (cat). Warms up the spine.', tips: ['One breath per movement', 'Never force the range'], deskNote: 'Lubricates the spine after sitting compressed at a desk.' },
  'Clamshells (band)':          { detail: 'Lie on side, band above knees, feet together. Lift top knee like a clamshell. Lower with control.', tips: ['Hips stay stacked', 'Slow both ways — squeeze at top'], kkNote: 'Activates glute medius before leg work — essential for knock knees.', deskNote: 'Wakes up the glutes that have been completely switched off while sitting.' },
  'Arm swings':                 { detail: 'Stand tall, swing both arms forward and back in big arcs, then across the body. Mobilises chest and shoulders.', tips: ['Relax the shoulders', 'Gradually increase the range'], deskNote: 'Opens the chest that collapses forward when hunching at a screen.' },

  // ── High impact – lower body / glutes
  'Squat jumps (sumo)':         { detail: 'Wide sumo stance. Squat deep, explode up, land back in wide squat softly with bent knees. Continuous.', tips: ['Feet stay wide the whole time', 'Push knees OUT on every rep', 'Land toes-first, absorb with bent knees'], kkNote: 'Wide stance landing is safer — knees track outward naturally.' },
  'Lateral squat jumps':        { detail: 'Jump side to side landing in a squat each time. Like a skier. Arms swing for momentum.', tips: ['Land in a squat — not standing', 'Push knees out at every landing', 'Arms help balance'], kkNote: 'Land wide, never narrow. Think "push out" at every landing.' },
  'Sumo squat + side kick':     { detail: 'Sumo squat, then as you rise kick one leg out to the side. Alternate each rep.', tips: ['Wide stance throughout', 'Controlled kick — not a fling', 'Core tight on the kick'], kkNote: 'Wide sumo base protects knees. Side kick activates hip abductors.' },
  'Glute bridge (band) pulses': { detail: 'Band above knees. Bridge up, hold at top and pulse 3 times, lower, repeat. Maximum glute activation.', tips: ['Knees OUT against band throughout', 'Pulse is small — 2-3 inch range at top', 'Squeeze hard every pulse'], kkNote: 'Band pushes knees into correct alignment for the whole set.' },
  'Squat + knee drive':         { detail: 'Sumo squat, then drive one knee up fast as you stand. Alternating knees. Cardio + glute combo.', tips: ['Full squat before each drive', 'Drive knee UP — not across body', 'Keep chest up'], kkNote: 'Go wide on the squat. Drive knee straight up, not inward.' },
  'Step jacks':                 { detail: 'Jumping jack arms with stepping feet — step one foot out then back, alternating. Scale up to full jacks if you feel good.', tips: ['Arms go fully overhead each time', 'Brisk pace for cardio effect', 'Knees slightly bent on step-out'] },
  'Pop squats':                 { detail: 'Feet together, jump out into a wide squat, jump feet back together. Repeat fast.', tips: ['Land soft every time', 'Wide on the out — push knees out', 'Arms swing overhead for momentum'], kkNote: 'Land wide every time — never narrow.' },
  'Reverse lunge + knee up':    { detail: 'Step one foot back into a lunge, then drive that knee forward and up fast as you return. Alternating.', tips: ['Front knee stays over ankle', 'Drive the back knee UP on return — that\'s the power move', 'Slow lunge, fast knee drive'], kkNote: 'Front knee tracks over toes — use a wide enough step back.' },

  // ── Low impact alternatives
  'Sumo squat slow':            { detail: '4 counts down, 2 counts hold at bottom, 2 counts up. Maximum time under tension — burns without jumping.', tips: ['Chest tall throughout', 'Push knees OUT hard, especially at bottom', 'Feel the inner thighs and glutes'], kkNote: 'Slow tempo lets you focus completely on knee tracking outward.' },
  'Sumo squat pulse':           { detail: 'Hold bottom of sumo squat and pulse 2–3 inches up and down. 20 pulses.', tips: ['Don\'t come all the way up', 'Knees OUT every single pulse', 'This burns — it\'s working'], kkNote: 'Sustained knee-out tension directly builds the muscles that correct knock knees.' },
  'Glute bridge (band) holds':  { detail: 'Band above knees, bridge up, hold 3 sec at the top, lower slowly. Knees out the whole time.', tips: ['Knees pushed out throughout', '3 sec hold = more glute activation than 10 fast reps', 'Press heels, not toes'], kkNote: 'Isometric hold with active band resistance is ideal for knock-knee correction.' },
  'Clamshells slow (band)':     { detail: 'Band above knees, lie on side. Lift top knee as high as possible, hold 2 sec, lower 3 sec. Ultra slow.', tips: ['Hips completely stacked', '2 sec up, 2 sec hold, 3 sec down', 'Feel the outer hip burning'], kkNote: 'Slow clamshells isolate glute medius — the primary knock-knee corrector.' },
  'Side-step squats (band)':    { detail: 'Band above knees, squat position. Step 3 steps right, 3 steps left. Stay low the entire time.', tips: ['Never stand up between steps', 'Full tension on band throughout', 'Slow deliberate steps burn more'], kkNote: 'Side steps in squat position are the most functional hip abductor exercise.' },
  'Standing torso twist':       { detail: 'Feet hip-width, arms at shoulder height. Rotate torso left and right. Oblique focus.', tips: ['Move from waist — not hips', 'Slow and deliberate', 'Hold a light dumbbell for extra work'] },
  'Wall sit (band)':            { detail: 'Band above knees, back against wall, thighs parallel. Push knees OUT against band and hold.', tips: ['Actively push knees out the whole time', 'Breathe steadily', 'This is glute medius + quad combined'], kkNote: 'Active knee-out hold under load is exactly what builds knock-knee correction strength.' },

  // ── Upper body
  'Wall push-up':               { detail: 'Stand arm\'s length from wall. Bend elbows to bring chest to wall, push back. Full upper body toner.', tips: ['Body straight like a plank', 'Elbows 45° — not flared wide', '3 sec lowering phase = more muscle'] },
  'Dumbbell shoulder press':    { detail: 'Seated or standing, dumbbells at shoulders. Press overhead, lower with control.', tips: ["Don't arch lower back", 'Core engaged', 'Light-medium weight'], deskNote: 'Directly counters the rounded-shoulder posture from sitting at a desk.' },
  'Dumbbell bicep curl':        { detail: 'Curl dumbbells up, squeeze at top, lower slowly 3 sec.', tips: ['Elbows pinned to sides', '3 sec lower = more toning', "Don't swing body"] },
  'Dumbbell lateral raise':     { detail: 'Arms at sides, raise to shoulder height, lower slowly.', tips: ["Don't shrug", 'Thumbs slightly down', 'Very light weight'], deskNote: 'Builds the shoulders that collapse inward after hours of typing.' },
  'Band row (seated)':          { detail: 'Sit on mat, band around both feet, pull handles to sides squeezing shoulder blades.', tips: ['Sit tall — this is easy to maintain', 'Squeeze blades at the end', 'Control the return'], deskNote: 'The #1 desk-job exercise. Undoes rounded upper back from screen time.' },
  'Dumbbell chest fly (floor)': { detail: 'Lie on back, knees bent. Dumbbells above chest, slight elbow bend. Open arms wide, return.', tips: ['Keep elbow bend fixed', 'Only as wide as comfortable', 'Very light weight'], deskNote: 'Opens the chest that gets chronically tight from leaning toward a screen.' },
  'Overhead tricep ext.':       { detail: 'Hold one dumbbell with both hands overhead. Lower behind your head, press back up.', tips: ['Keep elbows in — don\'t flare', 'This is a small movement', 'Light weight, full range'] },

  // ── Core
  'Plank (on knees)':           { detail: 'Forearms on mat, knees down, straight line from knees to shoulders. Hold and breathe.', tips: ["Hips don't sag", 'Squeeze glutes + core together', 'Knee plank is just as effective'], deskNote: 'Builds the deep core that gets completely inactive during desk work.' },
  'Plank shoulder taps (knees)':{ detail: 'High plank on knees. Tap one hand to opposite shoulder, alternate. Hips stay still.', tips: ['Wider feet = more stable', 'Core braced before each tap', 'Slow is harder than fast'] },
  'Glute bridge march':         { detail: 'Bridge up with band, hold hips high. Slowly lift one foot an inch off floor, lower, alternate. Hips stay level.', tips: ['Don\'t let hips drop on the march', 'Knees OUT throughout', 'Small movement — very controlled'], deskNote: 'Wakes up each glute independently — critical after a full day of sitting.' },
  'Dead bug (active)':          { detail: 'Lie on back, arms up, knees at 90°. Lower one arm + opposite leg to hover just above floor, return fast, switch. Active pace.', tips: ['Lower back stays on mat', 'Move at a controlled-but-brisk pace', 'Exhale on every extension'] },
  'Sit-up (crunches)':          { detail: 'Lie on back, knees bent. Curl upper body up until shoulder blades clear floor, lower with control.', tips: ['Hands behind head — don\'t pull neck', 'Exhale on the way up', 'Slow lower = more core work'] },

  // ── Cooldown / stretch
  'Hip flexor stretch (kneeling)': { detail: 'Kneel on one knee, step other foot forward. Sink hips forward and down until you feel a deep front-hip stretch. Hold.', tips: ['Keep torso upright', 'Tuck pelvis slightly under for deeper stretch', 'Breathe and let it release'], deskNote: 'THE most important stretch for desk workers. Your hip flexors are chronically shortened.' },
  'Child\'s pose':              { detail: 'Kneel, sit back on heels, arms forward on mat, forehead down. Full back release.', tips: ['Breathe into your lower back', 'Walk hands further for a deeper stretch'], deskNote: 'Decompresses the spine after sitting compressed at a desk all day.' },
  'Lying glute stretch':        { detail: 'On back, figure-4: ankle over opposite thigh. Pull both legs toward chest.', tips: ['The closer you pull, the deeper', 'Let the hip relax fully'], kkNote: 'Stretches glute medius — helps it recover properly.', deskNote: 'Glutes tighten from sitting even when they\'re inactive — stretch them out.' },
  'Chest opener':               { detail: 'Interlace fingers behind your back, squeeze shoulder blades together, lift chest. Hold.', tips: ['Look up slightly', 'Breathe into your chest', 'Feel the front of the shoulders stretch'], deskNote: 'Directly reverses the hunched forward posture of screen work.' },
  'Seated spinal twist':        { detail: 'Sit cross-legged. Place one hand behind you, other on opposite knee. Rotate and look behind. Hold, switch.', tips: ['Sit tall before twisting', 'Twist from the waist — not just the neck', 'Breathe deeply in the twist'], deskNote: 'Decompresses the spine and counteracts the stiffness from sitting in one position.' },
  'Deep breathing':             { detail: 'Inhale 4 counts, hold 2, exhale 6. Activates parasympathetic nervous system. Drops cortisol.', tips: ['Longer exhale is the key', 'This is not optional — do it every session', 'Feel your heart rate settle'] },
}

function makeEx(name, sets, reps, targetSec = null) {
  const d = EX[name] || { detail: 'Great for your goals.', tips: [] }
  return { name, sets: sets||1, reps, targetSec, detail: d.detail, tips: d.tips||[], kkNote: d.kkNote||null, deskNote: d.deskNote||null }
}

const warmupSection = {
  label: '🔥 Warm-up (5 min)', warmup: true,
  exercises: [
    makeEx('March in place',   1, '60 sec', 60),
    makeEx('Hip circles',      1, '45 sec', 45),
    makeEx('Arm swings',       1, '30 sec', 30),
    makeEx('Cat-cow stretch',  1, '45 sec', 45),
    makeEx('Clamshells (band)',1, '10 reps each side'),
  ],
}

const cooldownSection = {
  label: '🌙 Cool-down (5 min)', cooldown: true,
  exercises: [
    makeEx('Hip flexor stretch (kneeling)', 1, '45 sec each side', 90),
    makeEx('Child\'s pose',                 1, '45 sec',           45),
    makeEx('Lying glute stretch',           1, '30 sec each side', 60),
    makeEx('Chest opener',                  1, '30 sec',           30),
    makeEx('Deep breathing',                1, '45 sec',           45),
  ],
}

const warmupPreview = ['March in place – 60s','Hip circles – 45s','Arm swings – 30s','Cat-cow – 45s','Clamshells – 10 each']

// ─── HIGH IMPACT DAYS ─────────────────────────────────────────────────────
const highDays = [
  {
    title: 'Day 1 – Glute Ignition',
    subtitle: 'Lower body · Band required · 25 min',
    calories: '240–300 cal', duration: '25 min',
    note: 'Glutes are your biggest muscle — training them burns the most calories and directly shapes your body. Band on for every set.',
    completeMsg: 'Glutes fired, metabolism boosted. 🔥',
    sections: [
      warmupSection,
      {
        label: '⚡ Tabata Circuit × 4 Rounds', tabata: true, rounds: 4, rest: '60 sec between rounds',
        exercises: [
          makeEx('Squat jumps (sumo)',         4, '40 sec', 40),
          makeEx('Glute bridge (band) pulses', 4, '40 sec', 40),
          makeEx('Lateral squat jumps',        4, '40 sec', 40),
          makeEx('Sumo squat + side kick',     4, '40 sec', 40),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 2 – Upper Body + Posture',
    subtitle: 'Chest · Shoulders · Back · 25 min',
    calories: '180–220 cal', duration: '25 min',
    note: 'After sitting hunched at a desk, your chest is tight and your upper back is weak. Today fixes both while toning your arms and shoulders.',
    completeMsg: 'Posture improved, arms toned. 💪',
    sections: [
      warmupSection,
      {
        label: '💪 Circuit × 3 Rounds', rounds: 3, rest: '45 sec between rounds',
        exercises: [
          makeEx('Wall push-up',               3, '15 reps'),
          makeEx('Dumbbell shoulder press',    3, '12 reps'),
          makeEx('Band row (seated)',          3, '15 reps'),
          makeEx('Dumbbell lateral raise',     3, '12 reps'),
          makeEx('Dumbbell bicep curl',        3, '12 reps'),
          makeEx('Overhead tricep ext.',       3, '12 reps'),
          makeEx('Dumbbell chest fly (floor)', 3, '12 reps'),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 3 – Full Body Burn',
    subtitle: 'Max calorie burn · All muscles · 25 min',
    calories: '280–350 cal', duration: '25 min',
    note: 'Highest calorie burn of the week. Full body circuits keep your heart rate elevated for 15 solid minutes. This is the session that moves the scale.',
    completeMsg: 'Full body done. This is the one that burns for hours after. 🌡️',
    sections: [
      warmupSection,
      {
        label: '🔥 Full Body Tabata × 4 Rounds', tabata: true, rounds: 4, rest: '60 sec between rounds',
        exercises: [
          makeEx('Pop squats',              4, '40 sec', 40),
          makeEx('Wall push-up',            4, '40 sec', 40),
          makeEx('Squat + knee drive',      4, '40 sec', 40),
          makeEx('Step jacks',              4, '40 sec', 40),
          makeEx('Glute bridge (band) pulses', 4, '40 sec', 40),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 4 – Core + Waist',
    subtitle: 'Flat belly focus · Active core · 25 min',
    calories: '180–220 cal', duration: '25 min',
    note: 'Core work that actually flattens the belly — not crunches alone, but deep stabilisation combined with active movement. Desk workers especially need this.',
    completeMsg: 'Core activated. That deep burn = deep muscles working. ✨',
    sections: [
      warmupSection,
      {
        label: '🎯 Core Circuit × 4 Rounds', rounds: 4, rest: '40 sec between rounds',
        exercises: [
          makeEx('Plank (on knees)',            4, '35 sec', 35),
          makeEx('Sit-up (crunches)',           4, '15 reps'),
          makeEx('Glute bridge march',          4, '12 reps each side'),
          makeEx('Dead bug (active)',           4, '10 reps each side'),
          makeEx('Plank shoulder taps (knees)', 4, '20 taps'),
          makeEx('Standing torso twist',        4, '30 sec', 30),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 5 – Legs + Cardio Finisher',
    subtitle: 'End-of-week burn · Legs · Cardio · 25 min',
    calories: '260–320 cal', duration: '25 min',
    note: 'End the week strong. Legs + cardio combo burns calories and shapes the lower body simultaneously. Go as hard as you can — rest day tomorrow.',
    completeMsg: '5 days done! That\'s a whole week. You\'re building something real. 🌸',
    sections: [
      warmupSection,
      {
        label: '🦵 Legs + Cardio Tabata × 4 Rounds', tabata: true, rounds: 4, rest: '60 sec between rounds',
        exercises: [
          makeEx('Reverse lunge + knee up', 4, '40 sec', 40),
          makeEx('Squat jumps (sumo)',      4, '40 sec', 40),
          makeEx('Step jacks',             4, '40 sec', 40),
          makeEx('Sumo squat + side kick', 4, '40 sec', 40),
        ],
      },
      cooldownSection,
    ],
  },
]

// ─── LOW IMPACT DAYS ──────────────────────────────────────────────────────
const lowDays = [
  {
    title: 'Day 1 – Glute Activation',
    subtitle: 'Band work · Slow & heavy · No jumping · 25 min',
    calories: '180–220 cal', duration: '25 min',
    note: 'Slow reps with band = more glute activation than fast reps without. This still shapes your body — your hormones just stay balanced.',
    completeMsg: 'Glutes working, hormones happy. 🌿',
    sections: [
      warmupSection,
      {
        label: '🍑 Glute Circuit × 4 Rounds', rounds: 4, rest: '50 sec between rounds',
        exercises: [
          makeEx('Sumo squat slow',           4, '12 reps'),
          makeEx('Glute bridge (band) holds', 4, '12 reps'),
          makeEx('Sumo squat pulse',          4, '20 pulses'),
          makeEx('Side-step squats (band)',   4, '10 steps each way'),
          makeEx('Clamshells slow (band)',    4, '12 reps each side'),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 2 – Upper Body + Posture',
    subtitle: 'Identical to high impact · No modification needed',
    calories: '170–210 cal', duration: '25 min',
    note: 'Upper body work is already low-impact — no change needed here. Still builds and tones. Your posture will thank you.',
    completeMsg: 'Posture work done. Feel that chest open up. 💪',
    sections: [
      warmupSection,
      {
        label: '💪 Circuit × 3 Rounds', rounds: 3, rest: '45 sec between rounds',
        exercises: [
          makeEx('Wall push-up',               3, '15 reps'),
          makeEx('Dumbbell shoulder press',    3, '12 reps'),
          makeEx('Band row (seated)',          3, '15 reps'),
          makeEx('Dumbbell lateral raise',     3, '12 reps'),
          makeEx('Dumbbell bicep curl',        3, '12 reps'),
          makeEx('Overhead tricep ext.',       3, '12 reps'),
          makeEx('Dumbbell chest fly (floor)', 3, '12 reps'),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 3 – Full Body Strength',
    subtitle: 'Slow circuits · Full body · No jumping · 25 min',
    calories: '190–230 cal', duration: '25 min',
    note: 'Replace jumps with slow, heavy reps. Slow squats under tension burn almost as many calories as jump squats and build more muscle.',
    completeMsg: 'Full body strength done. Strong is the goal. 🌿',
    sections: [
      warmupSection,
      {
        label: '🔄 Full Body Circuit × 4 Rounds', rounds: 4, rest: '55 sec between rounds',
        exercises: [
          makeEx('Sumo squat slow',            4, '12 reps'),
          makeEx('Wall push-up',               4, '15 reps'),
          makeEx('Glute bridge (band) holds',  4, '12 reps'),
          makeEx('Band row (seated)',          4, '15 reps'),
          makeEx('Wall sit (band)',            4, '30 sec', 30),
          makeEx('Standing torso twist',       4, '30 sec', 30),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 4 – Core + Waist',
    subtitle: 'Same as high impact · Core is always low impact',
    calories: '160–200 cal', duration: '25 min',
    note: 'Core work is the same — it\'s always controlled and low-impact. This is your belly-flattening session.',
    completeMsg: 'Core done. Consistent core work is what creates lasting change. ✨',
    sections: [
      warmupSection,
      {
        label: '🎯 Core Circuit × 4 Rounds', rounds: 4, rest: '40 sec between rounds',
        exercises: [
          makeEx('Plank (on knees)',            4, '35 sec', 35),
          makeEx('Sit-up (crunches)',           4, '15 reps'),
          makeEx('Glute bridge march',          4, '12 reps each side'),
          makeEx('Dead bug (active)',           4, '10 reps each side'),
          makeEx('Plank shoulder taps (knees)', 4, '20 taps'),
          makeEx('Standing torso twist',        4, '30 sec', 30),
        ],
      },
      cooldownSection,
    ],
  },
  {
    title: 'Day 5 – Legs + Band Finisher',
    subtitle: 'Strong legs · Band focus · No jumping · 25 min',
    calories: '190–240 cal', duration: '25 min',
    note: 'End of week, low-impact style. Bands make this harder than it looks. Focus on slow squats and glute activation.',
    completeMsg: '5 days done, low-impact week complete. You stayed consistent — that\'s everything. 🌸',
    sections: [
      warmupSection,
      {
        label: '🦵 Legs + Band Circuit × 4 Rounds', rounds: 4, rest: '50 sec between rounds',
        exercises: [
          makeEx('Sumo squat slow',          4, '12 reps'),
          makeEx('Side-step squats (band)',  4, '12 steps each way'),
          makeEx('Sumo squat pulse',         4, '20 pulses'),
          makeEx('Glute bridge (band) holds',4, '15 reps'),
          makeEx('Clamshells slow (band)',   4, '12 reps each side'),
        ],
      },
      cooldownSection,
    ],
  },
]

// ─── State ────────────────────────────────────────────────────────────────
const cycle      = ref('high')
const openDay    = ref(null)
const exState    = reactive({})
const detailKeys = reactive({})
const timers     = {}

const activeDays = computed(() => cycle.value === 'high' ? highDays : lowDays)

function setCycle(c) { cycle.value = c; openDay.value = null }
function key(di,si,ei) { return `${cycle.value}-${di}-${si}-${ei}` }

function getState(di,si,ei) {
  const k = key(di,si,ei)
  if (!exState[k]) exState[k] = { running:false, elapsed:0, currentSet:0, completedSets:0, done:false }
  return exState[k]
}

function detailOpen(di,si,ei) { return !!detailKeys[key(di,si,ei)] }
function toggleDetail(di,si,ei) { const k=key(di,si,ei); detailKeys[k]=!detailKeys[k] }
function toggleDay(di) { openDay.value = openDay.value===di ? null : di }

function startSet(di,si,ei) {
  const s = getState(di,si,ei)
  if (s.running) return
  s.running=true; s.currentSet=s.completedSets+1; s.elapsed=0
  const k=key(di,si,ei); clearInterval(timers[k])
  timers[k] = setInterval(() => { s.elapsed++ }, 1000)
}

function stopSet(di,si,ei) {
  const s = getState(di,si,ei); s.running=false
  clearInterval(timers[key(di,si,ei)]); s.completedSets++
  if (s.completedSets >= activeDays.value[di].sections[si].exercises[ei].sets) s.done=true
}

function toggleDone(di,si,ei) {
  const s = getState(di,si,ei)
  if (s.running) stopSet(di,si,ei); else s.done=!s.done
}

function isExDone(di,si,ei) { return !!getState(di,si,ei).done }

function timerPct(di,si,ei) {
  const s = getState(di,si,ei)
  const ex = activeDays.value[di].sections[si].exercises[ei]
  return ex.targetSec ? Math.min(100, s.elapsed/ex.targetSec*100) : 0
}

function dayPct(di) {
  let total=0, done=0
  activeDays.value[di].sections.forEach((sec,si) => {
    sec.exercises.forEach((_,ei) => { total++; if (isExDone(di,si,ei)) done++ })
  })
  return total===0 ? 0 : Math.round(done/total*100)
}

function fmtTime(s) {
  return `${String(Math.floor(s/60)).padStart(2,'0')}:${String(s%60).padStart(2,'0')}`
}
</script>

<style scoped>
*{ box-sizing:border-box; margin:0; padding:0; }
.app{
  max-width:760px; margin:0 auto; padding:18px 14px 60px;
  background:#0f0f13; min-height:100vh;
  color:#e2e8f0; font-family:'Segoe UI',sans-serif;
}
h1{
  text-align:center; font-size:1.75rem; font-weight:800;
  background:linear-gradient(135deg,#f472b6,#a78bfa);
  -webkit-background-clip:text; -webkit-text-fill-color:transparent; margin-bottom:4px;
}
.subtitle{ text-align:center; color:#94a3b8; font-size:.82rem; margin-bottom:18px; }

/* CYCLE TOGGLE */
.cycle-toggle{ display:flex; gap:10px; margin-bottom:12px; }
.cycle-btn{
  flex:1; background:#1a1a24; border:2px solid #2e2e3e; border-radius:14px;
  padding:11px 10px; cursor:pointer; transition:all .2s; color:#94a3b8;
  font-size:.86rem; font-weight:700; display:flex; flex-direction:column; align-items:center; gap:3px;
}
.cycle-btn.active{ border-color:#f472b6; color:#f9a8d4; background:#1a0a1e; }
.cycle-btn.active-low{ border-color:#4ade80; color:#bbf7d0; background:#0a1a0e; }
.cycle-sub{ font-size:.68rem; font-weight:400; opacity:.7; }

.cycle-banner{
  display:flex; gap:10px; align-items:flex-start; background:#1a0a1e;
  border:1px solid #f472b640; border-radius:12px; padding:11px 14px;
  margin-bottom:16px; font-size:.8rem; color:#f9a8d4; line-height:1.6;
}
.cycle-banner.low{ background:#0a1a0e; border-color:#4ade8040; color:#bbf7d0; }
.cycle-banner span{ font-size:1.15rem; flex-shrink:0; }
.cycle-banner strong{ color:#f472b6; }
.cycle-banner.low strong{ color:#4ade80; }

/* DESK BANNER */
.desk-banner{
  background:#111118; border:1px solid #2e2e3e; border-radius:14px;
  padding:14px 16px; margin-bottom:18px;
}
.desk-title{ font-size:.82rem; font-weight:700; color:#fbbf24; margin-bottom:10px; }
.desk-grid{ display:grid; grid-template-columns:1fr 1fr; gap:8px; }
.desk-item{ display:flex; gap:8px; font-size:.76rem; color:#94a3b8; line-height:1.5; align-items:flex-start; }
.desk-item span:first-child{ font-size:1rem; flex-shrink:0; }
@media(max-width:480px){ .desk-grid{ grid-template-columns:1fr; } }

/* WARMUP BANNER */
.warmup-banner{
  background:linear-gradient(135deg,#1e1b4b,#312e81);
  border:1px solid #4338ca; border-radius:14px; padding:12px 16px; margin-bottom:18px;
}
.warmup-banner h3{ color:#a5b4fc; font-size:.88rem; margin-bottom:8px; }
.wu-note{ color:#6366f1; font-size:.7rem; font-weight:400; }
.wu-items{ display:flex; flex-wrap:wrap; gap:6px; }
.wu-tag{ background:#312e81; border:1px solid #4338ca; border-radius:20px; padding:3px 10px; font-size:.74rem; color:#c7d2fe; }

/* DAY CARD */
.day-card{ background:#1a1a24; border:1px solid #2e2e3e; border-radius:16px; margin-bottom:10px; overflow:hidden; transition:box-shadow .3s; }
.day-card.open{ box-shadow:0 0 0 1px #db2777,0 6px 28px #db277718; }
.day-header{ display:flex; align-items:center; padding:13px 16px; cursor:pointer; gap:11px; user-select:none; }
.day-num{
  width:36px; height:36px; border-radius:50%; flex-shrink:0;
  background:linear-gradient(135deg,#db2777,#9333ea);
  display:flex; align-items:center; justify-content:center; font-weight:800; font-size:.82rem;
}
.day-info{ flex:1; min-width:0; }
.day-title{ font-weight:700; font-size:.94rem; }
.day-sub{ font-size:.7rem; color:#94a3b8; margin-top:1px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.day-meta{ display:flex; flex-direction:column; align-items:flex-end; gap:3px; flex-shrink:0; }
.cal-badge{ font-size:.65rem; color:#fbbf24; background:#fbbf2415; border:1px solid #fbbf2430; border-radius:6px; padding:2px 6px; white-space:nowrap; }
.time-badge{ font-size:.65rem; color:#94a3b8; background:#ffffff08; border:1px solid #2e2e3e; border-radius:6px; padding:2px 6px; }
.prog-wrap{ flex-shrink:0; }
.prog-ring{ position:relative; width:36px; height:36px; }
.prog-ring svg{ transform:rotate(-90deg); }
.pct{ position:absolute; inset:0; display:flex; align-items:center; justify-content:center; font-size:.58rem; font-weight:700; color:#f472b6; }
.chevron{ color:#94a3b8; transition:transform .3s; font-size:.95rem; flex-shrink:0; }
.open .chevron{ transform:rotate(180deg); }

.day-body{ padding:0 13px 13px; }
.day-note{ background:#1e1a2e; border:1px solid #4c1d9580; border-radius:10px; padding:9px 13px; font-size:.79rem; color:#c4b5fd; margin-bottom:4px; line-height:1.5; }

.section-label{ font-size:.68rem; font-weight:700; letter-spacing:1.5px; text-transform:uppercase; color:#f472b6; margin:12px 0 6px; padding-left:3px; display:flex; align-items:center; gap:8px; }
.section-label.warmup-lbl{ color:#818cf8; }
.section-label.cooldown-lbl{ color:#34d399; }
.section-label.tabata-lbl{ color:#fb923c; }
.tabata-pill{ font-size:.6rem; background:#fb923c20; color:#fb923c; border:1px solid #fb923c40; border-radius:6px; padding:1px 7px; letter-spacing:.5px; }

.circuit-meta{ font-size:.73rem; color:#64748b; margin-bottom:7px; padding-left:3px; }

/* EX ROW */
.ex-row{ background:#22222f; border:1px solid #2e2e3e; border-radius:11px; margin-bottom:6px; overflow:hidden; }
.ex-row.warmup-ex{ border-color:#3730a355; }
.ex-row.cooldown-ex{ border-color:#05966940; }
.ex-row.tabata-ex{ border-color:#fb923c30; }
.ex-main{ display:flex; align-items:center; padding:10px 12px; gap:9px; }
.ex-check{ width:21px; height:21px; border-radius:50%; border:2px solid #2e2e3e; display:flex; align-items:center; justify-content:center; flex-shrink:0; cursor:pointer; transition:all .2s; }
.ex-check.done{ background:#34d399; border-color:#34d399; }
.check-icon{ color:#fff; font-size:.75rem; }
.ex-col{ flex:1; min-width:0; }
.ex-name{ font-size:.86rem; font-weight:600; }
.ex-name.done{ text-decoration:line-through; color:#94a3b8; }
.ex-reps-line{ font-size:.72rem; color:#64748b; margin-top:1px; }
.tag{ display:inline-block; font-size:.58rem; font-weight:700; letter-spacing:.8px; text-transform:uppercase; border-radius:5px; padding:1px 5px; margin-left:5px; vertical-align:middle; }
.warmup-tag{ background:#3730a325; color:#818cf8; border:1px solid #3730a350; }
.cooldown-tag{ background:#05966920; color:#34d399; border:1px solid #34d39935; }

.ex-sets{ display:flex; gap:4px; flex-wrap:wrap; flex-shrink:0; }
.set-pip{ width:19px; height:19px; border-radius:5px; border:2px solid #2e2e3e; display:flex; align-items:center; justify-content:center; font-size:.58rem; font-weight:700; color:#94a3b8; transition:all .2s; }
.set-pip.active{ border-color:#fbbf24; color:#fbbf24; background:#fbbf2418; animation:pulse 1s infinite; }
.set-pip.complete{ background:#34d399; border-color:#34d399; color:#fff; }
@keyframes pulse{ 0%,100%{box-shadow:0 0 0 0 #fbbf2450}50%{box-shadow:0 0 0 4px #fbbf2400} }

.ex-btns{ display:flex; gap:5px; align-items:center; flex-shrink:0; }
.btn{ border:none; border-radius:8px; font-weight:700; cursor:pointer; transition:all .15s; }
.btn-start{ background:linear-gradient(135deg,#db2777,#9333ea); color:#fff; padding:6px 12px; font-size:.76rem; }
.btn-start:hover{ filter:brightness(1.15); }
.btn-stop{ background:linear-gradient(135deg,#f87171,#dc2626); color:#fff; padding:6px 12px; font-size:.76rem; }
.btn-stop:hover{ filter:brightness(1.15); }
.btn-detail{ background:transparent; border:1px solid #2e2e3e; color:#64748b; padding:5px 9px; font-size:.74rem; }
.btn-detail:hover,.btn-detail.active{ border-color:#f472b6; color:#f472b6; }
.done-icon{ font-size:.95rem; }

.timer-bar{ display:flex; align-items:center; gap:9px; background:#fb923c12; border-top:1px solid #fb923c30; padding:7px 12px; }
.timer-val{ font-size:1.15rem; font-weight:800; color:#fb923c; font-variant-numeric:tabular-nums; min-width:48px; }
.timer-label{ font-size:.71rem; color:#94a3b8; flex:1; }
.timer-prog-wrap{ flex:1; height:4px; background:#2e2e3e; border-radius:2px; }
.timer-prog-bar{ height:100%; background:#fb923c; border-radius:2px; transition:width .8s linear; }

.detail-panel{ border-top:1px solid #2e2e3e; padding:11px 13px; background:#13131e; font-size:.79rem; color:#94a3b8; line-height:1.65; }
.detail-panel strong{ color:#f472b6; display:block; margin-bottom:5px; font-size:.85rem; }
.detail-panel p{ margin-bottom:6px; }
.detail-tips{ margin:6px 0 0 14px; }
.detail-tips li{ margin-bottom:3px; }
.kk-note{ margin-top:8px; background:#78350f18; border:1px solid #fbbf2430; border-radius:7px; padding:7px 10px; font-size:.76rem; color:#fde68a; line-height:1.5; }
.kk-note strong{ color:#fbbf24; font-size:.76rem; display:inline; }
.desk-tip{ margin-top:7px; background:#0c2a1a; border:1px solid #34d39930; border-radius:7px; padding:7px 10px; font-size:.76rem; color:#6ee7b7; line-height:1.5; }
.desk-tip strong{ color:#34d399; font-size:.76rem; display:inline; }

.complete-banner{ text-align:center; padding:15px; background:linear-gradient(135deg,#1a0030,#3b0764); border:1px solid #f472b6; border-radius:12px; margin-top:10px; }
.complete-banner div:first-child{ font-size:1.8rem; }
.complete-banner div:last-child{ color:#f9a8d4; font-weight:700; font-size:.9rem; margin-top:4px; }

/* RESULTS BOX */
.results-box{ background:#111118; border:1px solid #1e1e2e; border-radius:16px; padding:16px; margin-top:22px; }
.results-title{ font-size:.86rem; font-weight:700; color:#f472b6; margin-bottom:12px; }
.results-row{ display:flex; gap:10px; font-size:.78rem; color:#94a3b8; line-height:1.6; margin-bottom:9px; align-items:flex-start; }
.r-icon{ font-size:1rem; flex-shrink:0; margin-top:1px; }
.results-row strong{ color:#e2e8f0; }

.fade-enter-active,.fade-leave-active{ transition:opacity .2s,transform .2s; }
.fade-enter-from,.fade-leave-to{ opacity:0; transform:translateY(-5px); }
.scroll-fade-enter-active{ transition:all .28s ease; }
.scroll-fade-enter-from{ opacity:0; max-height:0; }
.scroll-fade-enter-to{ opacity:1; max-height:9999px; }
.scroll-fade-leave-active{ transition:all .22s ease; }
.scroll-fade-leave-from{ opacity:1; max-height:9999px; }
.scroll-fade-leave-to{ opacity:0; max-height:0; }
</style>