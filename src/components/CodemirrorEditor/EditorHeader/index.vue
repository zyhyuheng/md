<script setup lang="ts">
import type { Format } from 'vue-pick-colors'
import { Toaster } from '@/components/ui/sonner'
import {
  altSign,
  codeBlockThemeOptions,
  colorOptions,
  ctrlKey,
  ctrlSign,
  fontFamilyOptions,
  fontSizeOptions,
  legendOptions,
  shiftSign,
  themeOptions,
} from '@/config'
import { useDisplayStore, useStore } from '@/stores'
import {
  addPrefix,
  processClipboardContent,
} from '@/utils'
import {
  ChevronDownIcon,
  Moon,
  PanelLeftClose,
  PanelLeftOpen,
  Settings,
  Sun,
} from 'lucide-vue-next'
import PickColors from 'vue-pick-colors'

const emit = defineEmits([
  `addFormat`,
  `formatContent`,
  `startCopy`,
  `endCopy`,
])

const formatItems = [
  {
    label: `加粗`,
    kbd: [ctrlSign, `B`],
    emitArgs: [`addFormat`, `${ctrlKey}-B`],
  },
  {
    label: `斜体`,
    kbd: [ctrlSign, `I`],
    emitArgs: [`addFormat`, `${ctrlKey}-I`],
  },
  {
    label: `删除线`,
    kbd: [ctrlSign, `D`],
    emitArgs: [`addFormat`, `${ctrlKey}-D`],
  },
  {
    label: `超链接`,
    kbd: [ctrlSign, `K`],
    emitArgs: [`addFormat`, `${ctrlKey}-K`],
  },
  {
    label: `行内代码`,
    kbd: [ctrlSign, `E`],
    emitArgs: [`addFormat`, `${ctrlKey}-E`],
  },
  {
    label: `格式化`,
    kbd: [altSign, shiftSign, `F`],
    emitArgs: [`formatContent`],
  },
] as const

const store = useStore()
const displayStore = useDisplayStore()

const { isDark, isCiteStatus, output, primaryColor } = storeToRefs(store)

const { toggleDark, editorRefresh, citeStatusChanged } = store

const copyMode = useStorage(addPrefix(`copyMode`), `txt`)
const source = ref(``)
const { copy: copyContent } = useClipboard({ source })

// 复制到微信公众号
function copy() {
  emit(`startCopy`)
  setTimeout(() => {
    // 如果是深色模式，复制之前需要先切换到白天模式
    const isBeforeDark = isDark.value
    if (isBeforeDark) {
      toggleDark()
    }

    nextTick(async () => {
      processClipboardContent(primaryColor.value)
      const clipboardDiv = document.getElementById(`output`)!
      clipboardDiv.focus()
      window.getSelection()!.removeAllRanges()
      const temp = clipboardDiv.innerHTML
      if (copyMode.value === `txt`) {
        const range = document.createRange()
        range.setStartBefore(clipboardDiv.firstChild!)
        range.setEndAfter(clipboardDiv.lastChild!)
        window.getSelection()!.addRange(range)
        document.execCommand(`copy`)
        window.getSelection()!.removeAllRanges()
      }
      clipboardDiv.innerHTML = output.value
      if (isBeforeDark) {
        nextTick(() => toggleDark())
      }
      if (copyMode.value === `html`) {
        await copyContent(temp)
      }

      // 输出提示
      toast.success(
        copyMode.value === `html`
          ? `已复制 HTML 源码，请进行下一步操作。`
          : `已复制渲染后的文章到剪贴板，可直接到公众号后台粘贴。`,
      )

      editorRefresh()
      emit(`endCopy`)
    })
  }, 350)
}

function customStyle() {
  displayStore.toggleShowCssEditor()
  setTimeout(() => {
    store.cssEditor!.refresh()
  }, 50)
}

const pickColorsContainer = useTemplateRef<HTMLElement | undefined>(
  `pickColorsContainer`,
)
const format = ref<Format>(`rgb`)
const formatOptions = ref<Format[]>([`rgb`, `hex`, `hsl`, `hsv`])
</script>

<template>
  <header class="header-container h-15 flex items-center px-5">
    <Menubar class="menubar mr-auto">
      <FileDropdown />

      <MenubarMenu>
        <MenubarTrigger> 格式 </MenubarTrigger>
        <MenubarContent class="w-60" align="start">
          <MenubarCheckboxItem
            v-for="{ label, kbd, emitArgs } in formatItems"
            :key="label"
            @click="
              emitArgs[0] === 'addFormat'
                ? $emit(emitArgs[0], emitArgs[1])
                : $emit(emitArgs[0])
            "
          >
            {{ label }}
            <MenubarShortcut>
              <kbd
                v-for="item in kbd"
                :key="item"
                class="mx-1 bg-gray-2 dark:bg-stone-9"
              >
                {{ item }}
              </kbd>
            </MenubarShortcut>
          </MenubarCheckboxItem>
          <MenubarSeparator />
          <MenubarCheckboxItem
            :checked="isCiteStatus"
            @click="citeStatusChanged()"
          >
            微信外链转底部引用
          </MenubarCheckboxItem>
        </MenubarContent>
      </MenubarMenu>
      <EditDropdown />
      <StyleDropdown />
      <HelpDropdown />
    </Menubar>

    <Button
      v-if="!store.isOpenPostSlider"
      variant="outline"
      class="mr-2"
      @click="store.isOpenPostSlider = true"
    >
      <PanelLeftOpen class="size-4" />
    </Button>
    <Button
      v-else
      variant="outline"
      class="mr-2"
      @click="store.isOpenPostSlider = false"
    >
      <PanelLeftClose class="size-4" />
    </Button>
    <Popover>
      <PopoverTrigger>
        <Button variant="outline">
          <Settings class="h-4 w-4" />
        </Button>
      </PopoverTrigger>
      <PopoverContent class="h-100 w-100 overflow-auto px-6" align="end">
        <div class="space-y-4">
          <div class="space-y-2">
            <h2>主题</h2>
            <div class="grid grid-cols-3 justify-items-center gap-2">
              <Button
                v-for="{ label, value } in themeOptions"
                :key="value"
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': store.theme === value,
                }"
                @click="store.themeChanged(value)"
              >
                {{ label }}
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>字体</h2>
            <div class="grid grid-cols-3 justify-items-center gap-2">
              <Button
                v-for="{ label, value } in fontFamilyOptions"
                :key="value"
                variant="outline"
                class="w-full"
                :class="{
                  'border-black dark:border-white': store.fontFamily === value,
                }"
                @click="store.fontChanged(value)"
              >
                {{ label }}
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>字号</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                v-for="{ value, desc } in fontSizeOptions"
                :key="value"
                variant="outline"
                class="w-full"
                :class="{
                  'border-black dark:border-white': store.fontSize === value,
                }"
                @click="store.sizeChanged(value)"
              >
                {{ desc }}
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>主题色</h2>
            <div class="grid grid-cols-3 justify-items-center gap-2">
              <Button
                v-for="{ label, value } in colorOptions"
                :key="value"
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white':
                    store.primaryColor === value,
                }"
                @click="store.colorChanged(value)"
              >
                <span
                  class="mr-2 inline-block h-4 w-4 rounded-full"
                  :style="{
                    background: value,
                  }"
                />
                {{ label }}
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>自定义主题色</h2>
            <div ref="pickColorsContainer">
              <PickColors
                v-if="pickColorsContainer"
                v-model:value="primaryColor"
                show-alpha
                :format="format"
                :format-options="formatOptions"
                :theme="store.isDark ? 'dark' : 'light'"
                :popup-container="pickColorsContainer"
                @change="store.colorChanged"
              />
            </div>
          </div>
          <div class="space-y-2">
            <h2>代码块主题</h2>
            <div>
              <Select
                v-model="store.codeBlockTheme"
                @update:model-value="store.codeBlockThemeChanged"
              >
                <SelectTrigger>
                  <SelectValue placeholder="Select a fruit" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem
                    v-for="{ label, value } in codeBlockThemeOptions"
                    :key="label"
                    :value="value"
                  >
                    {{ label }}
                  </SelectItem>
                </SelectContent>
              </Select>
            </div>
          </div>
          <div class="space-y-2">
            <h2>图注格式</h2>
            <div class="grid grid-cols-3 justify-items-center gap-2">
              <Button
                v-for="{ label, value } in legendOptions"
                :key="value"
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': store.legend === value,
                }"
                @click="store.legendChanged(value)"
              >
                {{ label }}
              </Button>
            </div>
          </div>

          <div class="space-y-2">
            <h2>Mac 代码块</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': store.isMacCodeBlock,
                }"
                @click="!store.isMacCodeBlock && store.macCodeBlockChanged()"
              >
                开启
              </Button>
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': !store.isMacCodeBlock,
                }"
                @click="store.isMacCodeBlock && store.macCodeBlockChanged()"
              >
                关闭
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>微信外链转底部引用</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': store.isCiteStatus,
                }"
                @click="!store.isCiteStatus && store.citeStatusChanged()"
              >
                开启
              </Button>
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': !store.isCiteStatus,
                }"
                @click="store.isCiteStatus && store.citeStatusChanged()"
              >
                关闭
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>段落首行缩进</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': store.isUseIndent,
                }"
                @click="!store.isUseIndent && store.useIndentChanged()"
              >
                开启
              </Button>
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': !store.isUseIndent,
                }"
                @click="store.isUseIndent && store.useIndentChanged()"
              >
                关闭
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>自定义 CSS 面板</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white':
                    displayStore.isShowCssEditor,
                }"
                @click="!displayStore.isShowCssEditor && customStyle()"
              >
                开启
              </Button>
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': !displayStore.isShowCssEditor,
                }"
                @click="displayStore.isShowCssEditor && customStyle()"
              >
                关闭
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>编辑区位置</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': store.isEditOnLeft,
                }"
                @click="!store.isEditOnLeft && store.toggleEditOnLeft()"
              >
                左侧
              </Button>
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': !store.isEditOnLeft,
                }"
                @click="store.isEditOnLeft && store.toggleEditOnLeft()"
              >
                右侧
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>模式</h2>
            <div class="grid grid-cols-5 justify-items-center gap-2">
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': !isDark,
                }"
                @click="store.toggleDark(false)"
              >
                <Sun class="h-4 w-4" />
              </Button>
              <Button
                class="w-full"
                variant="outline"
                :class="{
                  'border-black dark:border-white': isDark,
                }"
                @click="store.toggleDark(true)"
              >
                <Moon class="h-4 w-4" />
              </Button>
            </div>
          </div>
          <div class="space-y-2">
            <h2>样式配置</h2>
            <Button @click="store.resetStyleConfirm">
              重置
            </Button>
          </div>
        </div>
      </PopoverContent>
    </Popover>

    <div
      class="space-x-1 bg-background text-background-foreground mx-2 flex items-center border rounded-md"
    >
      <Button variant="ghost" class="shadow-none" @click="copy">
        复制
      </Button>
      <Separator orientation="vertical" class="h-5" />
      <DropdownMenu v-model="copyMode">
        <DropdownMenuTrigger as-child>
          <Button variant="ghost" class="px-2 shadow-none">
            <ChevronDownIcon class="text-secondary-foreground h-4 w-4" />
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align="end" :align-offset="-5" class="w-[200px]">
          <DropdownMenuRadioGroup v-model="copyMode">
            <DropdownMenuRadioItem value="txt">
              公众号格式
            </DropdownMenuRadioItem>
            <DropdownMenuRadioItem value="html">
              HTML 格式
            </DropdownMenuRadioItem>
          </DropdownMenuRadioGroup>
        </DropdownMenuContent>
      </DropdownMenu>
    </div>

    <PostInfo />

    <Toaster rich-colors position="top-center" />
  </header>
</template>

<style lang="less" scoped>
.menubar {
  user-select: none;
}

kbd {
  display: inline-flex;
  justify-content: center;
  align-items: center;
  border: 1px solid #a8a8a8;
  padding: 1px 4px;
  border-radius: 2px;
}
</style>
