<template>
  <q-header class="q-py-sm">
    <q-toolbar>
      <template v-for="button in headerButtons" :key="button.text">
        <q-space v-if="button.text === null" />
        <q-btn
          v-else
          unelevated
          color="toolbar-button"
          text-color="toolbar-button-display"
          class="text-no-wrap text-bold q-mr-sm"
          :disable="button.disable.value"
          @click="button.click"
          >{{ button.text }}</q-btn
        >
      </template>
    </q-toolbar>
  </q-header>
</template>

<script setup lang="ts">
import { computed, ComputedRef } from "vue";
import {
  generateAndConnectAndSaveAudioWithDialog,
  generateAndSaveAllAudioWithDialog,
  generateAndSaveOneAudioWithDialog,
} from "./Dialog";
import { useStore } from "@/store";
import { setHotkeyFunctions } from "@/store/setting";
import {
  HotkeyAction,
  HotkeyReturnType,
  ToolbarButtonTagType,
} from "@/type/preload";
import { getToolbarButtonName } from "@/store/utility";

type ButtonContent = {
  text: string;
  click(): void;
  disable: ComputedRef<boolean>;
};

type SpacerContent = {
  text: null;
};

const store = useStore();

const uiLocked = computed(() => store.getters.UI_LOCKED);
const canUndo = computed(() => store.getters.CAN_UNDO);
const canRedo = computed(() => store.getters.CAN_REDO);
const activeAudioKey = computed(() => store.getters.ACTIVE_AUDIO_KEY);
const nowPlayingContinuously = computed(
  () => store.state.nowPlayingContinuously
);

const undoRedoHotkeyMap = new Map<HotkeyAction, () => HotkeyReturnType>([
  // undo
  [
    "元に戻す",
    () => {
      if (!uiLocked.value && canUndo.value) {
        undo();
      }
      return false;
    },
  ],
  // redo
  [
    "やり直す",
    () => {
      if (!uiLocked.value && canRedo.value) {
        redo();
      }
      return false;
    },
  ],
]);
setHotkeyFunctions(undoRedoHotkeyMap);

const hotkeyMap = new Map<HotkeyAction, () => HotkeyReturnType>([
  // play/stop continuously
  [
    "連続再生/停止",
    () => {
      if (!uiLocked.value) {
        if (nowPlayingContinuously.value) {
          stopContinuously();
        } else {
          playContinuously();
        }
      }
    },
  ],
]);

setHotkeyFunctions(hotkeyMap);

const undo = () => {
  store.dispatch("UNDO");
};
const redo = () => {
  store.dispatch("REDO");
};
const playContinuously = async () => {
  try {
    await store.dispatch("PLAY_CONTINUOUSLY_AUDIO");
  } catch (e) {
    let msg: string | undefined;
    // FIXME: GENERATE_AUDIO_FROM_AUDIO_ITEMのエラーを変えた場合変更する
    if (e instanceof Error && e.message === "VALID_MORPHING_ERROR") {
      msg = "モーフィングの設定が無効です。";
    } else {
      window.electron.logError(e);
    }
    store.dispatch("SHOW_ALERT_DIALOG", {
      title: "再生に失敗しました",
      message: msg ?? "エンジンの再起動をお試しください。",
    });
  }
};
const stopContinuously = () => {
  store.dispatch("STOP_CONTINUOUSLY_AUDIO");
};
const generateAndSaveOneAudio = async () => {
  if (activeAudioKey.value == undefined)
    throw new Error("activeAudioKey is undefined");
  await generateAndSaveOneAudioWithDialog({
    audioKey: activeAudioKey.value,
    dispatch: store.dispatch,
    disableNotifyOnGenerate: store.state.confirmedTips.notifyOnGenerate,
  });
};
const generateAndSaveAllAudio = async () => {
  await generateAndSaveAllAudioWithDialog({
    dispatch: store.dispatch,
    disableNotifyOnGenerate: store.state.confirmedTips.notifyOnGenerate,
  });
};
const generateAndConnectAndSaveAudio = async () => {
  await generateAndConnectAndSaveAudioWithDialog({
    dispatch: store.dispatch,
    disableNotifyOnGenerate: store.state.confirmedTips.notifyOnGenerate,
  });
};
const saveProject = async () => {
  await store.dispatch("SAVE_PROJECT_FILE", { overwrite: true });
};
const importTextFile = () => {
  store.dispatch("COMMAND_IMPORT_FROM_FILE", {});
};

const usableButtons: Record<
  ToolbarButtonTagType,
  Omit<ButtonContent, "text"> | null
> = {
  PLAY_CONTINUOUSLY: {
    click: playContinuously,
    disable: uiLocked,
  },
  STOP: {
    click: stopContinuously,
    disable: computed(() => !nowPlayingContinuously.value),
  },
  EXPORT_AUDIO_ONE: {
    click: generateAndSaveOneAudio,
    disable: computed(() => !activeAudioKey.value || uiLocked.value),
  },
  EXPORT_AUDIO_ALL: {
    click: generateAndSaveAllAudio,
    disable: uiLocked,
  },
  EXPORT_AUDIO_CONNECT_ALL: {
    click: generateAndConnectAndSaveAudio,
    disable: uiLocked,
  },
  SAVE_PROJECT: {
    click: saveProject,
    disable: uiLocked,
  },
  UNDO: {
    click: undo,
    disable: computed(() => !canUndo.value || uiLocked.value),
  },
  REDO: {
    click: redo,
    disable: computed(() => !canRedo.value || uiLocked.value),
  },
  IMPORT_TEXT: {
    click: importTextFile,
    disable: uiLocked,
  },
  EMPTY: null,
};

const headerButtons = computed(() =>
  store.state.toolbarSetting.map<ButtonContent | SpacerContent>((tag) => {
    const buttonContent = usableButtons[tag];
    if (buttonContent) {
      return {
        ...buttonContent,
        text: getToolbarButtonName(tag),
      };
    } else {
      return {
        text: null,
      };
    }
  })
);
</script>
