<!--
 * @FileDescription: 自定义脚本相关功能
 * @Author: Stapxs
 * @Date: 2024/08/12
 * @Version: 1.0
-->

<template>
    <div class="script-view">
        <div class="list">
            <header>
                <div>
                    <span>{{ $t('title_scripts') }}</span>
                    <font-awesome-icon :icon="['fas', 'globe']" style="margin-right: 15px;" />
                    <font-awesome-icon @click="cnewScript" :icon="['fas', 'add']" />
                </div>
                <span>{{ $t('scripts_tip') }}</span>
            </header>
            <div class="list-body">
                <div v-for="(item, index) in savedList"
                    :class="{selected: (select == item.title && editScript)}"
                    :key="index">
                    <div @click="selectItem(item)">
                        <div style="flex: 1">
                            <h2>{{ item.title }}</h2>
                            <span>
                                <font-awesome-icon :icon="['fas', 'code-branch']" />
                                {{ $t('scripts_run_' + item.condition) }}{{ $t('scripts_run_trigger') }}
                            </span>
                        </div>
                        <span v-if="item.enabled">{{ $t('statue_enabled') }}</span>
                        <span v-else>{{ $t('statue_disabled') }}</span>
                    </div>
                    <div>
                        <font-awesome-icon @click="item.enabled = !item.enabled;updateSave();" :icon="['fas', 'check']" />
                        <font-awesome-icon :icon="['fas', 'edit']" @click="selectItem(item)" />
                        <font-awesome-icon :icon="['fas', 'trash-alt']" @click="remove(item.title)" />
                    </div>
                </div>
            </div>
        </div>
        <div class="editor-main" v-if="editScript">
            <div class="save-controller">
                <font-awesome-icon :icon="['fas', 'code-branch']" />
                <span>{{ $t('scripts_run_condition') }}</span>
                <div class="opt-item">
                    <select v-model="condition">
                        <option value="message">{{ $t('scripts_run_message') }}</option>
                    </select>
                </div>
                <div style="flex: 1;"></div>
                <button class="ss-button"
                    @click="save">
                    <font-awesome-icon :icon="['fas', 'save']" />
                    <span>{{ $t('scripts_run_save') }}</span>
                </button>
                <button class="ss-button"
                    @click="editScript = false">
                    <font-awesome-icon :icon="['fas', 'times']" />
                </button>
            </div>
            <prism-editor class="editor" v-model="script"
                :highlight="highlighter" line-numbers></prism-editor>
        </div>
        <div class="editor-main notice" v-else>
            <div class="ss-card">
                <font-awesome-icon :icon="['fas', 'info-circle']" />
                <span>{{ $t('scripts_run_notice') }}</span>
                <button class="ss-button" style="width: 100%; margin-top: 10px;"
                    @click="openLink('https://github.com/Stapxs/Stapxs-QQ-Lite-2.0/blob/next/src/pages/Scripts.vue')">
                    Github
                </button>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import 'prismjs/themes/prism.css'

import { Connector } from '@/function/connect'
import { runtimeData } from '@/function/msg'
import { defineComponent } from 'vue'
import { openLink } from '@/function/utils/appUtil'

import { PrismEditor } from 'vue-prism-editor'
import { highlight, languages } from 'prismjs'

import { getMsgData } from '@/function/utils/msgUtil'
import { PopInfo, PopType } from '@/function/base'

export default defineComponent({
    name: 'ViewScripts',
    components: {
        PrismEditor
    },
    data() {
        return {
            runtimedata: runtimeData,
            openLink: openLink,
            script: '',
            condition: 'message',
            editScript: false,
            savedList: [] as {
                title: string,
                condition: string,
                script: string,
                enabled: boolean
            }[],
            select: '',

            message: null as any | null,
            msgInfo: null as {[key: string]: any} | null,
            isMe: false
        }
    },
    methods: {
        cnewScript() {
            this.select = ''
            this.script = '/*\n    title: Demo Script\n*/\n\nconsole.log(\'Hello World!\')'
            this.editScript = true
        },
        updateSave() {
            const saveJson = JSON.stringify(this.savedList)
            if(runtimeData.tags.isElectron) {
                runtimeData.reader?.send('sys:store', {
                    key: 'scripts', value: saveJson
                })
            } else {
                localStorage.setItem('scripts', saveJson)
            }
        },
        save() {
            this.editScript = false
            // title 获取 script 第二行 title: 后的内容
            const title = this.script.split('\n')[1].split(':')[1].trim()
            if(title == '') {
                new PopInfo().add(PopType.ERR, this.$t('scripts_run_title_err'))
                return
            }
            if(this.savedList.find((item) => item.title == title)) {
                // 覆盖
                this.savedList = this.savedList.map((item) => {
                    if(item.title == title) {
                        item.condition = this.condition
                        item.script = this.script
                    }
                    return item
                })
            } else {
                this.savedList.push({
                    title: title,
                    condition: this.condition,
                    script: this.script,
                    enabled: false
                })
            }
            this.updateSave()
        },
        remove(title: string) {
            this.savedList = this.savedList.filter((item) => item.title != title)
            this.updateSave()
        },

        selectItem(item: {
                title: string,
                condition: string,
                script: string,
                enabled: boolean
            }) {
            // 选中脚本
            this.select = item.title
            this.script = item.script
            this.condition = item.condition
            this.editScript = true
        },

        highlighter(code: any) {
            return highlight(code, languages.js, 'js')
        },

        onMessage() {
            // 筛选出消息中的 condition 为 message 的脚本
            const scripts = this.savedList.filter((item) => item.condition == 'message')
            for(const script of scripts) {
                if(script.enabled) {
                    eval(script.script)
                }
            }
        },

        // 发送消息
        sendText(text: string) {
            if(this.message && this.msgInfo) {
                const id = this.msgInfo.group_id ?? this.msgInfo.private_id
                const type = this.msgInfo.group_id ? 'group' : 'private'
                let action = runtimeData.jsonMap.message_list.name_group_send ?? 'send_msg'
                if(type == 'private') {
                    action = runtimeData.jsonMap.message_list.name_user_send ?? 'send_msg'
                }
                if(type == 'group') {
                    this.sendMsg(action, {group_id: id, message: { type: 'text', data: { text: text }}})
                } else {
                    this.sendMsg(action, {user_id: id, message: { type: 'text', data: { text: text }}})
                }
            }
        },

        sendMsg(action: string, params: {[key: string]: any}) {
            Connector.send(action, params, 'ScriptBack')
        }
    },
    async mounted() {
        // 读取保存的脚本
        const electron = window.require('electron')
        let data = null
        if(electron) {
            const reader = electron.ipcRenderer
            if(reader) data = await reader.invoke('sys:getStore', 'scripts')
        }
        if(data) {
            this.savedList = JSON.parse(data)
        } else {
            data = localStorage.getItem('scripts')
            if(data) {
                this.savedList = JSON.parse(data)
            }
        }
        // 监听数据
        this.$watch(() => runtimeData.watch.newMsg, () => {
            if(runtimeData.sysConfig.append_scripts) {
                this.message = runtimeData.watch.newMsg
                // FIXME 这儿没有判断是不是自身返回的消息，注意谨防死循环
                const infoList = getMsgData('message_info', this.message, runtimeData.jsonMap.message_info)
                if(infoList != undefined) {
                    this.msgInfo = infoList[0]
                }
                if(this.msgInfo) {
                    this.isMe = Number(this.msgInfo.sender) == Number(runtimeData.loginInfo.uin)
                } else {
                    this.isMe = false
                }
                this.onMessage()
            }
        })
    }
})
</script>

<style scoped>
.script-view {
    background: var(--color-bg);
    display: flex;
    height: 100%;
    width: 100%;
}
.list {
    width: 270px;
    display: flex;
    flex-direction: column;
}
.list > header {
    padding: 10px 10px 0 10px;
    margin: 15px;
}
.list > header > div {
    align-items: center;
    display: flex;
}
.list > header > div > span {
    color: var(--color-font);
    font-weight: bold;
    font-size: 1rem;
    flex: 1;
}
.list > header > div > svg {
    color: var(--color-font-2);
    cursor: pointer;
}
.list > header > span {
    background: var(--color-card-2);
    color: var(--color-font-1);
    font-size: 0.9rem;
    padding: 15px;
    margin-top: 10px;
    border-radius: 7px;
    display: block;
}

.list-body {
    overflow-x: scroll;
    padding: 0 25px;
    flex: 1;
}
.list-body::-webkit-scrollbar {
    display: none;
}
.list-body > div {
    background: var(--color-card-1);
    transition: background .3s;
    flex-direction: column;
    margin-bottom: 10px;
    border-radius: 7px;
    cursor: pointer;
    display: flex;
    padding: 10px;
}
.list-body > div.selected {
    background: var(--color-main);
}
.list-body > div.selected h2 {
    color: var(--color-font-r);
}
.list-body > div.selected span,
.list-body > div.selected > div:last-child > svg {
    color: var(--color-font-1-r);
}
.list-body h2 {
    font-size: 1rem;
    margin: 0;
    color: var(--color-main);
    transition: color .3s;
}
.list-body span {
    font-size: 0.9rem;
    color: var(--color-font-2);
    transition: color .3s;
    margin-left: 5px;
}
.list-body span svg {
    font-size: 0.85rem;
    margin: 0 5px;
}
.list-body > div > div:first-child {
    display: flex;
    flex: 1;
}
.list-body > div > div:last-child {
    justify-content: space-evenly;
    margin-top: 5px;
    display: flex;
}
.list-body > div > div:last-child > svg {
    transition: color .3s;
    color: var(--color-font-1);
    margin: 5px 10px;
}

.editor-main {
    background: var(--color-card);
    padding: 20px 0;
    flex: 1;
}

.editor-main.notice > div {
    width: 50%;
    background: var(--color-card-1);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    margin: 0 auto;
    margin-top: 50%;
    transform: translateY(-50%);
    padding: 30px;
}
.editor-main.notice > div > svg {
    font-size: 2.5rem;
    color: var(--color-main);
    margin: 20px 0;
}

.editor {
    font-size: 0.9rem;
}
.editor > div:first-child {
    color: var(--color-font-1);
}

.save-controller {
    display: flex;
    padding: 15px;
    margin-top: -15px;
    align-items: center;
}
.save-controller button:first-child {
    margin-left: 0;
}
.save-controller button {
    font-size: 0.8rem;
    margin-left: 10px;
    cursor: pointer;
    min-width: 35px;
    height: 35px;
}
.save-controller button > svg {
    margin: 0 5px;
}
.save-controller button > span {
    margin: 0 5px;
}
.save-controller > span,
.save-controller > svg {
    color: var(--color-font-2);
    font-size: 0.9rem;
    margin-right: 10px;
}

.opt-item {
    padding: 0;
}
.opt-item > select {
    height: 35px;
}
.opt-item:hover {
    background: transparent;
}
</style>
