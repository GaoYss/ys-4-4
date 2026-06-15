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
        <template #cell-latest_contact="{ row }">
          <span v-if="row.latest_contact_result" class="contact-tag" :class="row.latest_contact_result">{{ row.latest_contact_result_display }}</span>
          <span v-else class="contact-tag none">未记录</span>
        </template>
        <template #cell-next_follow="{ row }">{{ row.latest_next_follow_up || '-' }}</template>
        <template #actions="{ row }">
          <button @click="goToOwner(row.room)">详情</button>
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
          <button @click="goToOwner(row.room)">业主详情</button>
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
import { useRouter } from "vue-router";
import { propertyApi } from "../api/property";
import DataTable from "../components/DataTable.vue";
import StatusBadge from "../components/StatusBadge.vue";

const CONTACT_RESULT_LABELS = {
  reached_promise: "已接听-承诺缴费",
  reached_refuse: "已接听-拒绝缴费",
  unreached: "未接听",
  empty_number: "空号",
  other: "其他"
};

const router = useRouter();

const bills = ref([]);
const reminders = ref([]);
const channel = ref("sms");
const reminderFilter = ref("unpaid_only");
const showContactModal = ref(false);
const contactForm = reactive({ id: null, contact_result: "", next_follow_up: "" });

const channelLabels = { sms: "短信", phone: "电话", wechat: "微信", notice: "纸质通知" };

const isEditingContact = computed(() => {
  const reminder = reminders.value.find(r => r.id === contactForm.id);
  return reminder && reminder.contact_result;
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

function goToOwner(roomId) {
  router.push(`/owners/${roomId}`);
}

async function load() {
  const [b, r] = await Promise.all([
    propertyApi.listBills(),
    propertyApi.listReminders({ unpaid_only: reminderFilter.value === "unpaid_only" ? "true" : "" })
  ]);
  bills.value = b;
  reminders.value = r;
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

async function submitContact() {
  if (!contactForm.contact_result) return;
  const payload = { contact_result: contactForm.contact_result };
  if (contactForm.next_follow_up) {
    payload.next_follow_up = contactForm.next_follow_up;
  } else {
    payload.clear_next_follow_up = true;
  }
  const updated = await propertyApi.recordContact(contactForm.id, payload);

  const rIdx = reminders.value.findIndex(r => r.id === updated.id);
  if (rIdx !== -1) {
    reminders.value.splice(rIdx, 1, { ...reminders.value[rIdx], ...updated });
  }

  const billId = updated.bill;
  const billIdx = bills.value.findIndex(b => b.id === billId);
  if (billIdx !== -1) {
    const bill = bills.value[billIdx];
    let latestContact = null;
    let latestContactDisplay = null;
    let latestFollow = null;
    const billReminders = reminders.value
      .filter(r => r.bill === billId)
      .slice()
      .sort((a, b) => new Date(b.sent_at) - new Date(a.sent_at));
    for (const r of billReminders) {
      if (r.contact_result && !latestContact) {
        latestContact = r.contact_result;
        latestContactDisplay = r.contact_result_display || CONTACT_RESULT_LABELS[r.contact_result];
      }
      if (r.next_follow_up && !latestFollow) {
        latestFollow = r.next_follow_up;
      }
      if (latestContact && latestFollow) break;
    }
    bills.value.splice(billIdx, 1, {
      ...bill,
      latest_contact_result: latestContact,
      latest_contact_result_display: latestContactDisplay,
      latest_next_follow_up: latestFollow
    });
  }

  showContactModal.value = false;
}

onMounted(load);
</script>
