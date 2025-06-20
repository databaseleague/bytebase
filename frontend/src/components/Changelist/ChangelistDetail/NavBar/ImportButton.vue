<template>
  <UploadFilesButton @update="onUploadFiles">
    <template #trigger="{ onClick }">
      <NTooltip>
        <template #trigger>
          <NButton icon style="--n-padding: 0 10px" @click="onClick">
            <template #icon>
              <UploadIcon class="w-4 h-4" />
            </template>
          </NButton>
        </template>
        <template #default>
          <div class="whitespace-nowrap">
            {{ $t("changelist.import.upload-sql-or-zip-file") }}
          </div>
        </template>
      </NTooltip>
    </template>
  </UploadFilesButton>
</template>

<script setup lang="ts">
import { UploadIcon } from "lucide-vue-next";
import { NButton, NTooltip } from "naive-ui";
import { useI18n } from "vue-i18n";
import UploadFilesButton from "@/components/UploadFilesButton.vue";
import { pushNotification, useChangelistStore, useSheetV1Store } from "@/store";
import {
  Changelist_Change as Change,
  Changelist,
} from "@/types/proto/v1/changelist_service";
import { Sheet } from "@/types/proto/v1/sheet_service";
import { setSheetStatement } from "@/utils";
import { useChangelistDetailContext } from "../context";

const { t } = useI18n();
const { changelist, project } = useChangelistDetailContext();

const onUploadFiles = async (
  statementMap: { filename: string; statement: string }[]
) => {
  const createdSheets = await Promise.all(
    statementMap.map(async (m) => {
      const sheet = Sheet.fromPartial({
        title: m.filename,
      });
      setSheetStatement(sheet, m.statement);
      const created = await useSheetV1Store().createSheet(
        project.value.name,
        sheet
      );
      return created;
    })
  );
  const newChanges = createdSheets.map((sheet) =>
    Change.fromPartial({
      sheet: sheet.name,
    })
  );
  const changelistPatch = Changelist.fromPartial({
    ...changelist.value,
    changes: [...changelist.value.changes, ...newChanges],
  });
  await useChangelistStore().patchChangelist(changelistPatch, ["changes"]);
  pushNotification({
    module: "bytebase",
    style: "SUCCESS",
    title: t("common.updated"),
  });
};
</script>
