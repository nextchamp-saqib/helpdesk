<template>
  <HelpdeskTable
    v-model:selection="selection"
    :columns="columns"
    :data="tickets.list.data"
    :emit-row-click="true"
    :empty-message="emptyMessage"
    row-key="name"
    @row-click="toTicket"
  >
    <template #subject="{ data }">
      <TicketSummary
        class="col-subject"
        :name="data.name"
        :subject="data.subject"
        :communications="data.count_communication"
        :comments="data.count_comment"
        :seen="data._seen"
      />
    </template>
    <template #status="{ data }">
      <Badge
        :label="data.status"
        :theme="statusColormap[data.status]"
        variant="subtle"
      />
    </template>
    <template #priority="{ data }">
      <Badge
        :label="data.priority"
        :theme="priorityColormap[data.priority]"
        variant="subtle"
      />
    </template>
    <template #resolution_by="{ data }">
      <div
        :class="{
          'text-red-700': Date.parse(data.resolution_by) < Date.now(),
        }"
      >
        {{ data.resolution_by ? dayjs(data.resolution_by).fromNow() : "—" }}
      </div>
    </template>
    <template #creation="{ data }">
      {{ dayjs(data.creation).format(dateFormat) }}
    </template>
    <template #modified="{ data }">
      {{ dayjs(data.modified).format(dateFormat) }}
    </template>
    <template #via_customer_portal="{ data }">
      {{ data.via_customer_portal ? "Customer Portal" : "Email" }}
    </template>
    <template #row-extra="{ data }">
      <AssignedInfo :assign="data._assign" />
    </template>
    <template #actions="{ selection: s }">
      <Dropdown :options="assignOpts(s as Set<number>)">
        <template #default>
          <Button
            class="flex cursor-pointer items-center gap-1 text-gray-700"
            label="Assign"
            theme="gray"
            variant="ghost"
          >
            <template #prefix>
              <IconPlusCircle class="h-4 w-4" />
            </template>
          </Button>
        </template>
      </Dropdown>
    </template>
  </HelpdeskTable>
</template>

<script setup lang="ts">
import { useRouter } from "vue-router";
import { createResource, Badge, Dropdown } from "frappe-ui";
import dayjs from "dayjs";
import { AGENT_PORTAL_TICKET } from "@/router";
import { useAgentStore } from "@/stores/agent";
import { createToast } from "@/utils/toasts";
import HelpdeskTable from "@/components/HelpdeskTable.vue";
import { useTicketListStore } from "./data";
import AssignedInfo from "./AssignedInfo.vue";
import TicketSummary from "./TicketSummary.vue";
import IconPlusCircle from "~icons/lucide/plus-circle";

const router = useRouter();
const agentStore = useAgentStore();
const { selection, tickets } = useTicketListStore();
const emptyMessage =
  "🎉 Great news! There are currently no tickets to display. Keep up the good work!";
const dateFormat = "D/M/YYYY h:mm A";
const columns = [
  {
    title: "Subject",
    isTogglable: false,
    colKey: "subject",
    colClass: "col-subject",
  },
  {
    title: "Status",
    isTogglable: false,
    colKey: "status",
    colClass: "w-24",
  },
  {
    title: "Priority",
    isTogglable: false,
    colKey: "priority",
    colClass: "w-24",
  },
  {
    title: "Type",
    isTogglable: false,
    colKey: "ticket_type",
    colClass: "w-20",
  },
  {
    title: "Contact",
    isTogglable: false,
    colKey: "contact",
    colClass: "w-40",
  },
  {
    title: "Due in",
    isTogglable: false,
    colKey: "resolution_by",
    colClass: "w-24",
  },
  {
    title: "Customer",
    isTogglable: true,
    colKey: "customer",
    colClass: "w-40",
  },
  {
    title: "Created on",
    isTogglable: true,
    colKey: "creation",
    colClass: "w-36",
  },
  {
    title: "Last modified",
    isTogglable: true,
    colKey: "modified",
    colClass: "w-36",
  },
  {
    title: "Source",
    isTogglable: true,
    colKey: "via_customer_portal",
    colClass: "w-20",
  },
];

const bulkAssignTicketToAgent = createResource({
  url: "helpdesk.api.ticket.bulk_assign_ticket_to_agent",
  onSuccess: () => {
    createToast({
      title: "Tickets assigned to agent",
      icon: "check",
      iconClasses: "text-green-500",
    });
  },
  onError: () => {
    createToast({
      title: "Unable to assign tickets to agent.",
      icon: "x",
      iconClasses: "text-red-500",
    });
  },
});

const statusColormap = {
  Open: "red",
  Replied: "orange",
  Resolved: "green",
  Closed: "blue",
};

const priorityColormap = {
  Urgent: "red",
  High: "orange",
  Medium: "blue",
  Low: "green",
};

function assignOpts(selected: Set<number>) {
  return agentStore.options.map((a) => ({
    label: a.agent_name,
    onClick: () =>
      bulkAssignTicketToAgent.submit({
        ticket_ids: Array.from(selected),
        agent_id: a.name,
      }),
  }));
}

function toTicket(ticketId: string) {
  router.push({
    name: AGENT_PORTAL_TICKET,
    params: {
      ticketId,
    },
  });
}
</script>

<style scoped>
:deep(.col-subject) {
  width: 480px;
}
</style>
