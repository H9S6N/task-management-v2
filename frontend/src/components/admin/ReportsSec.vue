<script setup>
/* التقارير — v1.0.1
   تقرير أداء نظيف: كم مهمة كمل كل موظف/مجموعة موظفين بدون حركات النقاط (سحب وإضافة)
   + تاب منفصل لسجل النقاط + تاب سجل النظام (اللوك) بصلاحيات المنفذين */
import { ref, computed, watch } from 'vue'
import { useStore } from '../../store.js'
import { api } from '../../api.js'
import { pointsFmt, fmtDateTime, EMP_STATUSES, exportCSV, isOnLeaveNow, ratingPeriodFrom, PERIODS } from '../../utils.js'

const store = useStore()

/* ==================== تاب 1: ملخص الأداء ==================== */
const tab = ref('summary')            // summary | ledger | audit
const scope = ref('all')              // all | selected
const selected = ref(new Set())
const period = ref('rating')          // today | week | rating | all
const periodLabelMap = { today: 'اليوم', week: 'آخر 7 أيام', rating: 'فترة التقييم الحالية', all: 'كل الفترات' }

function toggleSel(id) {
  selected.value.has(id) ? selected.value.delete(id) : selected.value.add(id)
  selected.value = new Set(selected.value)
}

const fromRef = computed(() => {
  if (period.value === 'today') { const d = new Date(); d.setHours(0, 0, 0, 0); return d }
  if (period.value === 'week') { const d = new Date(); d.setHours(0, 0, 0, 0); d.setDate(d.getDate() - 6); return d }
  if (period.value === 'rating') return ratingPeriodFrom(store.settings.statsStartDay, store.settings.statsStartDate)
  return null
})

const scopedEmployees = computed(() => {
  const src = store.listEmployees(true)
  if (scope.value === 'all') return src
  return src.filter(e => selected.value.has(e.id))
})

const rows = computed(() => {
  const live = store.tasks.filter(t => !t.archived && !t.deleted)
  return scopedEmployees.value.map(e => {
    const mine = live.filter(t => t.assignedToId === e.id &&
      (!fromRef.value || (t.completedAt && new Date(t.completedAt) >= fromRef.value) ||
        (!t.completedAt && t.createdAt && new Date(t.createdAt) >= fromRef.value)))
    const earned = mine.filter(t => t.status === 'completed' && t.pointsAwarded)
      .reduce((s, t) => s + (Number(t.points) || 0), 0)
    return {
      e,
      completed: mine.filter(t => t.status === 'completed').length,
      working: mine.filter(t => ['accepted', 'in_progress'].includes(t.status)).length,
      awaiting: mine.filter(t => t.status === 'pending_approval').length,
      rejected: mine.filter(t => t.status === 'rejected').length,
      declined: mine.filter(t => (t.declinedBy || []).some(d => d.employeeId === e.id) &&
        (!t.createdAt || !fromRef.value || new Date(t.createdAt) >= fromRef.value)).length,
      earned,
      balance: Number(e.totalPoints) || 0,
    }
  })
})
const totals = computed(() => rows.value.reduce((a, r) => ({
  completed: a.completed + r.completed, working: a.working + r.working,
  awaiting: a.awaiting + r.awaiting, rejected: a.rejected + r.rejected,
  declined: a.declined + r.declined, earned: a.earned + r.earned,
}), { completed: 0, working: 0, awaiting: 0, rejected: 0, declined: 0, earned: 0 }))

function expSummary() {
  exportCSV('performance-report.csv',
    ['الموظف', 'الحالة', 'مكتملة', 'قيد التنفيذ', 'بانتظار الاعتماد', 'مرفوضة من الإدارة', 'رفضها الموظف', 'نقاط الفترة', 'الرصيد الحالي'],
    rows.value.map(r => [r.e.fullName, EMP_STATUSES[r.e.status], r.completed, r.working, r.awaiting, r.rejected, r.declined, r.earned, r.balance]))
}

/* ==================== تاب 2: سجل النقاط ==================== */
const ledEmp = ref('')
const ledPeriod = ref('month')
const ledDate = ref('')
const ledLimit = ref(50)
const ledData = ref({ total: 0, summary: { count: 0, sumPositive: 0, sumNegative: 0 }, items: [] })
const ledLoading = ref(false)

const iso = d => `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
function resetDate(r) {
  const d = new Date()
  if (r === 'day') return iso(d)
  if (r === 'month') return iso(d).slice(0, 7)
  if (r === 'year') return d.getFullYear()
  return ''
}
ledDate.value = resetDate('month')

const REC_PERIODS = [['day', 'اليوم'], ['month', 'الشهر'], ['year', 'السنة'], ['all', 'الكل']]

async function loadLedger() {
  ledLoading.value = true
  try {
    const params = new URLSearchParams()
    if (ledEmp.value) params.set('employeeId', ledEmp.value)
    params.set('period', ledPeriod.value)
    if (ledPeriod.value !== 'all') params.set('date', String(ledDate.value || ''))
    params.set('limit', String(ledLimit.value))
    ledData.value = await api('/api/ledger?' + params.toString())
  } catch (e) { store.toast(e.message, 'err') }
  ledLoading.value = false
}
watch([ledEmp, ledPeriod, ledDate, ledLimit], () => { if (ledPeriod.value !== 'all' && !ledDate.value) ledDate.value = resetDate(ledPeriod.value); loadLedger() }, { immediate: true })

const ledPeriodDate = computed(() => {
  if (ledPeriod.value === 'day') return { type: 'date' }
  if (ledPeriod.value === 'month') return { type: 'month' }
  if (ledPeriod.value === 'year') return { type: 'number' }
  return null
})

/* ==================== تاب 3: سجل النظام (اللوك) ==================== */
const audPeriod = ref('month')
const audDate = ref(resetDate('month'))
const audLimit = ref(50)
const audData = ref({ total: 0, items: [] })
const audLoading = ref(false)

const AUDIT_LABELS = {
  login: 'دخول', create_task: 'إنشاء مهمة', update_task: 'تعديل مهمة',
  accept_task: 'قبول مهمة', decline_task: 'رفض مهمة من موظف', start_task: 'بدء مهمة',
  submit_task: 'رفع تقرير', approve_task: 'اعتماد', reject_task: 'رفض اعتماد',
  cancel_task: 'إلغاء مهمة', unassign_task: 'إلغاء قبول',
  archive_task: 'أرشفة مهمة', unarchive_task: 'استعادة من الأرشيف',
  delete_task: 'حذف مهمة (ناعم)', restore_task: 'استعادة مهمة',
  manual_points: 'تعديل نقاط', settings: 'إعدادات', reset_points: 'تصفير نقاط',
  reset_counters: 'تصفير عدادات', change_password: 'تغيير كلمة المرور',
  change_credentials: 'تغيير بيانات ذاتية', auto_leave_end: 'انتهاء إجازة تلقائي',
  toggle_leave: 'إجازة/عودة', add_employee: 'إضافة موظف', update_employee: 'تعديل موظف',
  delete_employee: 'حذف موظف (ناعم)', restore_employee: 'استعادة موظف',
  edit_completed: 'تعديل عدد الإنجازات', reset_pin: 'إعادة تعيين رمز',
  add_manager: 'إضافة مدير مهام', update_manager: 'تعديل مدير مهام',
  scheduled_send: 'إرسال مهمة مجدولة', auto_backup: 'نسخة احتياطية تلقائية',
  backup_telegram: 'إرسال نسخة لتيليجرام', backup_upload: 'استعادة نسخة',
  restore_backup: 'تطبيق نسخة مستعادة', push_init: 'تهيئة إشعارات المتصفح',
  add_task_type: 'إضافة نوع', update_task_type: 'تعديل نوع', delete_task_type: 'حذف نوع',
}
const ROLE_BADGE = { admin: 'st-admin', manager: 'st-manager', employee: 'st-auto', system: 'st-cancelled' }
const ROLE_LABEL = { admin: 'المدير العام', manager: 'مدير مهام', employee: 'موظف', system: 'النظام' }

async function loadAudit() {
  audLoading.value = true
  try {
    const params = new URLSearchParams()
    params.set('period', audPeriod.value)
    if (audPeriod.value !== 'all') params.set('date', String(audDate.value || ''))
    params.set('limit', String(audLimit.value))
    audData.value = await api('/api/audit?' + params.toString())
  } catch (e) { store.toast(e.message, 'err') }
  audLoading.value = false
}
watch([audPeriod, audDate, audLimit], () => { if (audPeriod.value !== 'all' && !audDate.value) audDate.value = resetDate(audPeriod.value); loadAudit() }, { immediate: true })

const audPeriodDate = computed(() => {
  if (audPeriod.value === 'day') return { type: 'date' }
  if (audPeriod.value === 'month') return { type: 'month' }
  if (audPeriod.value === 'year') return { type: 'number' }
  return null
})

function expLedger() {
  exportCSV('points_ledger.csv', ['التاريخ', 'الموظف', 'التغيير', 'الرصيد', 'السبب', 'بواسطة', 'الصلاحية'],
    ledData.value.items.map(l => [fmtDateTime(l.createdAt), l.employeeName, l.changeAmount, l.balanceAfter, l.reason, l.performedBy || 'النظام',
      ROLE_LABEL[l.performedByRole] || 'النظام']))
}
function expAudit() {
  exportCSV('system_log.csv', ['التاريخ', 'الحدث', 'التفاصيل', 'بواسطة', 'الصلاحية'],
    audData.value.items.map(a => [fmtDateTime(a.createdAt), AUDIT_LABELS[a.action] || a.action, a.details, a.actor,
      ROLE_LABEL[a.actorRole] || a.actorRole]))
}
</script>

<template>
  <section>
    <h2 class="sec-title">⭐ التقارير</h2>

    <!-- ==================== ملخص الأداء ==================== -->
    <div class="filter-bar">
      <button class="chip" :class="{ active: tab === 'summary' }" @click="tab = 'summary'">📊 ملخص الأداء</button>
      <button class="chip" :class="{ active: tab === 'ledger' }" @click="tab = 'ledger'">📜 سجل النقاط</button>
      <button class="chip" :class="{ active: tab === 'audit' }" @click="tab = 'audit'">🛡️ سجل النظام (اللوك)</button>
    </div>

    <div v-if="tab === 'summary'" class="card">
      <div class="filter-bar">
        <button class="chip" :class="{ active: scope === 'all' }" @click="scope = 'all'">👥 كل الموظفين</button>
        <button class="chip" :class="{ active: scope === 'selected' }" @click="scope = 'selected'">✅ اختيار موظفين</button>
      </div>

      <div v-if="scope === 'selected'" class="checks" style="margin-bottom:12px">
        <label v-for="e in store.listEmployees(true)" :key="e.id" class="check-pill"
               :class="{ 'pill-active': selected.has(e.id) }">
          <input type="checkbox" :checked="selected.has(e.id)" @change="toggleSel(e.id)"> {{ e.fullName }}
        </label>
      </div>

      <div class="filter-bar">
        <button v-for="[k, lbl] in [['today', 'اليوم'], ['week', 'آخر 7 أيام'], ['rating', 'فترة التقييم الحالية'], ['all', 'كل الفترات']]"
                :key="k" class="chip" :class="{ active: period === k }" @click="period = k">{{ lbl }}</button>
      </div>
      <p v-if="period === 'rating' && fromRef" class="muted" style="margin-bottom:10px">
        فترة التقييم تبدأ من <b>{{ fromRef.toLocaleDateString('ar-IQ-u-nu-latn') }}</b>
        (يوم بداية التقييم بالإعدادات: {{ store.settings.statsStartDay || 1 }} من كل شهر)
      </p>

      <div class="table-wrap">
        <table class="table">
          <thead>
            <tr><th>الموظف</th><th>الحالة</th><th>✅ مكتملة</th><th>🔧 قيد التنفيذ</th>
              <th>⏳ بانتظار اعتماد</th><th>❌ مرفوضة من الإدارة</th><th>🚫 رفضها الموظف</th>
              <th>⭐ نقاط الفترة</th><th>💰 الرصيد</th></tr>
          </thead>
          <tbody>
            <tr v-for="r in rows" :key="r.e.id" :class="{ 'emp-disabled': r.e.status === 'disabled' }">
              <td><b>{{ r.e.fullName }}</b></td>
              <td><span class="st" :class="'st-' + r.e.status">{{ EMP_STATUSES[r.e.status] }}</span></td>
              <td><b>{{ r.completed }}</b></td>
              <td>{{ r.working }}</td>
              <td>{{ r.awaiting }}</td>
              <td>{{ r.rejected }}</td>
              <td>{{ r.declined }}</td>
              <td class="plus">+{{ pointsFmt(r.earned) }}</td>
              <td>{{ pointsFmt(r.balance) }}</td>
            </tr>
            <tr v-if="rows.length && rows.length > 1" style="background:var(--primary-light)">
              <td><b>المجموع ({{ rows.length }} موظف)</b></td>
              <td></td>
              <td><b>{{ totals.completed }}</b></td>
              <td><b>{{ totals.working }}</b></td>
              <td><b>{{ totals.awaiting }}</b></td>
              <td><b>{{ totals.rejected }}</b></td>
              <td><b>{{ totals.declined }}</b></td>
              <td><b class="plus">+{{ pointsFmt(totals.earned) }}</b></td>
              <td></td>
            </tr>
            <tr v-if="!rows.length"><td colspan="9" class="empty">اختر موظفين لعرض التقرير</td></tr>
          </tbody>
        </table>
      </div>
      <div class="btn-row">
        <button class="btn btn-outline" @click="expSummary">📥 تصدير التقرير CSV</button>
      </div>
      <p class="hint">هذا التقرير يعرض الأداء فقط بدون حركات النقاط (السحب والإضافة) — حركات النقاط بتاب "سجل النقاط"</p>
    </div>

    <!-- ==================== سجل النقاط ==================== -->
    <div v-if="tab === 'ledger'" class="card">
      <div class="sec-head">
        <h3>📜 سجل النقاط</h3>
        <span class="muted">{{ ledLoading ? '⏳ جاري التحميل...' : `${ledData.total} حركة` }}</span>
      </div>

      <div class="filter-bar">
        <select v-model="ledEmp" class="input-inline">
          <option value="">كل الموظفين</option>
          <option v-for="e in store.employees" :key="e.id" :value="e.id">{{ e.fullName }}</option>
        </select>
        <button v-for="[k, lbl] in REC_PERIODS" :key="k" class="chip" :class="{ active: ledPeriod === k }"
                @click="ledPeriod = k; ledDate.value !== undefined && (ledDate = resetDate(k))">{{ lbl }}</button>
        <input v-if="ledPeriodDate" v-model="ledDate" :type="ledPeriodDate.type" class="input-inline" style="max-width:150px">
        <select v-model.number="ledLimit" class="input-inline" style="max-width:110px">
          <option :value="10">10 مخرجات</option><option :value="50">50 مخرج</option><option :value="100">100 مخرج</option>
        </select>
      </div>

      <p v-if="ledPeriod !== 'all'" class="muted" style="margin:6px 0">
        ملخص الفترة: <b>{{ ledData.summary.count }}</b> حركة —
        زيادات: <b class="plus">+{{ pointsFmt(ledData.summary.sumPositive) }}</b> —
        خصومات: <b class="minus">{{ pointsFmt(ledData.summary.sumNegative) }}</b>
      </p>

      <div class="table-wrap">
        <table class="table">
          <thead><tr><th>التاريخ</th><th>الموظف</th><th>التغيير</th><th>الرصيد بعدها</th><th>السبب</th><th>بواسطة</th></tr></thead>
          <tbody>
            <tr v-for="(l, i) in ledData.items" :key="l.id || i">
              <td>{{ fmtDateTime(l.createdAt) }}</td>
              <td>{{ l.employeeName }}</td>
              <td :class="l.changeAmount >= 0 ? 'plus' : 'minus'">{{ l.changeAmount >= 0 ? '+' : '' }}{{ pointsFmt(l.changeAmount) }}</td>
              <td>{{ pointsFmt(l.balanceAfter) }}</td>
              <td>{{ l.reason }}</td>
              <td>
                {{ l.performedBy || 'النظام' }}
                <span v-if="l.performedByRole" class="st" :class="ROLE_BADGE[l.performedByRole]">{{ ROLE_LABEL[l.performedByRole] }}</span>
              </td>
            </tr>
            <tr v-if="!ledData.items.length"><td colspan="6" class="empty">لا توجد حركات بهذه الفترة</td></tr>
          </tbody>
        </table>
      </div>
      <div class="btn-row"><button class="btn btn-outline" @click="expLedger">📥 تصدير CSV</button></div>
    </div>

    <!-- ==================== سجل النظام ==================== -->
    <div v-if="tab === 'audit'" class="card">
      <div class="sec-head">
        <h3>🛡️ سجل النظام — من سوى شو وبأي صلاحية</h3>
        <span class="muted">{{ audLoading ? '⏳ جاري التحميل...' : `${audData.total} حدث` }}</span>
      </div>

      <div class="filter-bar">
        <button v-for="[k, lbl] in REC_PERIODS" :key="k" class="chip" :class="{ active: audPeriod === k }"
                @click="audPeriod = k; audDate = resetDate(k)">{{ lbl }}</button>
        <input v-if="audPeriodDate" v-model="audDate" :type="audPeriodDate.type" class="input-inline" style="max-width:150px">
        <select v-model.number="audLimit" class="input-inline" style="max-width:110px">
          <option :value="10">10 مخرجات</option><option :value="50">50 مخرج</option><option :value="100">100 مخرج</option>
        </select>
      </div>

      <div class="table-wrap">
        <table class="table">
          <thead><tr><th>التاريخ</th><th>الصلاحية</th><th>بواسطة</th><th>الحدث</th><th>التفاصيل</th></tr></thead>
          <tbody>
            <tr v-for="(a, i) in audData.items" :key="a.id || i">
              <td>{{ fmtDateTime(a.createdAt) }}</td>
              <td><span class="st" :class="ROLE_BADGE[a.actorRole] || 'st-auto'">{{ ROLE_LABEL[a.actorRole] || a.actorRole }}</span></td>
              <td><b>{{ a.actor }}</b></td>
              <td>{{ AUDIT_LABELS[a.action] || a.action }}</td>
              <td style="max-width:340px">{{ a.details }}</td>
            </tr>
            <tr v-if="!audData.items.length"><td colspan="5" class="empty">لا توجد أحداث بهذه الفترة</td></tr>
          </tbody>
        </table>
      </div>
      <div class="btn-row"><button class="btn btn-outline" @click="expAudit">📥 تصدير CSV</button></div>
    </div>
  </section>
</template>
