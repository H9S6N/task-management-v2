<script setup>
/* لوحة التحكم — v1.0.1
   عدادات تحترم فترة التقييم (يوم بداية الشهر + تاريخ بداية الاحتساب)
   + نظرة عامة على حالة الموظفين: شغال / عنده مهام / مجاز / موقوف */
import { computed } from 'vue'
import { useStore } from '../../store.js'
import { pointsFmt, statusLabel, statusClass, ratingPeriodFrom, isOnLeaveNow, EMP_STATUSES } from '../../utils.js'

const store = useStore()

const periodFrom = computed(() => ratingPeriodFrom(store.settings.statsStartDay, store.settings.statsStartDate))
const periodLabel = computed(() => {
  const d = periodFrom.value
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
})

const live = computed(() => store.tasks.filter(t => !t.archived && !t.deleted))
const inPeriod = computed(() => live.value.filter(t =>
  t.completedAt && new Date(t.completedAt) >= periodFrom.value))
const todayStart = computed(() => { const d = new Date(); d.setHours(0, 0, 0, 0); return d })
const completedToday = computed(() => live.value.filter(t => t.completedAt && new Date(t.completedAt) >= todayStart.value))
const pointsAwardedPeriod = computed(() =>
  inPeriod.value.filter(t => t.pointsAwarded).reduce((s, t) => s + (Number(t.points) || 0), 0))

/* حالة الموظفين الآن */
const empRows = computed(() => {
  return store.employees.filter(e => !e.deleted).map(e => {
    const openTasks = live.value.filter(t =>
      t.assignedToId === e.id && ['accepted', 'in_progress', 'pending_approval'].includes(t.status))
    const doneToday = live.value.filter(t =>
      t.assignedToId === e.id && t.status === 'completed' && t.completedAt && new Date(t.completedAt) >= todayStart.value).length
    return {
      e, open: openTasks.length, doneToday,
      state: e.status === 'disabled' ? 'disabled'
        : e.status === 'on_leave' && isOnLeaveNow(e) ? 'leave'
        : e.status === 'on_leave' ? 'active'
        : openTasks.length ? 'working' : 'idle',
    }
  }).sort((a, b) => {
    const order = { working: 0, idle: 1, leave: 2, disabled: 3 }
    return order[a.state] - order[b.state] || b.e.totalPoints - a.e.totalPoints
  })
})
const stateMeta = {
  working: { txt: 'يعمل على مهام', cls: 'st-accepted' },
  idle: { txt: 'متفرغ — ما عنده مهام', cls: 'st-available' },
  leave: { txt: 'في إجازة 🏖️', cls: 'st-on_leave' },
  disabled: { txt: 'موقوف', cls: 'st-disabled' },
}

const stats = computed(() => [
  ['👥', store.activeEmployees.filter(e => e.status === 'active' && !isOnLeaveNow(e)).length, 'موظف متاح الآن'],
  ['🪙', live.value.filter(t => t.status === 'available').length, 'مهمة متاحة'],
  ['⏰', store.scheduledTasks.length, 'مهام مجدولة'],
  ['🔧', live.value.filter(t => ['accepted', 'in_progress'].includes(t.status)).length, 'قيد التنفيذ'],
  ['⏳', store.pendingTasks.length, 'بانتظار الاعتماد'],
  ['✅', completedToday.value.length, 'اكتملت اليوم'],
  ['🏆', inPeriod.value.length, `اكتملت بفترة التقييم (من ${periodLabel.value})`],
  ['⭐', pointsFmt(pointsAwardedPeriod.value), 'نقاط موزعة بالفترة'],
])
</script>

<template>
  <section>
    <h2 class="sec-title">{{ store.isManager ? '📊 لوحة المتابعة' : '📊 لوحة التحكم' }}</h2>

    <div class="stats-grid">
      <div v-for="(s, i) in stats" :key="i" class="stat">
        <div class="num">{{ s[0] }} {{ s[1] }}</div>
        <div class="lbl">{{ s[2] }}</div>
      </div>
    </div>

    <div class="dash-cols">
      <div class="card">
        <h3>👥 حالة الموظفين الآن</h3>
        <div class="mini-list">
          <div v-for="row in empRows" :key="row.e.id" class="mini-item emp-row"
               :class="{ 'emp-disabled': row.e.status === 'disabled', 'emp-on-leave': row.state === 'leave' }">
            <span class="t">👷 {{ row.e.fullName }}</span>
            <span class="st" :class="stateMeta[row.state].cls">{{ stateMeta[row.state].txt }}</span>
            <span class="muted" style="white-space:nowrap">
              🔧 {{ row.open }} — ✅ اليوم: {{ row.doneToday }}
            </span>
          </div>
          <div v-if="!empRows.length" class="empty">لا يوجد موظفون بعد</div>
        </div>
      </div>

      <div class="card">
        <h3>⏳ بانتظار الاعتماد</h3>
        <div class="mini-list">
          <template v-if="store.pendingTasks.length">
            <div v-for="t in store.pendingTasks.slice(0, 6)" :key="t.id" class="mini-item" @click="store.setSection('approvals')">
              <span class="t">#{{ t.no }} {{ t.title }}</span>
              <span :class="statusClass(t.status)">{{ statusLabel(t.status) }}</span>
            </div>
            <p v-if="store.pendingTasks.length > 6" class="muted">+{{ store.pendingTasks.length - 6 }} أخرى بقسم الاعتماد</p>
          </template>
          <div v-else class="empty"><span class="big">🎉</span>لا يوجد شيء بانتظار الاعتماد</div>

          <template v-if="store.scheduledTasks.length">
            <h3 style="margin-top:14px">⏰ مهام مجدولة</h3>
            <div v-for="t in store.scheduledTasks.slice(0, 4)" :key="t.id" class="mini-item" @click="store.setSection('tasks')">
              <span class="t">#{{ t.no }} {{ t.title }}</span>
              <span class="st st-scheduled">ترسل لاحقاً</span>
            </div>
          </template>
        </div>
      </div>
    </div>
  </section>
</template>
