<template>
  <span class="status-badge" :class="status">{{ labels[status] || status }}</span>
</template>

<script setup>
defineProps({ status: { type: String, required: true } });

const labels = {
  unpaid: "待缴费",
  paid: "已缴费",
  overdue: "已逾期",
  cancelled: "已作废",
  pending: "待发送",
  sent: "已发送",
  reached_promise: "承诺缴费",
  reached_refuse: "拒绝缴费",
  unreached: "未接听",
  empty_number: "空号",
  other: "其他"
};
</script>
