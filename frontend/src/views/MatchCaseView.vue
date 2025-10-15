<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useResultsStore } from '@/stores/results'
import Layout from '@/layout/Layout.vue'

const router = useRouter()
const resultsStore = useResultsStore()

// Base API URL (จาก .env: VITE_API_URL)
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000'

// ---------- state: คำตอบผู้ใช้ ----------
const userCase = ref(null)

// ข้อมูลที่โหลดจาก backend
const faculties = ref([])
const interestds = ref([])
const subjectGroups = ref([])
const grades = ref([])
const subjects = ref([])
const groupwork = ref([])
const soloWork = ref([])
const exam = ref([])
const attendance = ref([])
const instruction = ref([])
const present = ref([])
const experience = ref([])
const challenge = ref([])
const time = ref([])
const solowork = ref([])

// ตัวแปรที่ผูกกับ v-model
const studentId = ref('')
const selectedStudentLevel = ref('null')
const selectedFaculty = ref('null')
const selectedGrade = ref('')
const selectedInterestd = ref([])
const selectedSubjectGroup = ref('')
const selectedSubject = ref('')
const selectedGroupwork = ref('')
const selectedsolowork = ref('')
const selectedexam = ref('')
const selectedattendance = ref('')
const selectedinstruction = ref([])
const selectedpresent = ref('')
const selectedexperience = ref('')
const selectedchallenge = ref('')
const selectedtime = ref('')

const selectedGroupTypes = ref([])

const resultGroups = ref([])

// ---------- state: ผลลัพธ์/โหลด/เออเรอร์ ----------
const loading = ref(false)
const errorMsg = ref('')
const results = ref([])

// ---------- helpers ----------
function authHeaders() {
  const headers = { 'Content-Type': 'application/json' }
  const token = localStorage.getItem('token')
  if (token) headers.Authorization = `Bearer ${token}`
  return headers
}

// robust: เลือกคีย์คะแนนที่มีอยู่จริง (กันชื่อไม่ตรง)
function pickSimilarity(obj) {
  const keys = ['similarity', 'similarityPct', 'score', 'percent', 'pct', 'Similarity']
  for (const k of keys) {
    if (obj && Object.prototype.hasOwnProperty.call(obj, k) && obj[k] != null) return obj[k]
  }
  return null
}

// ฟังก์ชันช่วยแปลงเลขเป็น %
function pct(v) {
  if (v == null) return '-'
  const num = Number(v)
  if (!Number.isFinite(num)) return '-'
  const p = num > 1 ? num : num * 100 // รองรับกรณี backend ส่ง 0–1
  return p.toFixed(2) + '%'
}

function normalizeGroups(data) {
  const arr = Array.isArray(data) ? data : (data?.items ?? [])
  return arr
    .map(x => ({
      GroupType_ID:   x.GroupType_ID   ?? x.group_type_id ?? x.groupTypeId ?? x.id,
      GroupType_Name: x.GroupType_Name ?? x.group_type_name ?? x.groupTypeName ?? x.name,
    }))
    .filter(x => x.GroupType_ID && x.GroupType_Name)
}

// helper: fetch GET + json + throw on not ok
async function fetchGet(path) {
  const res = await fetch(`${API_URL}${path.startsWith('/') ? '' : '/'}${path}`, {
    method: 'GET',
    headers: authHeaders()
  })
  const j = await res.json().catch(() => null)
  if (!res.ok) {
    throw new Error(j?.message || res.statusText || 'Request failed')
  }
  return j
}

// helper: fetch POST + json + throw on not ok
async function fetchPost(path, body) {
  const res = await fetch(`${API_URL}${path.startsWith('/') ? '' : '/'}${path}`, {
    method: 'POST',
    headers: authHeaders(),
    body: JSON.stringify(body)
  })
  const j = await res.json().catch(() => null)
  if (!res.ok) {
    throw new Error(j?.message || res.statusText || 'Request failed')
  }
  return j
}

// เปิด/ปิดการล็อกดีบักในคอนโซล (ตั้ง false เมื่อปล่อยโปรดักชัน)
const DEBUG_LOG = import.meta.env.DEV || import.meta.env.VITE_DEBUG_LOG === 'true'

function logDebugItem(c) {
  if (!DEBUG_LOG || !c?.dbg) return

  const name = c.subject_Name || `วิชา #${c.subject_ID}`
  const simPct = Number(c.similarity ?? 0).toFixed(2)

  console.groupCollapsed(`DBG: ${name} (similarity ${simPct}%)`)
  console.log('ผู้ใช้กรอก:', c.dbg.user_input)
  console.log('ค่าของเคส (DB):', c.dbg.case_values)

  const contribs = c.dbg.contributions || {}
  const rows = Object.keys(contribs).map((k) => ({
    dim: k,
    sim: Number(c.sims?.[k] ?? 0).toFixed(3),
    weight: contribs[k]?.w,
    ws: contribs[k]?.ws,
    pct: `${contribs[k]?.ws_pct}%`,
  }))
  if (rows.length) console.table(rows)
  else console.log('ไม่มี contributions')

  console.log('สรุป:', c.dbg.sums)
  console.groupEnd()
}

// โหลดข้อมูลตอน mount (เรียกหลาย endpoint พร้อมกัน)
onMounted(async () => {
  try {
    // ตรวจเช็ก user email / login (เหมือนเดิม)
    const email = localStorage.getItem('userEmail')
    if (!email) {
      return router.push({ name: 'login', query: { redirect: '/review' } })
    }

    // เรียก API พร้อมกัน (ใช้ fetchGet helper)
    const paths = [
      '/subject-groups',
      '/faculty',
      '/interestd',
      '/grades',
      '/groupwork',
      '/solowork',
      '/exam',
      '/attendance',
      '/instruction',
      '/present',
      '/experience',
      '/challenge',
      '/time'
    ]

    // ทำ Promise.all ของ fetchGet
    const responses = await Promise.all(paths.map(p => fetchGet(p)))

    // แยกผลตามลำดับเดิม
    const [
      gRes, fRes, iRes, grRes, gwRes, swRes, exRes, attRes, inRes, preRes, expRes, cRes, tRes
    ] = responses

    // กำหนดค่าให้ state
    subjectGroups.value = normalizeGroups(gRes ?? [])
    faculties.value   = fRes ?? []
    interestds.value  = iRes ?? []
    grades.value      = grRes ?? []
    groupwork.value   = gwRes ?? []
    soloWork.value    = swRes ?? []
    exam.value        = exRes ?? []
    attendance.value  = attRes ?? []
    instruction.value = inRes ?? []
    present.value     = preRes ?? []
    experience.value  = expRes ?? []
    challenge.value   = cRes ?? []
    time.value        = tRes ?? []

    // ค่า default จาก localStorage (ของเดิม)
    studentId.value            = localStorage.getItem('student_ID')   || ''
    selectedStudentLevel.value = localStorage.getItem('studentLevel') || ''
    selectedFaculty.value      = localStorage.getItem('facultyId')    || ''
  } catch (err) {
    console.error("โหลดข้อมูลไม่สำเร็จ:", err)
    errorMsg.value = err?.message || 'โหลดข้อมูลเริ่มต้นล้มเหลว'
  }
})

// ---------- submit ----------
async function onSubmit() {
  loading.value = true
  errorMsg.value = ''
  results.value = []
  resultGroups.value = []

  const toD = (v) => /^\d+$/.test(String(v)) ? `D${v}` : String(v)
  const instructionTokens = Array.isArray(selectedinstruction.value)
    ? selectedinstruction.value.map(toD)
    : []

  try {
    const payload = {
      interestd: selectedInterestd.value,
      groupwork: selectedGroupwork.value,
      solowork: selectedsolowork.value,
      exam: selectedexam.value,
      attendance: selectedattendance.value,
      instructions: instructionTokens,
      instruction: instructionTokens[0] || '',
      instruction_CSV: instructionTokens.join(','),
      present: selectedpresent.value,
      experience: selectedexperience.value,
      challenge: selectedchallenge.value,
      time: selectedtime.value,
      group_types: selectedGroupTypes.value,
      debug: true
    }

    console.log('PLYLOAD /cbr-match:', payload)

    // เรียก POST /cbr-match ผ่าน fetchPost helper
    const data = await fetchPost('/cbr-match', payload)

    // รับ groups จาก backend
    resultGroups.value = Array.isArray(data.groups) ? data.groups : []

    // ใช้ top ถ้ามี ไม่งั้นใช้ all
    const raw = (Array.isArray(data.top) && data.top.length ? data.top : data.all) || []

    // ทำ mapping ให้มี field similarity เสมอ
    results.value = raw.map(r => {
      const s = pickSimilarity(r)
      const n = Number(s)
      return { ...r, similarity: Number.isFinite(n) ? n : 0 }
    })

    // เก็บผลลง store และไปหน้าแสดงผล
    resultsStore.setResults({
      resultGroups: resultGroups.value,
      results: results.value,
      payload
    })

    if (DEBUG_LOG) {
      for (const g of resultGroups.value || []) {
        for (const c of g.items || []) {
          logDebugItem(c)
        }
      }
    }

    router.push({ name: 'showresults' })

  } catch (e) {
    console.error('CBR error:', e)
    errorMsg.value = e?.message || 'เกิดข้อผิดพลาดในการประมวลผล'
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <Layout>
    <form class="p-6 space-y-6" @submit.prevent="onSubmit">
      <!-- ความสนใจ -->
      <div class="bg-[#F992AF]/50 p-6 rounded-3xl grid grid-cols-1 md:grid-cols-2 gap-4">
        <div>
          <h2 class="font-bold mb-2">ความสนใจ (เลือกได้มากกว่า 1 คำตอบ)</h2>
          <label class="block" v-for="it in interestds" :key="it.interest_ID" :for="`interest-${it.interest_ID}`">
            <input type="checkbox"
                   class="checkbox checkbox-sm border-pink-400 bg-pink-300 checked:border-pink-700 checked:bg-pink-600"
                   :id="`interest-${it.interest_ID}`"
                   :value="String(it.interest_ID)"
                   v-model="selectedInterestd">
            {{ it.interest_Name }}
          </label>
        </div>

        <div>
          <h2 class="font-bold mb-2">หมวดวิชาศึกษาทั่วไปที่นิสิตจะลงเรียน</h2>
          <label class="block" v-for="g in subjectGroups" :key="g.GroupType_ID">
            <input type="checkbox"
                   class="checkbox checkbox-sm border-pink-400 bg-pink-300 checked:border-pink-700 checked:bg-pink-600"
                   :value="g.GroupType_ID"
                   v-model="selectedGroupTypes">
            {{ g.GroupType_Name }}
          </label>
        </div>
      </div>

      <!-- งานกลุ่ม -->
      <div class="bg-[#FFAE00]/35 p-6 rounded-3xl">
        <h2 class="font-bold mb-3">เลือกคำตอบที่นิสิตคิดว่าตรงกับตนเองมากที่สุด</h2>
        <fieldset class="mb-4 pl-5">
          <legend>
            1. นิสิตต้องการให้มีการมอบหมาย
            <span style="color:red;">งานกลุ่ม</span>
            ในรายวิชาอย่างไร <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in groupwork" :key="o.groupwork_ID">
              <input type="radio" name="groupwork"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.groupwork_ID"
                     v-model="selectedGroupwork">
              {{ o.groupwork_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- งานเดี่ยว -->
      <div class="bg-[#FBCAA8]/95 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            2. นิสิตต้องการให้มีการมอบหมาย
            <span style="color:red;">งานเดี่ยว</span>
            ในรายวิชาอย่างไร <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in soloWork" :key="o.solowork_ID">
              <input type="radio" name="solowork"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.solowork_ID"
                     v-model="selectedsolowork">
              {{ o.solowork_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- การสอบ -->
      <div class="bg-[#F992AF]/50 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            3.นิสิตต้องการให้มีรูปแบบ
            <span style="color:red;">การสอบ</span> แบบใด
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in exam" :key="o.exam_ID">
              <input type="radio" name="exam"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.exam_ID"
                     v-model="selectedexam">
              {{ o.exam_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- เช็คชื่อ -->
      <div class="bg-[#FFAE00]/35 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            4.นิสิตต้องการให้มีการ <span style="color:red;">เช็คชื่อ</span> เข้าห้องเรียนอย่างไร
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in attendance" :key="o.attendance_ID">
              <input type="radio" name="attendance"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.attendance_ID"
                     v-model="selectedattendance">
              {{ o.attendance_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- การสอน -->
      <div class="bg-[#FBCAA8]/95 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            5.นิสิตต้องการให้รูปแบบ
            <span style="color:red;">การสอน</span> เป็นอย่างไร (ตอบได้มากกว่า 1 ข้อ)
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in instruction" :key="o.instruction_ID" :for="`inst-${o.instruction_ID}`">
              <input type="checkbox"
                     class="checkbox checkbox-sm border-pink-700 bg-white/50"
                     :id="`inst-${o.instruction_ID}`"
                     :value="String(o.instruction_ID)"
                     v-model="selectedinstruction">
              {{ o.instruction_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- นำเสนอ -->
      <div class="bg-[#F992AF]/50 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            6.นิสิตชอบให้มีการ
            <span style="color:red;">นำเสนอหน้าชั้นเรียน</span> มากน้อยเพียงใด
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in present" :key="o.present_ID">
              <input type="radio" name="present"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.present_ID"
                     v-model="selectedpresent">
              {{ o.present_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- ประสบการณ์ -->
      <div class="bg-[#FFAE00]/35 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            7.นิสิตต้องการ
            <span style="color:red;">ประสบการณ์ใหม่ๆ</span> จากวิชานี้หรือไม่
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in experience" :key="o.experience_ID">
              <input type="radio" name="experience"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.experience_ID"
                     v-model="selectedexperience">
              {{ o.experience_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- ความยากง่าย -->
      <div class="bg-[#FBCAA8]/95 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            8.ระดับ
            <span style="color:red;">ความยากง่าย</span> ที่นิสิตต้องการ
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in challenge" :key="o.challenge_ID">
              <input type="radio" name="challenge"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.challenge_ID"
                     v-model="selectedchallenge">
              {{ o.challenge_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- ช่วงเวลา -->
      <div class="bg-[#F992AF]/50 p-6 rounded-3xl">
        <fieldset class="pl-5">
          <legend>
            9.
            <span style="color:red;">ช่วงเวลา</span> ในการเรียนที่นิสิตต้องการ(ช่วงเช้า = 8.00-11.50 , ช่วงบ่าย = 13.00-16.50)
            <span style="color:red;">*</span>
          </legend>
          <div class="pl-5">
            <label class="block" v-for="o in time" :key="o.time_ID">
              <input type="radio" name="time"
                     class="radio radio-sm radio-error bg-white/50"
                     :value="o.time_ID"
                     v-model="selectedtime">
              {{ o.time_Name }}
            </label>
          </div>
        </fieldset>
      </div>

      <!-- ปุ่ม submit -->
      <div class="text-center">
        <button type="submit" class="btn bg-[#FFB74D] hover:bg-[#F57C00] text-white">คำนวณ</button>
      </div>
    </form>
  </Layout>
</template>
