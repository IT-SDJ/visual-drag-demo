<template>
    <div>
        <div :class="isDarkMode ? 'dark toolbar' : 'toolbar'">
            <el-button @click="handleAceEditorChange">JSON</el-button>
            <el-button @click="undo">撤消</el-button>
            <el-button @click="redo">重做</el-button>
            <label for="input" class="insert">
                插入图片
                <input
                    id="input"
                    type="file"
                    hidden
                    @change="handleFileChange"
                />
            </label>

            <el-button style="margin-left: 10px;" @click="preview(false)">预览</el-button>
            <el-button @click="save">保存</el-button>
            <el-button @click="clearCanvas">清空画布</el-button>
            <el-button :disabled="!areaData.components.length" @click="compose">组合</el-button>
            <el-button
                
                :disabled="!curComponent || curComponent.isLock || curComponent.component != 'Group'"
                @click="decompose"
            >
                拆分
            </el-button>

            <el-button :disabled="!curComponent || curComponent.isLock" @click="lock">锁定</el-button>
            <el-button :disabled="!curComponent || !curComponent.isLock" @click="unlock">解锁</el-button>
            <el-button @click="preview(true)">截图</el-button>

            <div class="canvas-config">
                <span>画布大小</span>
                <input v-model="canvasStyleData.width" />
                <span>*</span>
                <input v-model="canvasStyleData.height" />
            </div>
            <div class="canvas-config">
                <span>画布比例</span>
                <input v-model="scale" @input="handleScaleChange" /> %
            </div>
            <el-switch
                v-model="switchValue"
                class="dark-mode-switch"
                active-icon-class="el-icon-sunny"
                inactive-icon-class="el-icon-moon"
                active-color="#000"
                @change="handleToggleDarkMode"
            >
            </el-switch>
        </div>

        <!-- 预览 -->
        <Preview v-if="isShowPreview" :is-screenshot="isScreenshot" @close="handlePreviewChange" />
        <AceEditor v-if="isShowAceEditor" @closeEditor="closeEditor" />
    </div>
</template>

<script>
import generateID from '@/utils/generateID'
import toast from '@/utils/toast'
import { mapState } from 'vuex'
import Preview from '@/components/Editor/Preview'
import AceEditor from '@/components/Editor/AceEditor.vue'
import { commonStyle, commonAttr } from '@/custom-component/component-list'
import eventBus from '@/utils/eventBus'
import { $ } from '@/utils/utils'
import changeComponentsSizeWithScale, { changeComponentSizeWithScale } from '@/utils/changeComponentsSizeWithScale'

export default {
    components: { Preview, AceEditor },
    data() {
        return {
            isShowPreview: false,
            isShowAceEditor: false,
            timer: null,
            isScreenshot: false,
            scale: 100,
            switchValue: false,
        }
    },
    computed: mapState(['componentData', 'canvasStyleData', 'areaData', 'curComponent', 'curComponentIndex', 'isDarkMode']),

    created() {
        eventBus.$on('preview', this.preview)
        eventBus.$on('save', this.save)
        eventBus.$on('clearCanvas', this.clearCanvas)

        this.scale = this.canvasStyleData.scale
        const savedMode = JSON.parse(localStorage.getItem('isDarkMode'))
        if (savedMode) {
            this.handleToggleDarkMode(savedMode)
        }
    },
    methods: {
        handleToggleDarkMode(value) {
            if (value !== null) {
                this.$store.commit('toggleDarkMode', value)
                this.switchValue = value
            }
        },
        handleScaleChange() {
            clearTimeout(this.timer)
            this.timer = setTimeout(() => {
                // 画布比例设一个最小值，不能为 0
                // eslint-disable-next-line no-bitwise
                this.scale = ~~this.scale || 1
                changeComponentsSizeWithScale(this.scale)
            }, 1000)
        },

        handleAceEditorChange() {
            this.isShowAceEditor = !this.isShowAceEditor
        },

        closeEditor() {
            this.handleAceEditorChange()
        },

        lock() {
            this.$store.commit('lock')
        },

        unlock() {
            this.$store.commit('unlock')
        },

        compose() {
            this.$store.commit('compose')
            this.$store.commit('recordSnapshot')
        },

        decompose() {
            this.$store.commit('decompose')
            this.$store.commit('recordSnapshot')
        },

        undo() {
            this.$store.commit('undo')
        },

        redo() {
            this.$store.commit('redo')
        },

        handleFileChange(e) {
            const file = e.target.files[0]
            if (!file.type.includes('image')) {
                toast('只能插入图片')
                return
            }

            const reader = new FileReader()
            reader.onload = (res) => {
                const fileResult = res.target.result
                const img = new Image()
                img.onload = () => {
                    const component = {
                        ...commonAttr,
                        id: generateID(),
                        component: 'Picture',
                        label: '图片',
                        icon: '',
                        propValue: {
                            url: fileResult,
                            flip: {
                                horizontal: false,
                                vertical: false,
                            },
                        },
                        style: {
                            ...commonStyle,
                            top: 0,
                            left: 0,
                            width: img.width,
                            height: img.height,
                        },
                    }

                    // 根据画面比例修改组件样式比例 https://github.com/woai3c/visual-drag-demo/issues/91
                    changeComponentSizeWithScale(component)

                    this.$store.commit('addComponent', { component })
                    this.$store.commit('recordSnapshot')

                    // 修复重复上传同一文件，@change 不触发的问题
                    $('#input').setAttribute('type', 'text')
                    $('#input').setAttribute('type', 'file')
                }

                img.src = fileResult
            }

            reader.readAsDataURL(file)
        },

        preview(isScreenshot) {
            this.isScreenshot = isScreenshot
            this.isShowPreview = true
            this.$store.commit('setEditMode', 'preview')
        },

        save() {
            localStorage.setItem('canvasData', JSON.stringify(this.componentData))
            localStorage.setItem('canvasStyle', JSON.stringify(this.canvasStyleData))
            this.$message.success('保存成功')
        },

        clearCanvas() {
            this.$store.commit('setCurComponent', { component: null, index: null })
            this.$store.commit('setComponentData', [])
            this.$store.commit('recordSnapshot')
        },

        handlePreviewChange() {
            this.isShowPreview = false
            this.$store.commit('setEditMode', 'edit')
        },
    },
}
</script>

<style lang="scss" scoped>
.toolbar {
    padding: 15px 10px;
    white-space: nowrap;
    overflow-x: auto;
    background: var(--main-bg-color);
    border-color: var(--ace-bg-color);
    border-bottom: 1px solid var(--border-color);

    .canvas-config {
        display: inline-block;
        margin-left: 10px;
        font-size: 14px;
        color: var(--text-color);

        input {
            width: 50px;
            margin-left: 4px;
            outline: none;
            padding: 0 5px;
            border: 1px solid var(--border-color);
            color: #606266;
        }

        span {
            margin-left: 10px;
        }
    }

    .el-button--small {
        background: var(--main-bg-color);
        border: 1px solid var(--toolbar-border-color);
        color: var(--button-text-color);
    }

    .el-button--small:hover {
        background: var(--button-active-text-color);
        border-color: var(--actived-bg-color);
        color: var(--main-bg-color);
    }

    .insert {
        display: inline-block;
        line-height: 1;
        white-space: nowrap;
        cursor: pointer;
        border: 1px solid var(--toolbar-border-color);
        color: var(--text-color);
        appearance: none;
        text-align: center;
        box-sizing: border-box;
        outline: 0;
        margin: 0;
        transition: .1s;
        font-weight: 500;
        padding: 9px 15px;
        font-size: 12px;
        border-radius: 3px;
        margin-left: 10px;
    }

    .insert {
        background: var(--main-bg-color);
        border: 1px solid var(--toolbar-border-color);
        color: var(--button-text-color);

        &:active {
            background: var(--button-active-text-color);
            border-color: var(--actived-bg-color);
            color: var(--main-bg-color);
        }

        &:hover {
            background: var(--button-active-text-color);
            border-color: var(--actived-bg-color);
            color: var(--main-bg-color);
        }
    }

    .el-button.is-disabled {
        color: var(--disable-text-color);
        border-color: var(--disable-border-color);
        background: var(--main-bg-color);

        &:hover {
            background: var(--main-bg-color);
        }
    }
}
</style>
