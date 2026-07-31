<template>
  <div class="container py-4">
    <div class="bg-white p-4 rounded shadow-sm border">
      <h2 class="text-center text-primary fw-bold mb-4">
        📊 Отчёт по сотрудникам
      </h2>

      <!-- Фильтры и таймер -->
      <div class="row g-3 align-items-end mb-1">
        <!-- Период -->
        <div class="col-md-3">
          <label class="form-label fw-semibold">Период:</label>
          <input
            ref="datePicker"
            type="text"
            class="form-control"
            placeholder="ДД.ММ.ГГ – ДД.ММ.ГГ"
          />
        </div>

        <!-- Отдел -->
        <div class="col-md-3">
          <label class="form-label fw-semibold">Отдел:</label>
          <select v-model="filters.department" class="form-select">
            <option value="">Все</option>
            <option
              v-for="d in departments"
              :key="d.ID"
              :value="String(d.ID)"
            >{{ d.NAME }}</option>
          </select>
        </div>

        <!-- Менеджер -->
        <div class="col-md-3">
          <label class="form-label fw-semibold">Менеджер:</label>
          <div class="dropdown">
            <button
              class="btn btn-outline-secondary dropdown-toggle w-100 text-start"
              type="button"
              id="managerDropdown"
              data-bs-toggle="dropdown"
              aria-expanded="false"
            >
              Выбрать менеджеров
            </button>
            <div class="dropdown-menu p-3" aria-labelledby="managerDropdown" style="max-height:200px; overflow-y:auto;">
              <div class="fw-semibold mb-2">Активные</div>
              <div v-for="m in availableManagers.filter(m => m.active)" :key="`act-${m.id}`" class="form-check">
                <input
                  class="form-check-input"
                  type="checkbox"
                  :id="`mgr-${m.id}`"
                  :value="m.name"
                  v-model="filters.manager"
                />
                <label class="form-check-label" :for="`mgr-${m.id}`">
                  {{ m.name }}
                </label>
              </div>
              <hr class="dropdown-divider">
              <div class="fw-semibold text-muted mb-2">Уволенные</div>
              <div v-for="m in availableManagers.filter(m => !m.active)" :key="`off-${m.id}`" class="form-check">
                <input
                  class="form-check-input"
                  type="checkbox"
                  :id="`mgr-off-${m.id}`"
                  :value="m.name"
                  v-model="filters.manager"
                />
                <label class="form-check-label text-muted" :for="`mgr-off-${m.id}`">
                  {{ m.name }} (Уволен)
                </label>
              </div>
            </div>
          </div>
        </div>

        <!-- Воронка -->
        <div class="col-md-3">
          <label class="form-label fw-semibold">Воронка:</label>
          <select v-model="filters.pipeline" class="form-select">
            <option value="">Все</option>
            <option value="0">Общая</option>
            <option value="1">Москва</option>
            <option value="2">Промка</option>
          </select>
        </div>
      </div>

      <!-- Таймер -->
      <div class="text-end mb-4 small text-muted">
        Следующее обновление через: {{ timerCount }} сек
      </div>

      <!-- Вкладки -->
      <ul class="nav nav-tabs mb-4">
        <li class="nav-item">
          <button
            class="nav-link"
            :class="{ active: tab === 'deals' }"
            @click="tab = 'deals'"
          >Сделки</button>
        </li>
        <li class="nav-item">
          <button
            class="nav-link"
            :class="{ active: tab === 'calls' }"
            @click="tab = 'calls'"
          >Звонки</button>
        </li>
        <li class="nav-item">
          <button
            class="nav-link"
            :class="{ active: tab === 'tasks' }"
            @click="tab = 'tasks'"
          >Задачи</button>
        </li>
        <li class="nav-item">
          <button
            class="nav-link"
            :class="{ active: tab === 'stats' }"
            @click="tab = 'stats'"
          >Общая статистика</button>
        </li>
      </ul>

      <!-- Сделки -->
      <div v-if="tab === 'deals'" class="table-responsive">
        <table class="table table-bordered align-middle text-center">
          <thead class="table-primary">
            <tr>
              <th>Менеджер</th>
              <th>Новые лиды</th>
              <th>Недозвон</th>
              <th>Отправлено КП</th>
              <th>Договор</th>
              <th>Размещение</th>
              <th>Обучение</th>
              <th>1-я продажа</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="m in filteredManagers" :key="m.id">
  <td>{{ m.name }}</td>
  <td>{{ m.stages['Новая'] || 0 }}</td>
  <td>{{ m.stages['Недозвон'] || 0 }}</td>
  <td>{{ m.stages['Отправлено КП'] || 0 }}</td>
  <td>{{ m.stages['Договор'] || 0 }}</td>
  <td>{{ m.stages['Размещение на сайте'] || 0 }}</td>
  <td>{{ m.stages['Назначено тех.обучение'] || 0 }}</td>
  <td>{{ m.stages['1я продажа'] || 0 }}</td>
</tr>
          </tbody>
        </table>
      </div>

      <!-- Звонки -->
      <div v-else-if="tab === 'calls'" class="table-responsive">
        <table class="table table-bordered align-middle text-center">
          <thead class="table-info">
            <tr>
              <th>Менеджер</th>
              <th>Всего звонков</th>
              <th>Успешные</th>
              <th>Длительность (мин)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="m in filteredManagers" :key="m.id">
              <td>{{ m.name }}</td>
              <td>{{ m.calls }}</td>
              <td>{{ m.success }}</td>
              <td>{{ m.durationMinutes }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Задачи -->
      <transition-group
        name="fade-slide"
        tag="div"
        class="row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4"
        v-else-if="tab === 'tasks'"
      >
        <div v-for="m in filteredManagers" :key="m.id" class="col">
          <div class="card border shadow-sm h-100">
            <div
              :class="[
                'card-header text-white',
                m.total >= maxTotal ? 'bg-success' : 'bg-primary'
              ]"
            >
              {{ m.name }}
            </div>
            <div class="card-body bg-light">
              <ul class="list-group list-group-flush small">
                <li class="list-group-item d-flex justify-content-between">
                  <span>Всего:</span><span>{{ m.total }}</span>
                </li>
                <li class="list-group-item d-flex justify-content-between">
                  <span>На сегодня:</span><span>{{ m.today }}</span>
                </li>
                <li class="list-group-item d-flex justify-content-between">
                  <span>Без задач:</span><span>{{ m.noTasks }}</span>
                </li>
                <li class="list-group-item d-flex justify-content-between">
                  <span>&gt; 2 недель:</span><span>{{ m.old }}</span>
                </li>
                <li class="list-group-item d-flex justify-content-between">
                  <span>Просрочено:</span>
                  <span class="text-danger">{{ m.overdue }}</span>
                </li>
              </ul>
            </div>
          </div>
        </div>
      </transition-group>

      <!-- Общая статистика -->
      <div v-else-if="tab === 'stats'" class="stats-wrapper p-4 bg-light rounded border">
        <h4 class="text-primary text-center mb-4">Общая статистика</h4>
        <div class="row g-4">
          <div v-for="chart in statCharts" :key="chart.title" class="col-md-6">
            <h6 class="text-center">{{ chart.title }}</h6>
            <component
              :is="chart.type === 'line' ? 'LineChart' : 'BarChart'"
              :chart-data="chart.data"
              class="w-100"
              style="height:250px;"
            />
          </div>
        </div>
        <div class="table-responsive mt-4">
          <table class="table table-bordered text-center">
            <thead class="table-secondary">
              <tr>
                <th>Менеджер</th>
                <th>Лиды</th>
                <th>Недозвон</th>
                <th>КП</th>
                <th>Звонки</th>
                <th>Задачи</th>
                <th>Конверсия %</th>
                <th>Нагрузка</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="m in filteredManagers" :key="m.id">
                <td>{{ m.name }}</td>
                <td>{{ m.leads }}</td>
                <td>{{ m.missed }}</td>
                <td>{{ m.kp }}</td>
                <td>{{ m.calls }}</td>
                <td>{{ m.total }}</td>
                <td>{{ m.leads ? ((m.contracts/m.leads)*100).toFixed(1) : 0 }}</td>
                <td>{{ m.calls + m.total }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Кнопка обновления -->
      <div class="text-center mt-4">
        <button
          @click="refreshData"
          class="btn btn-primary btn-animated px-4 py-2 fw-semibold shadow-sm"
        >
          Сформировать отчёт
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
/* ---------- импорт библиотек, как и раньше ---------- */
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import flatpickr from 'flatpickr'
import { Russian } from 'flatpickr/dist/l10n/ru.js'
import 'flatpickr/dist/flatpickr.css'
import moment from 'moment'
import axios from 'axios'               // используем для вызова веб‑хуков POST

/* ---------- константы, взятые из indexEpishura.html ---------- */
const WEBHOOK_URL = ''
const allowedPlanEditors = [31, 37]     // кто может редактировать план

/* ---------- реактивное состояние ---------- */
const tab          = ref('deals')
const timerCount   = ref(420)
const datePickerEl = ref(null)

const filters = ref({
  dateRange : [new Date(), new Date()], // flatpickr заполнит при onReady
  department: '',
  manager   : [],
  pipeline  : ''
})

/* справочники */
const departments = ref([])
const managers    = ref([])   // все пользователи (active)
const stageMap    = ref({})   // мэппинг стадий двух воронок

/* фактические строки отчёта */
const tableData = ref([])

/* ---------- вспомогательные утилиты  (1‑в‑1 из файла) ---------- */
function obj2params (obj){
  const p=new URLSearchParams()
  for(const k in obj){
    const v=obj[k]
    if(Array.isArray(v))          v.forEach(x=>p.append(k+'[]',x))
    else if(v&&typeof v==='object')
      Object.entries(v).forEach(([sk,sv])=>p.append(`${k}[${sk}]`,sv))
    else p.append(k,v)
  }
  return p
}
async function callWebhook(method, params = {}){
  const { data } = await axios.post(
    WEBHOOK_URL + method + '.json',
    obj2params(params)
  )
  return data
}

/* ---------- загрузка справочников ---------- */
async function loadDepartments(){
  const res = await callWebhook('crm.department.list') // proxy в indexEpishura нет, берём прямой метод
  departments.value = res.result || []
}
async function loadUsers(){
  const res = await callWebhook('crm.user.list', { filter:{ ACTIVE:'Y' } })
  managers.value = (res.result||[]).map(u=>({
    id         : u.ID,
    name       : `${u.NAME} ${u.LAST_NAME}`.trim(),
    department : Array.isArray(u.UF_DEPARTMENT)
      ? u.UF_DEPARTMENT[0] : u.UF_DEPARTMENT,
    active     : true
  }))
}

/* ---------- мэппинг стадий двух воронок ---------- */
async function getStageMapping(){
  const [d0,d1] = await Promise.all([
    callWebhook('crm.status.list', { filter:{ ENTITY_ID:'DEAL_STAGE' } }),
    callWebhook('crm.status.list', { filter:{ ENTITY_ID:'DEAL_STAGE_1' } })
  ])
  const mapping={}
  function add(list){
    (list.result||[]).forEach(s=>{
      const key=s.NAME.trim().toLowerCase().normalize('NFKC')
      ;(mapping[key] ||= []).push(s.STATUS_ID)
    })
  }
  add(d0); add(d1)
  stageMap.value = mapping
}

/* ---------- helpers для сделок / звонков (сокращённые) ---------- */
function normalizeStage(name){ return name.trim().toLowerCase().normalize('NFKC') }
function stageCodes(name){ return stageMap.value[normalizeStage(name)] || [] }

async function allDealIds(uid){
  let ids=[], start=0
  do{
    const res=await callWebhook('crm.deal.list',{
      filter:{ ASSIGNED_BY_ID:uid },
      select:['ID'], start, NAV_PARAMS:{ nPageSize:50 }
    })
    ids.push(...(res.result||[]).map(d=>d.ID))
    start=res.next??false
  }while(start)
  return ids
}
async function countStageDeals(uid,stageName,dates,codes,ids){
  if(!codes.length||!ids.length) return 0
  const from = moment(dates[0]).startOf('day').utcOffset(180,true).format()
  const to   = moment(dates[1]).endOf('day')  .utcOffset(180,true).format()
  const chunks=[]
  for(let i=0;i<ids.length;i+=50) chunks.push(ids.slice(i,i+50))
  const cmd={}
  chunks.forEach((chunk,idx)=>{
    let q='crm.stagehistory.list?entityTypeId=2'
    codes.forEach(c=>q+=`&filter[STAGE_ID][]=${encodeURIComponent(c)}`)
    q+=`&filter[>=CREATED_TIME]=${encodeURIComponent(from)}`
    q+=`&filter[<=CREATED_TIME]=${encodeURIComponent(to)}`
    chunk.forEach(id=>q+=`&filter[OWNER_ID][]=${id}`)
    cmd['q'+idx]=q
  })
  const res=await callWebhook('batch',{ halt:0,cmd })
  const uniq=new Set()
  Object.values(res.result?.result||{}).forEach(r=>{
    (r.items||[]).forEach(i=>uniq.add(i.OWNER_ID))
  })
  return uniq.size
}
async function totalCalls(uid,dates){
  const res=await callWebhook('voximplant.statistic.get',{
    filter:{
      PORTAL_USER_ID:uid,
      '>=CALL_START_DATE':moment(dates[0]).startOf('day').utcOffset(180,true).format(),
      '<=CALL_START_DATE':moment(dates[1]).endOf('day')  .utcOffset(180,true).format()
    }
  })
  return res.total ?? (res.result||[]).length
}
async function successCalls(uid,dates){
  const res=await callWebhook('voximplant.statistic.get',{
    filter:{
      PORTAL_USER_ID:uid,
      '>=CALL_START_DATE':moment(dates[0]).startOf('day').utcOffset(180,true).format(),
      '<=CALL_START_DATE':moment(dates[1]).endOf('day')  .utcOffset(180,true).format(),
      '>=CALL_DURATION':30
    }
  })
  return res.total ?? (res.result||[]).length
}
async function callDuration(uid,dates){
  let sum=0,start=0
  do{
    const page=await callWebhook('voximplant.statistic.get',{
      filter:{
        PORTAL_USER_ID:uid,
        '>=CALL_START_DATE':moment(dates[0]).startOf('day').utcOffset(180,true).format(),
        '<=CALL_START_DATE':moment(dates[1]).endOf('day')  .utcOffset(180,true).format()
      }, start, NAV_PARAMS:{ nPageSize:500 }
    })
    (page.result||[]).forEach(c=>sum+=+c.CALL_DURATION||0)
    start=page.next??false
  }while(start)
  return sum
}
async function newLeads(uid,dates){
  const from=moment(dates[0]).startOf('day').utcOffset(180,true).format()
  const to  =moment(dates[1]).endOf('day')  .utcOffset(180,true).format()
  async function cnt(cat){
    let c=0,s=0
    do{
      const p=await callWebhook('crm.deal.list',{
        filter:{ ASSIGNED_BY_ID:uid,'>=DATE_CREATE':from,'<=DATE_CREATE':to,CATEGORY_ID:cat },
        select:['ID'], start:s, NAV_PARAMS:{ nPageSize:50 }
      })
      c += (p.result||[]).length
      s =  p.next??false
    }while(s)
    return c
  }
  return (await cnt(0)) + (await cnt(1))
}

/* ---------- формирование отчёта ---------- */
const metricStages = [
  { key:'otpravleno_kp', title:'Отправлено КП' },
  { key:'dogovor',       title:'Договор' },
  { key:'razmeshenie',   title:'Размещение на сайте' },
  { key:'tech_training', title:'Назначено тех.обучение' },
  { key:'first_sale',    title:'1я продажа' }
]
async function buildReport(){
  const dates = filters.value.dateRange.length
    ? filters.value.dateRange
    : [new Date(), new Date()]

  const mgrs = managers.value.filter(m=>{
    const depOk = !filters.value.department || String(m.department) === filters.value.department
    const mgrOk = !filters.value.manager.length || filters.value.manager.includes(m.name)
    return depOk && mgrOk
  })

  const rows = await Promise.all(mgrs.map(async m=>{
    const [calls, success, dur, leads, ids] = await Promise.all([
      totalCalls(m.id, dates),
      successCalls(m.id, dates),
      callDuration(m.id, dates),
      newLeads(m.id, dates),
      allDealIds(m.id)
    ])
    const stageCounts = {}
    for(const st of metricStages){
      stageCounts[st.key] = await countStageDeals(
        m.id, st.title, dates, stageCodes(st.title), ids
      )
    }
    return {
      id: m.id,
      name: m.name,
      /* звонки */
      calls,
      success,
      durationMinutes:(dur/60).toFixed(1),
      /* лиды */
      leads,
      missed:0,
      kp: stageCounts.otpravleno_kp,
      contracts: stageCounts.dogovor,
      /* задачи (в оригинальном скрипте нет — выводим нули) */
      total:0, today:0, noTasks:0, old:0, overdue:0,
      /* сделки по стадиям для таба «Сделки» */
      stages:{
        'Новая'              : leads,
        'Недозвон'           : 0,
        'Отправлено КП'      : stageCounts.otpravleno_kp,
        'Договор'            : stageCounts.dogovor,
        'Размещение на сайте': stageCounts.razmeshenie,
        'Назначено тех.обучение': stageCounts.tech_training,
        '1я продажа'         : stageCounts.first_sale
      }
    }
  }))
  tableData.value = rows
}

/* ---------- computed, которые использует ваш шаблон ---------- */
const availableManagers = computed(()=>managers.value)
const filteredManagers  = computed(()=>tableData.value)
const maxTotal          = computed(()=>Math.max(0,...tableData.value.map(r=>r.total)))

const statCharts = computed(()=>{
  const lbl = filteredManagers.value.map(r=>r.name)
  const conv = filteredManagers.value.map(r=>r.leads
    ? ((r.contracts/r.leads)*100).toFixed(1) : 0)
  return [{
    title:'Договоры / лиды, %',
    type :'bar',
    data :{ labels:lbl, datasets:[{ label:'Конверсия', data:conv }] }
  }]
})

/* ---------- refresh / таймер ---------- */
let ticker=null
async function refreshData(){
  await buildReport()
  timerCount.value = 420
}

/* ---------- lifecycle ---------- */
onMounted(async ()=>{
  /* flatpickr */
  flatpickr(datePickerEl.value,{
    mode:'range', dateFormat:'d.m.Y', locale:Russian,
    defaultDate:new Date(),
    onChange:dates=>filters.value.dateRange=dates,
    onReady :(dates)=>filters.value.dateRange=dates
  })
  /* справочники + мэппинг */
  await Promise.all([loadDepartments(), loadUsers(), getStageMapping()])
  await refreshData()

  /* таймер авто‑обновления */
  ticker = setInterval(()=>{
    timerCount.value ? timerCount.value-- : refreshData()
  },1000)
})
onUnmounted(()=>clearInterval(ticker))

/* ---------- экспорт в шаблон (если нужно) ---------- */
defineExpose({ statCharts })
</script>










<style scoped>
/* чтобы сам контейнер прокручивался, если карточек/менеджеров больше чем экран */
.container {
  max-height: 100vh;
  overflow-y: auto;
}
</style>

<style scoped>
/* Скролл для общей статистики */
.stats-wrapper {
  max-height: 70vh;
  overflow-y: auto;
  padding-right: 0.5rem;
}

/* Кнопки */
.btn-animated {
  color: #fff;
  background-color: #0d6efd;
  border-color: #0d6efd;
  transition: all 0.3s ease;
}
.btn-animated:hover,
.btn-animated:focus,
.btn-animated:active {
  background-color: #0056b3 !important;
  border-color: #004a99 !important;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 86, 179, 0.4);
}

/* Навигационные табы hover */
.nav-tabs .nav-link:hover {
  background-color: #0d6efd;
  color: #fff;
  transition: 0.2s;
}

/* Анимация карточек */
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.4s ease;
}
.fade-slide-enter-from,
.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(10px);
}
</style>
