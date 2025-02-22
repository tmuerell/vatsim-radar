<template>
    <div
        class="atc-popup-container"
        :class="{
            'atc-popup-container--absolute': absolute,
            'atc-popup-container--small': small,
        }"
    >
        <common-popup-block class="atc-popup">
            <template
                v-if="$slots.title"
                #title
            >
                <slot name="title"/>
            </template>
            <template
                v-if="$slots.additionalTitle"
                #additionalTitle
            >
                <slot name="additionalTitle"/>
            </template>
            <div class="atc-popup_list">
                <common-info-block
                    v-for="(controller, controllerIndex) in controllers"
                    :key="controller.cid+controllerIndex"
                    class="atc-popup_atc"
                    is-button
                    :top-items="[
                        controller.callsign,
                        controller.name,
                        controller.frequency,
                        (showAtis && controller.atis_code) ? `Info ${ controller.atis_code }` : (!showAtis || !controller.text_atis?.length) ? controller.logon_time : undefined,
                    ]"
                    @click="[mapStore.addAtcOverlay(controller.callsign), emit('overlay')]"
                >
                    <template #top="{ item, index }">
                        <template v-if="index === 0">
                            <div
                                class="atc-popup__position"
                                :style="{ '--color': controllerColor(controller) ?? 'currentColor' }"
                            >
                                <div
                                    v-if="showFacility"
                                    class="atc-popup__position_facility"
                                    :style="{ background: getControllerPositionColor(controller) }"
                                >
                                    {{ controller.isATIS ? 'ATIS' : controller.facility === -2 ? 'CTAF' : dataStore.vatsim.data.facilities.value.find(x => x.id === controller.facility)?.short }}
                                </div>
                                <div class="atc-popup__position_name">
                                    {{ item }}
                                </div>
                            </div>
                        </template>
                        <template v-else-if="index === 1">
                            <common-spoiler type="controller">
                                <div
                                    class="atc-popup__controller"
                                    :style="{ '--color': controllerColor(controller) ?? 'currentColor' }"
                                >
                                    <div class="atc-popup__controller_name">
                                        {{ item }}
                                    </div>
                                    <common-blue-bubble class="atc-popup__controller_rating">
                                        {{
                                            dataStore.vatsim.data.ratings.value.find(x => x.id === controller.rating)?.short ?? ''
                                        }}
                                    </common-blue-bubble>
                                </div>

                                <template #name>
                                    Controller
                                </template>
                            </common-spoiler>
                        </template>
                        <template v-else-if="index === 2">
                            <div
                                class="atc-popup__frequency"
                                @click.stop="[copy(item as string), copiedFor = controller.callsign]"
                            >
                                <template v-if="!isCopied(controller.callsign)">
                                    {{ item }}

                                    <save-icon width="12"/>
                                </template>
                                <template v-else>
                                    Copied!
                                </template>
                            </div>
                        </template>
                        <template v-else-if="index === 3 && (!showAtis || !controller.text_atis?.length)">
                            <div class="atc-popup__time">
                                {{ getATCTime(controller) }}
                            </div>
                        </template>
                        <template v-else>
                            {{ item }}
                        </template>
                    </template>
                    <template
                        v-if="showAtis && controller.text_atis?.length"
                        #bottom
                    >
                        <ul class="atc-popup_atc__atis">
                            <li
                                v-for="atis in getATIS(controller)"
                                :key="atis"
                                class="atc-popup_atc__atis_line"
                            >
                                {{ parseEncoding(atis, controller.callsign) }}<br>
                            </li>
                        </ul>
                        <common-atc-time-online
                            v-if="controller.logon_time"
                            :controller="controller"
                        />
                    </template>
                </common-info-block>
            </div>
        </common-popup-block>
    </div>
</template>

<script setup lang="ts">
import type { PropType } from 'vue';
import type { VatsimShortenedController } from '~/types/data/vatsim';
import { parseEncoding } from '~/utils/data';
import { useMapStore } from '~/store/map';
import CommonBlueBubble from '~/components/common/basic/CommonBubble.vue';
import CommonPopupBlock from '~/components/common/popup/CommonPopupBlock.vue';
import CommonInfoBlock from '~/components/common/blocks/CommonInfoBlock.vue';
import CommonAtcTimeOnline from '~/components/common/vatsim/CommonAtcTimeOnline.vue';
import CommonSpoiler from '~/components/common/vatsim/CommonSpoiler.vue';
import SaveIcon from '@/assets/icons/kit/save.svg?component';
import { getUserList } from '~/composables/lists';
import { getStringColorFromSettings } from '~/composables/colors';

defineProps({
    controllers: {
        type: Array as PropType<VatsimShortenedController[]>,
        required: true,
    },
    showFacility: {
        type: Boolean,
        default: false,
    },
    showAtis: {
        type: Boolean,
        default: false,
    },
    absolute: {
        type: Boolean,
        default: false,
    },
    small: {
        type: Boolean,
        default: false,
    },
    maxHeight: {
        type: String,
        default: '400px',
    },
});

const emit = defineEmits({
    overlay() {
        return true;
    },
});
defineSlots<{ title?(): any; additionalTitle?(): any }>();
const dataStore = useDataStore();
const mapStore = useMapStore();
const { copy, copyState } = useCopyText();
const copiedFor = ref('');

const controllerColor = (controller: VatsimShortenedController) => {
    const list = getUserList(controller.cid);

    return (list && getStringColorFromSettings(list.color)) ?? undefined;
};

const isCopied = (key: string) => {
    return copiedFor.value === key && copyState.value;
};

const getATIS = (controller: VatsimShortenedController) => {
    if (!controller.isATIS) return controller.text_atis;
    if (controller.text_atis && controller.text_atis.filter(x => x.replaceAll(' ', '').length > 20).length > controller.text_atis.length - 2) return [controller.text_atis.join(' ')];
    return controller.text_atis;
};
</script>

<style scoped lang="scss">
.atc-popup {
    display: flex;
    flex-direction: column;
    gap: 4px;

    &-container {
        cursor: initial;

        z-index: 20;

        width: max-content;
        max-width: 450px;
        padding: 5px 0;

        &--small {
            max-width: min(450px, 100%);
        }

        &--absolute {
            position: absolute;
        }

        @include mobileOnly {
            max-width: 80vw;
        }
    }

    &_title {
        margin-bottom: 10px;
        font-weight: 600;
    }

    &__position {
        display: flex;
        gap: 8px;
        align-items: center;

        &_name {
            color: var(--color, currentColor);
        }

        &_facility {
            width: 40px;
            padding: 2px 4px;

            color: $lightgray0Orig;
            text-align: center;

            border-radius: 4px;
        }
    }

    &__controller {
        display: flex;
        gap: 8px;
        align-items: center;

        font-weight: 400;
        word-break: break-word;

        &_name {
            color: var(--color, currentColor);
        }
    }

    &__frequency {
        display: flex;
        gap: 4px;
        align-items: center;
        color: $primary400;
    }

    &__time {
        padding: 2px 4px;
        background: $darkgray950;
        border-radius: 4px;
    }

    &_list {
        overflow: auto;
        display: flex;
        flex-direction: column;
        gap: 8px;

        max-height: v-bind(maxHeight);
    }

    &_atc {
        @at-root .atc-popup-container:not(.atc-popup-container--small) & {
            :deep(.info-block__separator:nth-child(2)) {
                flex: 1 0 auto;

                svg {
                    display: none;
                }
            }

            :deep(.info-block_top) {
                flex-wrap: nowrap;
            }
        }


        &__atis {
            display: flex;
            flex-direction: column;
            gap: 5px;

            margin: 0;
            padding-left: 16px;

            word-break: break-word;

            &_line:only-child {
                margin-left: -16px;
                list-style: none;
            }
        }
    }
}
</style>
