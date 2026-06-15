<template>
  <div class="page-stack">
    <section class="panel">
      <div class="panel-head">
        <h2>待催缴账单</h2>
        <div class="toolbar">
          <select v-model="channel">
            <option value="sms">短信</option>
            <option value="phone">电话</option>
            <option value="wechat">微信</option>
            <option value="notice">纸质通知</option>
          </select>
          <button @click="createReminders">生成催缴</button>
        </div>
      </div>
      <DataTable :columns="debtBillColumns" :rows="debtBills">
        <template #cell-status="{ row }"><StatusBadge :status="row.status" /></template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
        <template #cell-latest_contact="{ row }">{{ getLatestContact(row) }}</template>
        <template #cell-next_follow="{ row }">{{ getNextFollow(row) }}</template>
        <template #actions="{ row }">
          <button @click="openTimeline(row)">详情</button>
        </template>
      </DataTable>
    </section>

    <section class="panel">
      <div class="panel-head">
        <h2>催缴记录</h2>
        <div class="toolbar">
          <select v-model="reminderFilter">
            <option value="unpaid_only">仅未缴</option>
            <option value="all">全部</option>
          </select>
          <button @click="load">刷新</button>
        </div>
      </div>
      <DataTable :columns="reminderColumns" :rows="filteredReminders">
        <template #cell-status="{ row }"><StatusBadge :status="row.status" /></template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
        <template #cell-contact_result="{ row }">
          <span v-if="row.contact_result" class="contact-tag" :class="row.contact_result">{{ row.contact_result_display }}</span>
          <span v-else class="contact-tag none">未记录</span>
        </template>
        <template #cell-next_follow_up="{ row }">{{ row.next_follow_up || '-' }}</template>
        <template #actions="{ row }">
          <button @click="openContactForm(row)">{{ row.contact_result ? '修改' : '记录' }}</button>
          <button @click="openTimelineByRoom(row)">时间线</button>
        </template>
      </DataTable>
    </section>

    <div v-if="showContactModal" class="modal-overlay" @click.self="showContactModal = false">
      <div class="modal">
        <h3>{{ isEditingContact ? '修改' : '记录' }}联系结果</h3>
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
            <input v-model="contactForm.next_follow_up" type="datetime-local" />
          </label>
        </div>
        <div class="modal-actions">
          <button @click="showContactModal = false">取消</button>
          <button @click="submitContact" :disabled="!contactForm.contact_result">确认</button>
        </div>
      </div>
    </div>

    <div v-if="showTimelineModal" class="modal-overlay" @click.self="showTimelineModal = false">
      <div class="modal modal-wide">
        <div class="panel-head">
          <h3>{{ timelineOwner }} - 催缴时间线</h3>
          <button @click="showTimelineModal = false">关闭</button>
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
                <strong>{{ item.sent_at }}</strong>
                <span class="channel-tag">{{ channelLabels[item.channel] || item.channel }}</span>
                <StatusBadge v-if="item.bill_status === 'paid'" :status="item.bill_status" />
                <StatusBadge v-else :status="item.bill_status" />
              </div>
              <p>{{ item.message }}</p>
              <div class="timeline-meta">
                <span v-if="item.contact_result" class="contact-tag" :class="item.contact_result">{{ item.contact_result_display }}</span>
                <span v-else class="contact-tag none">未记录联系结果</span>
                <span v-if="item.next_follow_up" class="follow-up-tag">跟进: {{ item.next_follow_up }}</span>
              </div>
              <div class="timeline-bill">
                {{ item.bill_no }} | {{ item.fee_name }} | {{ item.period }} | ¥{{ Number(item.amount).toFixed(2) }}
              </div>
              <div class="timeline-actions">
                <button @click="openEditContactFromTimeline(item)">
                  {{ item.contact_result ? '修改联系结果' : '记录联系结果' }}
                </button>
              </div>
            </div>
          </div>
        </div>
        <div v-else class="placeholder">暂无催缴记录</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from "vue";
import { propertyApi } from "../api/property";
import DataTable from "../components/DataTable.vue";
import StatusBadge from "../components/StatusBadge.vue";

const bills = ref([]);
const reminders = ref([]);
const rooms = ref([]);
const channel = ref("sms");
const reminderFilter = ref("unpaid_only");
const showContactModal = ref(false);
const showTimelineModal = ref(false);
const contactForm = reactive({ id: null, contact_result: "", next_follow_up: "" });
const timelineData = ref([]);
const timelineOwner = ref("");
const currentTimelineRoomId = ref(null);

const channelLabels = { sms: "短信", phone: "电话", wechat: "微信", notice: "纸质通知" };

const isEditingContact = computed(() => {
  const reminder = reminders.value.find(r => r.id === contactForm.id);
  return reminder && reminder.contact_result;
});

const roomMap = computed(() => {
  const map = {};
  for (const room of rooms.value) {
    map[room.id] = room;
  }
  return map;
});

const debtBills = computed(() =>
  bills.value.filter((bill) => ["unpaid", "overdue"].includes(bill.status))
);

const filteredReminders = computed(() => {
  if (reminderFilter.value === "unpaid_only") {
    return reminders.value.filter((r) => ["unpaid", "overdue"].includes(r.bill_status));
  }
  return reminders.value;
});

const billReminderMap = computed(() => {
  const map = {};
  for (const r of reminders.value) {
    if (!map[r.bill]) {
      map[r.bill] = r;
    }
  }
  return map;
});

function getLatestContact(row) {
  const r = billReminderMap.value[row.id];
  if (!r || !r.contact_result) return "-";
  return r.contact_result_display;
}

function getNextFollow(row) {
  const r = billReminderMap.value[row.id];
  if (!r || !r.next_follow_up) return "-";
  return r.next_follow_up;
}

const debtBillColumns = [
  { key: "room_label", label: "房屋" },
  { key: "owner_name", label: "业主" },
  { key: "phone", label: "电话" },
  { key: "fee_name", label: "费用" },
  { key: "amount", label: "欠费" },
  { key: "due_date", label: "截止日期" },
  { key: "status", label: "状态" },
  { key: "latest_contact", label: "最近联系结果" },
  { key: "next_follow", label: "下次跟进" }
];

const reminderColumns = [
  { key: "reminder_no", label: "催缴编号" },
  { key: "room_label", label: "房屋" },
  { key: "owner_name", label: "业主" },
  { key: "channel", label: "渠道" },
  { key: "contact_result", label: "联系结果" },
  { key: "next_follow_up", label: "下次跟进" },
  { key: "status", label: "状态" }
];

async function load() {
  const [b, r, rm] = await Promise.all([
    propertyApi.listBills(),
    propertyApi.listReminders({ unpaid_only: reminderFilter.value === "unpaid_only" ? "true" : "" }),
    propertyApi.listRooms()
  ]);
  bills.value = b;
  reminders.value = r;
  rooms.value = rm;
}

async function createReminders() {
  await propertyApi.createOverdueReminders({ channel: channel.value });
  await load();
}

function openContactForm(row) {
  contactForm.id = row.id;
  contactForm.contact_result = row.contact_result || "";
  contactForm.next_follow_up = row.next_follow_up
    ? row.next_follow_up.substring(0, 16)
    : "";
  showContactModal.value = true;
}

function openEditContactFromTimeline(item) {
  contactForm.id = item.id;
  contactForm.contact_result = item.contact_result || "";
  contactForm.next_follow_up = item.next_follow_up
    ? item.next_follow_up.substring(0, 16)
    : "";
  showTimelineModal.value = false;
  showContactModal.value = true;
}

async function submitContact() {
  if (!contactForm.contact_result) return;
  const payload = { contact_result: contactForm.contact_result };
  if (contactForm.next_follow_up) {
    payload.next_follow_up = contactForm.next_follow_up;
  }
  await propertyApi.recordContact(contactForm.id, payload);
  showContactModal.value = false;
  await load();
  if (timelineOwner.value && currentTimelineRoomId) {
    timelineData.value = await propertyApi.roomReminderTimeline(currentTimelineRoomId);
    showTimelineModal.value = true;
  }
}

async function openTimeline(billRow) {
  const roomId = billRow.room;
  currentTimelineRoomId.value = roomId;
  const room = roomMap.value[roomId];
  timelineOwner.value = room ? `${room.building_name}-${room.room_no} ${room.owner_name}` : "";
  timelineData.value = await propertyApi.roomReminderTimeline(roomId);
  showTimelineModal.value = true;
}

async function openTimelineByRoom(reminderRow) {
  const roomId = reminderRow.room || findRoomIdByBill(reminderRow.bill);
  if (!roomId) return;
  currentTimelineRoomId.value = roomId;
  const room = roomMap.value[roomId];
  timelineOwner.value = room ? `${room.building_name}-${room.room_no} ${room.owner_name}` : "";
  timelineData.value = await propertyApi.roomReminderTimeline(roomId);
  showTimelineModal.value = true;
}

function findRoomIdByBill(billId) {
  const bill = bills.value.find((b) => b.id === billId);
  return bill ? bill.room : null;
}

onMounted(load);
</script>
