<template>
  <default-style-select-dialog
    v-if="
      selectedCharacterInfo &&
      selectedStyleIndexes[selectedCharacterInfo.metas.speakerUuid] !==
        undefined
    "
    v-model:isOpen="showStyleSelectDialog"
    v-model:selectedStyleIndex="
      selectedStyleIndexes[selectedCharacterInfo.metas.speakerUuid]
    "
    :character-info="selectedCharacterInfo"
  />
  <q-dialog
    v-model="modelValueComputed"
    maximized
    transition-show="jump-up"
    transition-hide="jump-down"
    class="transparent-backdrop"
  >
    <q-layout container view="hHh Lpr lff" class="bg-background">
      <q-header class="q-py-sm">
        <q-toolbar>
          <div class="column">
            <q-toolbar-title class="text-display"
              >設定 / デフォルトスタイル・試聴</q-toolbar-title
            >
          </div>

          <q-space />

          <div class="row items-center no-wrap">
            <q-btn
              unelevated
              label="完了"
              color="toolbar-button"
              text-color="toolbar-button-display"
              class="text-no-wrap"
              @click="closeDialog"
            />
          </div>
        </q-toolbar>
      </q-header>

      <q-page-container>
        <q-page class="main">
          <div class="character-items-container">
            <div>
              <q-item
                v-for="speaker of speakerWithMultipleStyles"
                :key="speaker.metas.speakerUuid"
                v-ripple="isHoverableItem"
                clickable
                class="q-pa-none character-item"
                :class="[isHoverableItem && 'hoverable-character-item']"
                @click="openStyleSelectDialog(speaker)"
              >
                <div class="character-item-inner">
                  <img
                    :src="
                      speaker.metas.styles[
                        selectedStyleIndexes[speaker.metas.speakerUuid]
                      ].iconPath
                    "
                    class="style-icon"
                  />
                  <span class="text-subtitle1 q-ma-sm">{{
                    characterInfosMap[speaker.metas.speakerUuid].metas
                      .speakerName
                  }}</span>
                  <div
                    v-if="
                      characterInfosMap[speaker.metas.speakerUuid].metas.styles
                        .length > 1
                    "
                    class="style-select-container"
                  >
                    <span
                      >{{
                        selectedStyles[speaker.metas.speakerUuid]
                          ? selectedStyles[speaker.metas.speakerUuid].styleName
                          : "ノーマル"
                      }}（{{
                        characterInfosMap[speaker.metas.speakerUuid].metas
                          .styles.length
                      }}スタイル）</span
                    >
                  </div>
                </div>
              </q-item>
            </div>
          </div>
        </q-page>
      </q-page-container>
    </q-layout>
  </q-dialog>
</template>

<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { useStore } from "@/store";
import { CharacterInfo, SpeakerId, StyleInfo } from "@/type/preload";
import DefaultStyleSelectDialog from "@/components/DefaultStyleSelectDialog.vue";

const props =
  defineProps<{
    modelValue: boolean;
    characterInfos: CharacterInfo[];
  }>();
const emit =
  defineEmits<{
    (e: "update:modelValue", val: boolean): void;
  }>();

const store = useStore();

const modelValueComputed = computed({
  get: () => props.modelValue,
  set: (val) => emit("update:modelValue", val),
});

// 選択中のキャラクター
const selectedCharacter = ref(props.characterInfos[0].metas.speakerUuid);

const showStyleSelectDialog = ref<boolean>(false);
const selectedCharacterInfo = computed(() => {
  return props.characterInfos.find(
    (characterInfo) =>
      characterInfo.metas.speakerUuid === selectedCharacter.value
  );
});

const characterInfosMap = computed(() => {
  const map: { [key: SpeakerId]: CharacterInfo } = {};
  props.characterInfos.forEach((characterInfo) => {
    map[characterInfo.metas.speakerUuid] = characterInfo;
  });
  return map;
});

// サンプルボイス一覧のキャラクター順番
const speakerWithMultipleStyles = ref<CharacterInfo[]>([]);

// 選択中のスタイル
const selectedStyleIndexes = ref<Record<SpeakerId, number>>({});
const selectedStyles = computed(() => {
  const map: { [key: string]: StyleInfo } = {};
  props.characterInfos.forEach((characterInfo) => {
    const selectedStyleIndex: number | undefined =
      selectedStyleIndexes.value[characterInfo.metas.speakerUuid];
    map[characterInfo.metas.speakerUuid] =
      characterInfo.metas.styles[selectedStyleIndex ?? 0];
  });
  return map;
});

// ダイアログが開かれたときに初期値を求める
watch([() => props.modelValue], async ([newValue]) => {
  if (newValue) {
    speakerWithMultipleStyles.value = store.state.userCharacterOrder
      .map((speakerUuid) => characterInfosMap.value[speakerUuid])
      .filter((characterInfo) => characterInfo !== undefined)
      .filter(
        (characterInfo) => characterInfo.metas.styles.length > 1
      ) as CharacterInfo[];
    // FIXME: エンジン未起動状態でデフォルトスタイル選択ダイアログを開くと
    // 未起動エンジンのキャラのデフォルトスタイルが消えてしまう
    selectedStyleIndexes.value = Object.fromEntries(
      [
        ...store.state.userCharacterOrder.map(
          (speakerUuid) => [speakerUuid, 0] as const
        ),
        ...store.state.defaultStyleIds.map(
          (defaultStyle) =>
            [
              defaultStyle.speakerUuid,
              characterInfosMap.value[
                defaultStyle.speakerUuid
              ]?.metas.styles.findIndex(
                (style) => style.styleId === defaultStyle.defaultStyleId
              ),
            ] as const
        ),
      ].filter(([speakerUuid]) => speakerUuid in characterInfosMap.value)
    );
  }
});

// キャラクター枠のホバー状態を表示するかどうか
// 再生ボタンなどにカーソルがある場合はキャラクター枠のホバーUIを表示しないようにするため
const isHoverableItem = ref(true);

const closeDialog = () => {
  store.dispatch(
    "SET_DEFAULT_STYLE_IDS",
    Object.entries(selectedStyleIndexes.value).map(
      ([speakerUuidStr, styleIndex]) => {
        const speakerUuid = SpeakerId(speakerUuidStr);
        return {
          speakerUuid,
          defaultStyleId:
            characterInfosMap.value[speakerUuid].metas.styles[styleIndex]
              .styleId,
          engineId:
            characterInfosMap.value[speakerUuid].metas.styles[styleIndex]
              .engineId,
        };
      }
    )
  );
  stop();
  modelValueComputed.value = false;
};
const openStyleSelectDialog = (characterInfo: CharacterInfo) => {
  stop();
  selectedCharacter.value = characterInfo.metas.speakerUuid;
  showStyleSelectDialog.value = true;
};
</script>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.q-toolbar div:first-child {
  min-width: 0;
}

.main {
  height: calc(
    100vh - #{vars.$menubar-height + vars.$header-height +
      vars.$window-border-width}
  );

  display: flex;
  flex-direction: row;
}

.character-items-container {
  height: 100%;
  padding: 5px 16px;

  flex-grow: 1;

  display: flex;
  flex-direction: column;
  overflow-y: scroll;

  > div {
    $character-item-size: 200px;
    display: grid;
    grid-template-columns: repeat(auto-fit, $character-item-size);
    grid-auto-rows: 200px;
    column-gap: 10px;
    row-gap: 10px;
    align-content: center;
    justify-content: center;
    .character-item {
      box-shadow: 0 0 0 1px rgba(colors.$primary-light-rgb, 0.5);
      border-radius: 10px;
      overflow: hidden;
      &.selected-character-item {
        box-shadow: 0 0 0 2px colors.$primary-light;
      }
      &:hover :deep(.q-focus-helper) {
        opacity: 0 !important;
      }
      &.hoverable-character-item:hover :deep(.q-focus-helper) {
        opacity: 0.15 !important;
      }
      .character-item-inner {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        width: 100%;
        height: 100%;
        .style-icon {
          $icon-size: $character-item-size / 2;
          width: $icon-size;
          height: $icon-size;
          border-radius: 5px;
        }
        .style-select-container {
          display: flex;
          flex-direction: row;
          justify-content: center;
          align-items: center;
          margin-top: -1rem;
        }
      }
    }
  }
}

.q-layout-container > :deep(.absolute-full) {
  right: 0 !important;
  > .scroll {
    width: unset !important;
    overflow: hidden;
  }
}
</style>
