<template>
  <v-hover v-model="isHovering">
    <template #default="{ props: hoverProps }">
      <div :id="props.id" class="mt-4 d-flex flex-row flex-grow-width" :class="nestedClass" @keydown.ctrl.alt.m.stop="commentBtnRef?.createComment()" v-bind="hoverProps">
        <div class="flex-grow-width">
          <!-- String -->
          <markdown-text-field
            v-if="definition.type === FieldDataType.STRING"
            v-model="formValue"
            :collab="props.collab"
            validate-on="input lazy"
            :spellcheck-supported="definition.spellcheck"
            :rules="[(v: string) => validateRegexPattern(v)]"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </markdown-text-field>

          <!-- Markdown -->
          <markdown-field
            v-else-if="definition.type === FieldDataType.MARKDOWN"
            v-model="formValue"
            :collab="props.collab"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
            <template v-if="$slots['markdown-context-menu']" #context-menu="slotData">
              <slot 
                name="markdown-context-menu"
                :id="props.id"
                :value="formValue"
                :definition="definition"
                v-bind="slotData"
              />
            </template>
          </markdown-field>

          <!-- Date -->
          <s-date-picker
            v-else-if="definition.type === FieldDataType.DATE"
            v-model="formValue"
            :locale="props.lang || undefined"
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-date-picker>

          <!-- Enum -->
          <s-autocomplete
            v-else-if="definition.type === FieldDataType.ENUM"
            v-model="formValue"
            :items="[{value: null as string|null, label: '---'}].concat(definition.choices!)"
            item-title="label"
            item-value="value"
            :clearable="!props.readonly"
            spellcheck="false"
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-autocomplete>

          <!-- Combobox -->
          <s-combobox
            v-else-if="definition.type === FieldDataType.COMBOBOX"
            v-model="formValue"
            :items="comboboxSuggestions"
            :clearable="!props.readonly"
            spellcheck="false"
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-combobox>

          <!-- Number -->
          <s-number-input
            v-else-if="definition.type === FieldDataType.NUMBER"
            :model-value="formValue"
            @update:model-value="emitUpdate($event)"
            :min="definition.minimum ?? undefined"
            :max="definition.maximum ?? undefined"
            clearable
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-number-input>

          <!-- Boolean -->
          <s-checkbox
            v-else-if="definition.type === FieldDataType.BOOLEAN"
            :model-value="formValue || false"
            @update:model-value="emitUpdate($event)"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-checkbox>

          <!-- CVSS -->
          <s-cvss-field
            v-else-if="definition.type === FieldDataType.CVSS"
            v-model="formValue"
            :cvss-version="definition.cvss_version"
            :disable-validation="props.disableValidation"
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-cvss-field>

          <!-- CWE -->
          <s-cwe-field
            v-else-if="definition.type === FieldDataType.CWE"
            v-model="formValue"
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-cwe-field>

          <!-- User -->
          <s-user-selection
            v-else-if="definition.type === FieldDataType.USER"
            :model-value="formValue"
            @update:model-value="emitUpdate(($event as UserShortInfo|null)?.id || null)"
            :selectable-users="selectableUsers"
            @focus="collabFocus"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-user-selection>

          <!-- JSON -->
          <s-codeblock-field
            v-else-if="definition.type === FieldDataType.JSON"
            :model-value="formValue"
            @update:model-value="emitUpdate($event)"
            :rules="[(v: string) => validateJson(v)]"
            @focus="collabFocus"
            variant="outlined"
            v-bind="fieldAttrs"
          >
            <template #label v-if="$slots.label"><slot name="label" /></template>
          </s-codeblock-field>

          <!-- Object -->
          <s-card v-else-if="definition.type === FieldDataType.OBJECT">
            <v-card-item class="pb-0">
              <v-card-title class="text-body-1"><slot name="label">{{ label }}</slot></v-card-title>
              <v-card-subtitle v-if="definition.help_text">{{ definition.help_text }}</v-card-subtitle>
              <template #append>
                <comment-btn
                  v-if="props.collab?.comments"
                  ref="commentBtnRef"
                  density="comfortable"
                  v-bind="commentBtnAttrs"
                  v-show="!mobile"
                />
              </template>
            </v-card-item>

            <v-card-text>
              <dynamic-input-field
                v-for="objectField in (props.definition.properties || [])"
                :key="objectField.id"
                :model-value="formValue[objectField.id]"
                @update:model-value="emitInputObject(objectField.id as string, $event)"
                :definition="objectField"
                :collab="collabSubpathProps[objectField.id]"
                :id="props.id ? (props.id + '.' + objectField.id) : undefined"
                :error-messages="props.errorMessages?.[objectField.id]"
                v-bind="inheritedAttrs"
              >
                <template v-for="name in inheritedSlots" #[name]="slotData: any"><slot :name="name" v-bind="slotData" /></template>
              </dynamic-input-field>
            </v-card-text>
          </s-card>

          <!-- List -->
          <s-card v-else-if="definition.type === FieldDataType.LIST">
            <v-card-item class="pb-0">
              <v-card-title class="text-body-1"><slot name="label">{{ label }}</slot></v-card-title>
              <v-card-subtitle v-if="definition.help_text">{{ definition.help_text }}</v-card-subtitle>
              <v-alert v-if="Array.isArray(props.errorMessages) && props.errorMessages.length > 0" color="error">
                <ul>
                  <li v-for="e in props.errorMessages" :key="e">{{ e }}</li>
                </ul>
              </v-alert>
              <template #append>
                <comment-btn
                  v-if="props.collab?.comments"
                  ref="commentBtnRef"
                  density="comfortable"
                  v-bind="commentBtnAttrs"
                  v-show="!mobile"
                />
                <s-btn-icon
                  v-if="definition.items!.type === FieldDataType.STRING"
                  @click="bulkEditList = !bulkEditList"
                  density="comfortable"
                >
                  <v-icon v-if="bulkEditList" icon="mdi-format-list-bulleted" />
                  <v-icon v-else icon="mdi-playlist-edit" />
                  <s-tooltip activator="parent">
                    <span v-if="bulkEditList">Edit as list</span>
                    <span v-else>Bulk edit list items</span>
                  </s-tooltip>
                </s-btn-icon>
              </template>
            </v-card-item>

            <v-card-text>
              <!-- Bulk edit list items of list[string] -->
              <v-textarea
                v-if="definition.items!.type === FieldDataType.STRING && bulkEditList"
                :model-value="(formValue || []).join('\n')"
                @update:model-value="emitInputStringList"
                auto-grow
                hide-details="auto"
                spellcheck="false"
                variant="outlined"
                v-bind="fieldAttrs"
                label="Enter one item per line"
                class="mt-4"
              />
              <v-list v-else class="pa-0 bg-inherit">
                <draggable
                  :model-value="formValue.map((value: any, index: number) => ({value, index}))"
                  @change="e => e.moved ? emitInputList('move', e.moved.oldIndex, e.moved.newIndex) : undefined"
                  :item-key="(item: any) => item.index"
                  :disabled="props.disabled || props.readonly"
                  handle=".draggable-handle"
                >
                  <template #item="{element: {value: entryVal}, index: entryIdx}">
                    <v-list-item class="pa-0">
                      <template #default>
                        <dynamic-input-field
                          :model-value="entryVal"
                          @update:model-value="emitInputList('update', entryIdx as number, $event)"
                          :definition="props.definition.items!"
                          :collab="collabSubpathProps[`[${entryIdx}]`]"
                          @keydown="onListKeyDown"
                          :id="id ? `${id}[${entryIdx}]` : undefined"
                          :error-messages="props.errorMessages?.[entryIdx]"
                          v-bind="inheritedAttrs"
                        >
                          <template v-for="(_, name) in inheritedSlots" #[name]="slotData: any"><slot :name="name" v-bind="slotData" /></template>
                        </dynamic-input-field>
                      </template>
                      <template #append>
                        <div 
                          v-if="[FieldDataType.MARKDOWN, FieldDataType.OBJECT, FieldDataType.LIST].includes(props.definition.items!.type as any)"
                          class="d-flex flex-column"
                        >
                          <btn-delete
                            :delete="() => emitInputList('delete', entryIdx as number)"
                            :confirm="!isEmptyOrDefault(entryVal, definition.items!)"
                            :disabled="props.disabled || props.readonly"
                            density="compact"
                            button-variant="icon"
                            class="mb-4"
                          />

                          <s-btn-icon 
                            @click="emitInputList('move', entryIdx, entryIdx - 1)"
                            :disabled="props.disabled || props.readonly || entryIdx === 0"
                            density="compact"
                          >
                            <v-icon icon="mdi-arrow-up-drop-circle-outline" />
                            <s-tooltip activator="parent" text="Move up in list" />
                          </s-btn-icon>
                          <s-btn-icon
                            @click="emitInputList('move', entryIdx, entryIdx + 1)"
                            :disabled="props.disabled || props.readonly || entryIdx === formValue.length - 1"
                            density="compact"
                          >
                            <v-icon icon="mdi-arrow-down-drop-circle-outline" />
                            <s-tooltip activator="parent" text="Move down in list" />
                          </s-btn-icon>
                        </div>
                        <div v-else>
                          <v-icon
                            size="x-large"
                            class="draggable-handle" 
                            icon="mdi-drag-horizontal" 
                            :disabled="props.disabled || props.readonly"
                            v-show="!mobile"
                          />
                          <btn-delete
                            :delete="() => emitInputList('delete', entryIdx as number)"
                            :confirm="false"
                            :disabled="props.disabled || props.readonly"
                            button-variant="icon"
                          />
                        </div>
                      </template>
                    </v-list-item>
                  </template>
                </draggable>

                <v-list-item class="pa-0">
                  <s-btn-secondary
                    @click="emitInputList('add')"
                    :disabled="props.disabled || props.readonly"
                    prepend-icon="mdi-plus"
                    text="Add"
                  />
                </v-list-item>
              </v-list>
            </v-card-text>
          </s-card>

          <div v-else>
            {{ definition }}
          </div>
        </div>
        <comment-btn
          v-if="props.collab?.comments && ![FieldDataType.MARKDOWN, FieldDataType.LIST, FieldDataType.OBJECT].includes(definition.type as any)"
          ref="commentBtnRef"
          v-bind="commentBtnAttrs"
          v-show="!mobile"
        />
      </div>
    </template>
  </v-hover>
</template>

<script setup lang="ts">
import Draggable from 'vuedraggable';
import { isEqual, omit, pick, uniq } from 'lodash-es';
import regexWorkerUrl from '~/workers/regexWorker?worker&url';
import { collabSubpath, type MarkdownEditorMode, FieldDataType, type MarkdownProps, type FieldDefinition, type UserShortInfo, useCollabSubpaths } from '#imports';
import { workerUrlPolicy } from '~/plugins/trustedtypes';
import { wait } from '@base/utils/helpers';

defineOptions({
  inheritAttrs: false,
});

const props = defineProps<MarkdownProps & {
  modelValue?: any;
  definition: FieldDefinition;
  collab?: CollabPropType;
  id?: string;
  showFieldIds?: boolean;
  selectableUsers?: UserShortInfo[];
  fieldValueSuggestions?: Map<string, string[]>;
  disabled?: boolean;
  readonly?: boolean;
  autofocus?: boolean;
  disableValidation?: boolean;
  nestingLevel?: number;
  errorMessages?: any;
}>();
const emit = defineEmits<{
  'update:modelValue': [value: any];
  'collab': [value: any];
  'comment': [value: any];
  'search': [value: string];
  'update:spellcheckEnabled': [value: boolean];
  'update:markdownEditorMode': [value: MarkdownEditorMode];
}>();
defineSlots();


const { mobile } = useDisplay();


function getInitialValue(fieldDef: FieldDefinition, useDefault = true): any {
  if (fieldDef.default && useDefault) {
    return fieldDef.default;
  } else if (fieldDef.type === "list") {
    return [];
  } else if (fieldDef.type === 'object') {
    return Object.fromEntries(fieldDef.properties!.map(d => [d.id, getInitialValue(d, useDefault)]));
  } else {
    return null;
  }
}
function emitUpdate(val: any, options?: { preventCollabEvent?: boolean }) {
  if (props.collab && !options?.preventCollabEvent) {
    emit('collab', {
      type: CollabEventType.UPDATE_KEY,
      path: props.collab.path,
      value: val,
      updateAwareness: true,
    });
  }
  emit('update:modelValue', val);
}
function emitInputObject(objectFieldId: string, val: any) {
  emitUpdate({ ...formValue.value, [objectFieldId]: val }, { preventCollabEvent: true });
}
async function emitInputList(action: string, entryIdx?: number, entryVal: any|number|null = null) {
  const newVal = [...formValue.value];
  if (action === "update") {
    newVal[entryIdx!] = entryVal;
  } else if (action === "delete") {
    newVal.splice(entryIdx!, 1);
    if (props.collab) {
      if (isListEditingLocked.value) {
        const confirmed = await collabConfirmToast();
        if (!confirmed) {
          return;
        }
      }

      emit('collab', {
        type: CollabEventType.DELETE,
        path: collabSubpath(props.collab, `[${entryIdx}]`).path,
      });
    }
  } else if (action === 'add') {
    if (entryVal === null) {
      entryVal = getInitialValue(props.definition.items!);
    }

    newVal.push(entryVal);
    if (props.collab) {
      emit('collab', {
        type: CollabEventType.CREATE,
        path: props.collab.path,
        value: entryVal,
      });
    }
  } else if (action === 'move') {
    const [moved] = newVal.splice(entryIdx!, 1);
    newVal.splice(entryVal!, 0, moved);

    if (props.collab) {
      if (isListEditingLocked.value) {
        const confirmed = await collabConfirmToast();
        if (!confirmed) {
          return;
        }
      }
      
      emit('collab', {
        type: CollabEventType.UPDATE_KEY,
        path: props.collab.path,
        value: newVal,
        sort: [{ id: entryIdx!, order: entryVal! }],
      });
    }
  }
  emitUpdate(newVal, { preventCollabEvent: true });
}

const formValue = computed({
  get: () => {
    if (props.modelValue === null || props.modelValue === undefined) {
      return getInitialValue(props.definition, false);
    }
    return props.modelValue;
  },
  set: val => emitUpdate(val, { 
    preventCollabEvent: [
      FieldDataType.MARKDOWN, 
      FieldDataType.STRING, 
      FieldDataType.OBJECT,
      FieldDataType.LIST,
    ].includes(props.definition.type) 
  }),
});
const label = computed(() => {
  let out = props.definition.label || props.definition.id || '';
  if (props.showFieldIds && props.id) {
    if (out) {
      out += ' (' + props.id + ')';
    } else {
      out = props.id;
    }
  }
  return out;
})

function isEmptyOrDefault(value: any, definition: FieldDefinition): boolean {
  if (definition.type === FieldDataType.LIST) {
    return !value || value.length === 0 || isEqual(value, definition.default) || value.every((v: any) => isEmptyOrDefault(v, definition.items!));
  } else if (definition.type === FieldDataType.OBJECT) {
    return !value || definition.properties!.every(d => isEmptyOrDefault(value[d.id], d));
  } else {
    return !value || value === definition.default;
  }
}

// Regex validation
// Global worker instance reused for all components
const regexWorker = useState<Worker|null>('regexWorker', () => null);
async function validateRegexPattern(value: string) {
  if (props.definition.type !== FieldDataType.STRING || !props.definition.pattern || !value) {
    return true;
  }

  try {
    const pattern = new RegExp(props.definition.pattern);

    if (!regexWorker.value) {
      regexWorker.value = new Worker(workerUrlPolicy.createScriptURL!(regexWorkerUrl) as string, { type: 'module' });
    }

    const threadedRegexMatch = new Promise((resolve) => {
      regexWorker.value!.addEventListener('message', (e: MessageEvent) => resolve(e.data), { once: true });
      regexWorker.value!.postMessage({ pattern, value }); // Send the string and regex to the worker
    });
    async function timeoutReject(timeoutMs: number) {
      await wait(timeoutMs);
      throw new Error("RegEx timeout");
    }
    try {
      // Execute regex in a web worker with timeout to prevent RegEx DoS
      const res = await Promise.race([threadedRegexMatch, timeoutReject(500)]);
      if (!res) {
        return `Invalid format. Value does not match pattern ${pattern}`;
      }
    } catch (e: any) {
      const w = regexWorker.value;
      regexWorker.value = null;
      await w.terminate();
      return e.message;
    }
  } catch (e: any) {
    return e.message;
  }
  return true;
}
onBeforeUnmount(async () => {
  if (regexWorker.value) {
    await regexWorker.value.terminate();
    regexWorker.value = null;
  }
})

// JSON validation
function validateJson(v: string) {
  if (props.definition.type !== FieldDataType.JSON || !v) {
    return true;
  }

  try {
    JSON.parse(v);
  } catch (e: any) {
    return `Invalid JSON: ${e.message}`;
  }

  return true;
}

const bulkEditList = ref(false);
function emitInputStringList(valuesListString?: string) {
  const values = (valuesListString || '').split('\n').filter(v => !!v);
  emitUpdate(values, { preventCollabEvent: false });
}
function onListKeyDown(event: KeyboardEvent) {
  // Disable vuetify list keyboard navigation
  if (['ArrowDown', 'ArrowUp', 'Home', 'End'].includes(event.key)) {
    event.stopPropagation();
  }
}
const isListEditingLocked = computed(() => {
  // Other users are currently editing the list.
  // Some collaborative list operations are not race-condition safe and should not be locked.
  return props.definition.type === FieldDataType.LIST && props.collab && props.collab.clients.some(c => !c.isSelf);
});

const comboboxSuggestions = computedCached(() => {
  const suggestions = [] as string[];
  if (props.definition.suggestions) {
    suggestions.push(...props.definition.suggestions);
  }
  if (props.fieldValueSuggestions && props.id) {
    suggestions.push(...props.fieldValueSuggestions.get(props.id.replaceAll(/\[\d+\]/g, '[]')) || []);
  }
  return uniq(suggestions);
});

const collabSubpathNames = computedCached(() => {
  if (props.definition.type === FieldDataType.LIST) {
    return formValue.value.map((_: any, i: number) => `[${i}]`);
  } else if (props.definition.type === FieldDataType.OBJECT) {
    return props.definition.properties?.map(d => d.id) || [];
  } else {
    return [];
  }
});
const collabSubpathProps = useCollabSubpaths(() => props.collab, collabSubpathNames);
function collabFocus() {
  if (props.collab) {
    emit('collab', {
      type: CollabEventType.AWARENESS,
      path: props.collab.path,
    });
  }
}

const nestedClass = computed(() => {
  if ([FieldDataType.OBJECT, FieldDataType.LIST].includes(props.definition.type)) {
    return (props.nestingLevel || 0) % 2 === 0 ? 'field-highlight-nested1' : 'field-highlight-nested2';
  }
  return undefined;
});

const attrs = useAttrs() as any;
const fieldAttrs = computed(() => ({
  ...attrs,
  label: label.value,
  hint: props.definition.help_text || undefined,
  ...pick(props, [
    'disabled', 'readonly', 'errorMessages', 'autofocus', 'lang', 'spellcheckEnabled', 'markdownEditorMode', 
    'referenceItems', 'rewriteFileUrlMap', 'uploadFile',
  ]),
  onCollab: (v: any) => emit('collab', v),
  onComment: (v: any) => emit('comment', v),
  onSearch: (v: any) => emit('search', v),
  'onUpdate:spellcheckEnabled': (v: boolean) => emit('update:spellcheckEnabled', v),
  'onUpdate:markdownEditorMode': (v: MarkdownEditorMode) => emit('update:markdownEditorMode', v),
}));
const inheritedAttrs = computed(() => ({
  ...omit(fieldAttrs.value, ['errorMessages']),
  ...pick(props, ['selectableUsers', 'disableValidation', 'showFieldIds', 'fieldValueSuggestions']),
  nestingLevel: (props.nestingLevel || 0) + 1,
}));
const slots = useSlots() as any;
const inheritedSlots = computed(() => Object.keys(slots).filter(name => !['label'].includes(name)));

const isHovering = ref(false);
const comments = computedList(() => props.collab?.comments?.filter(c => c.collabPath === props.collab?.path) || [], c => c.id);
const commentBtnAttrs = computed(() => ({
  comments: comments.value,
  onComment: (v: any) => emit('comment', v),
  collabPath: props.collab?.path || '',
  isHovering: isHovering.value,
  disabled: props.disabled || props.readonly,
}));
const commentBtnRef = useTemplateRef('commentBtnRef');
</script>

<style lang="scss" scoped>
.draggable-handle {
  cursor: grab;
}
.v-list-item:deep(.v-list-item__append .v-list-item__spacer) {
  width: 0.5em;
}

.visible-on-hover {
  opacity: 0;
}
.dynamic-input-field-wrapper:hover .visible-on-hover {
  opacity: 1;
}
</style>
