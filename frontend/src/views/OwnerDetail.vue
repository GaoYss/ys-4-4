<template>
  <div class="page-stack">
    <section class="panel">
      <div class="panel-head">
        <h2>业主详情</h2>
        <button @click="goBack">返回</button>
      </div>
      <div v-if="room" class="owner-info-grid">
        <div class="info-card">
          <span class="label">房屋</span>
          <strong>{{ room.building_name }} - {{ room.room_no }}</strong>
        </div>
        <div class="info-card">
          <span class="label">业主姓名</span>
          <strong>{{ room.owner_name }}</strong>
        </div>
        <div class="info-card">
          <span class="label">联系电话</span>
          <strong>{{ room.phone || '-' }}</strong>
        </div>
        <div class="info-card">
          <span class="label">建筑面积</span>
          <strong>{{ room.area }} ㎡</strong>
        </div>
        <div class="info-card">
          <span class="label">入住状态</span>
          <strong>{{ room.is_active ? '已入住' : '未入住' }}</strong>
        </div>
      </div>
    </section>

    <section class="panel">
      <div class="panel-head">
        <h2>账单列表</h2>
        <div class="toolbar">
          <select v-model="billStatusFilter">
            <option value="all">全部</option>
            <option value="unpaid">待缴费</option>
            <option value="paid">已缴费</option>
            <option value="overdue">已逾期</option>
            <option value="cancelled">已作废</option>
          </select>
          <button @click="loadBills">刷新</button>
        </div>
      </div>
      <DataTable :columns="billColumns" :rows="filteredBills">
        <template #cell-status="{ row }">
          <StatusBadge :status="row.status" />
        </template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
        <template #actions="{ row }">
          <button v-if="row.status === 'unpaid' || row.status === 'overdue'" @click="payBill(row)">缴费</button>
        </template>
      </DataTable>
    </section>

    <section class="panel">
      <div class="panel-head">
        <h2>催缴时间线</h2>
        <button @click="loadTimeline">刷新</button>
      </div>
      <div class="bill-status-legend">
        <span class="legend-item"><span class="legend-dot unpaid"></span>未缴/逾期</span>
        <span class="legend-item"><span class="legend-dot paid"></span>已缴费</span>
      </div>
      <div v-if="timelineData.length" class="timeline">
        <div v-for="item in timelineData" :key="item.id" class="timeline-item" :class="item.bill_status">
          <div class="timeline-dot" :class="item.bill_status"></div>
          <div class="timeline-body" :class="item.bill_status">
            <div class="timeline-header">
              <strong>{{ formatDate(item.sent_at) }}</strong>
              <span class="channel-tag">{{ channelLabels[item.channel] || item.channel }}</span>
              <StatusBadge v-if="item.bill_status === 'paid'" :status="item.bill_status" />
              <StatusBadge v-else :status="item.bill_status" />
            </div>
            <p class="timeline-message">{{ item.message }}</p>
            <div class="timeline-bill-info">
              <span class="bill-tag">{{ item.bill_no }}</span>
              <span>{{ item.fee_name }} · {{ item.period }} · ¥{{ Number(item.amount).toFixed(2) }}</span>
            </div>
            <div class="timeline-meta">
              <span v-if="item.contact_result" class="contact-tag" :class="item.contact_result">{{ item.contact_result_display }}</span>
              <span v-else class="contact-tag none">未记录联系结果</span>
              <span v-if="item.next_follow_up" class="follow-up-tag">下次跟进: {{ formatDate(item.next_follow_up) }}</span>
            </div>
            <div class="timeline-actions">
              <button @click="openEditContact(item)">
                {{ item.contact_result ? '修改联系结果' : '记录联系结果' }}
              </button>
            </div>
          </div>
        </div>
      </div>
      <div v-else class="placeholder">暂无催缴记录</div>
    </section>

    <div v-if="showContactModal" class="modal-overlay" @click.self="showContactModal = false">
      <div class="modal">
        <h3>{{ editingReminder && editingReminder.contact_result ? '修改' : '记录' }}联系结果</h3>
        <div class="form-grid">
          <label>联系结果
            <select v-model="contactForm.contact_result">
              <option value="">请选择</option>
              <option value="reached_promise">已接听-承诺缴费</option>
              <option value="reached_refuse">已接听-拒绝缴费</option>
              <option value="unreached">未接听</option>
              <option value="empty_number">空号</option>
              <option value="other">其他</option>
            </select>
          </label>
          <label>下次跟进时间
            <div style="display:flex; gap:6px;">
              <input v-model="contactForm.next_follow_up" type="datetime-local" style="flex:1;" />
              <button type="button" v-if="contactForm.next_follow_up" class="btn-secondary" @click="contactForm.next_follow_up = ''">清空</button>
            </div>
          </label>
        </div>
        <div class="modal-actions">
          <button @click="showContactModal = false">取消</button>
          <button @click="submitContact" :disabled="!contactForm.contact_result">确认</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from "vue";
import { useRoute, useRouter } from "vue-router";
import { propertyApi } from "../api/property";
import DataTable from "../components/DataTable.vue";
import StatusBadge from "../components/StatusBadge.vue";

const route = useRoute();
const router = useRouter();

const room = ref(null);
const bills = ref([]);
const timelineData = ref([]);
const billStatusFilter = ref("all");
const showContactModal = ref(false);
const editingReminder = ref(null);
const contactForm = reactive({ id: null, contact_result: "", next_follow_up: "" });

const channelLabels = { sms: "短信", phone: "电话", wechat: "微信", notice: "纸质通知" };

const filteredBills = computed(() => {
  if (billStatusFilter.value === "all") return bills.value;
  return bills.value.filter(b => b.status === billStatusFilter.value);
});

const billColumns = [
  { key: "bill_no", label: "账单编号" },
  { key: "fee_name", label: "费用" },
  { key: "period", label: "账期" },
  { key: "amount", label: "金额" },
  { key: "due_date", label: "截止日期" },
  { key: "status", label: "状态" },
  { key: "paid_at", label: "缴费时间" }
];

function formatDate(value) {
  if (!value) return "-";
  return value.replace("T", " ").substring(0, 19);
}

function goBack() {
  router.back();
}

async function loadRoom() {
  room.value = await propertyApi.getRoom(route.params.id);
}

async function loadBills() {
  bills.value = await propertyApi.roomBills(route.params.id);
}

async function loadTimeline() {
  timelineData.value = await propertyApi.roomReminderTimeline(route.params.id);
}

function openEditContact(item) {
  editingReminder.value = item;
  contactForm.id = item.id;
  contactForm.contact_result = item.contact_result || "";
  contactForm.next_follow_up = item.next_follow_up
    ? item.next_follow_up.substring(0, 16)
    : "";
  showContactModal.value = true;
}

async function submitContact() {
  if (!contactForm.contact_result) return;
  const payload = { contact_result: contactForm.contact_result };
  if (contactForm.next_follow_up) {
    payload.next_follow_up = contactForm.next_follow_up;
  } else {
    payload.clear_next_follow_up = true;
  }
  await propertyApi.recordContact(contactForm.id, payload);
  showContactModal.value = false;
  await loadTimeline();
  await loadBills();
}

async function payBill(row) {
  if (confirm(`确认对账单 ${row.bill_no} 进行缴费？`)) {
    await propertyApi.payBill(row.id, { method: "wechat", payer: room.value.owner_name });
    await loadBills();
    await loadTimeline();
  }
}

async function load() {
  await Promise.all([loadRoom(), loadBills(), loadTimeline()]);
}

onMounted(load);
</script>
