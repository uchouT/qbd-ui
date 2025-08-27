<template>
    <el-dialog v-model="dialogVisible" :close-on-click-modal="!metadataDownloading || !torrentParsed"
        :close-on-press-escape="!metadataDownloading || !torrentParsed" title="任务添加" :width="dialogWidth"
        class="torrent-dialog">
        <div v-show="!torrentParsed">
            <el-tabs v-model="activeTab" tab-position="top" style="height: auto">
                <el-tab-pane label="Url" name="url">
                    <el-form :model="torrentUrl" :rules="rules">
                        <el-form-item label="链接" prop="url">
                            <el-input v-model:model-value="torrentUrl.url" placeholder=" torrent url..."
                                :autosize="{ minRows: 2 }" type="textarea" :disabled="metadataDownloading" />
                        </el-form-item>
                        <el-form-item label="保存路径" prop="save_path">
                            <el-input v-model:model-value="torrentUrl.save_path" type="textarea"
                                :autosize="{ minRows: 1 }" :disabled="metadataDownloading" :placeholder="save_path" />
                        </el-form-item>
                    </el-form>
                </el-tab-pane>
                <el-tab-pane label="File" name="file">
                    <el-upload ref="upload" action="/api/torrent" accept=".torrent" :limit="1" :auto-upload="false"
                        :data="uploadData" name="torrent" :headers="uploadHeaders" :on-success="handleUploadSuccess"
                        :on-error="handleUploadError" :on-change="handleFileChange" v-model:file-list="filelist">
                        <template #trigger>
                            <el-button type="primary">选择文件</el-button>
                        </template>
                    </el-upload>

                    <el-form-item label="保存路径" style="margin-top: 16px;">
                        <el-input v-model:model-value="torrentFile.save_path" type="textarea" :placeholder="save_path"
                            :disabled="metadataDownloading" />
                    </el-form-item>
                    <template #tip>
                        <div class="el-upload__tip text-red">
                            限制上传1个文件，新文件将覆盖旧文件
                        </div>
                    </template>
                </el-tab-pane>
            </el-tabs>
        </div>
        <div v-if="torrentParsed">
            <Task v-model:taskData="taskAdd" @ok="addTask" @cancel="handleCancel" />
        </div>
        <template #footer v-if="!torrentParsed">
            <span class="dialog-footer">
                <el-button @click="handleCancel">取消</el-button>
                <el-button type="primary" @click="handleConfirm" :loading="metadataDownloading" :disabled="!canConfirm">
                    {{ getConfirmButtonText() }}
                </el-button>
            </span>
        </template>
    </el-dialog>
</template>

<script setup>
import { ref, computed } from 'vue'
import { ElMessage } from 'element-plus'
import { useWindowSize } from '@vueuse/core'
import api from '../api'
import Task from './Task.vue'
import CONFIG from '../config'
const save_path = ref();
const emit = defineEmits(['load'])
const { width } = useWindowSize()
const dialogVisible = ref(false)
const metadataDownloading = ref(false)
const activeTab = ref('url') // 当前激活的标签页
const filelist = ref([])
const fileSelected = ref(false) // 文件是否已选择
const torrentParsed = ref(false)
const torrentUrl = ref({
    url: '',
    save_path: '',
})
const torrentFile = ref({
    save_path: ''
})
const upload = ref(null)
const rules = ref({
    url: [{ required: true, message: '请输入 torrent 链接', trigger: 'blur' }]
})

const taskAdd = ref({
    torrent_res: {
        torrent_name: '',
        save_path: '',
        hash: ''
    },
    upload_type: {
        type: 'Rclone',
    },
    upload_path: '',
    max_size: null,
    seeding_time_limit: -2,
    ratio_limit: -2,
    custom_content: false,
    selected_file_index: null
})

const addTask = (fun) => {
    api.post('/api/task', taskAdd.value)
        .then(res => {
            ElMessage.success("任务添加成功")
            dialogVisible.value = false
            torrentParsed.value = false
            // 触发任务列表刷新
            emit('load')
        }).finally(fun)
}
// 上传相关配置
const uploadHeaders = ref({
    'Authorization': window.authorization || ''
})

// 计算属性：上传数据
const uploadData = computed(() => ({
    save_path: torrentFile.value.save_path
}))

// 计算属性：是否可以确认
const canConfirm = computed(() => {
    if (activeTab.value === 'url') {
        return torrentUrl.value.url.trim()
    } else if (activeTab.value === 'file') {
        return fileSelected.value
    }
    return false
})

// 计算属性：确认按钮文本
const getConfirmButtonText = () => {
    if (metadataDownloading.value) {
        return activeTab.value === 'url' ? '元数据下载中...' : '上传中...'
    }
    return activeTab.value === 'url' ? '添加URL' : '上传并添加'
}

// 响应式对话框宽度
const dialogWidth = computed(() => {
    if (width.value <= 768) {
        return '95%'
    } else if (width.value <= 1024) {
        return '70%'
    } else {
        return '50%'
    }
})

const show = () => {
    dialogVisible.value = true
    activeTab.value = 'url'
    save_path.value = CONFIG.value.default_save_path || "请输入保存路径";
    fileSelected.value = false
    if (!torrentParsed.value) {
        taskAdd.value = {
            torrent_res: {
                torrent_name: '',
                save_path: '',
                hash: ''
            },
            upload_type: {
                type: 'Rclone',
            },
            upload_path: '',
            max_size: null,
            seeding_time_limit: -2,
            ratio_limit: -2,
            custom_content: false,
            selected_file_index: null
        }
    }
    torrentUrl.value = {
        url: '',
        save_path: ''
    }
    torrentFile.value = {
        save_path: ''
    }
    filelist.value = []

    // 更新授权头
    uploadHeaders.value.Authorization = window.authorization || ''
}

const handleConfirm = async () => {
    if (activeTab.value === 'url') {
        await handleUrlConfirm()
    } else if (activeTab.value === 'file') {
        await handleFileConfirm()
    }
}

// URL 页面的确认处理
const handleUrlConfirm = async () => {
    metadataDownloading.value = true

    // 准备请求数据，如果 save_path 为空则不包含该字段
    const requestData = {
        url: torrentUrl.value.url,
        save_path: torrentUrl.value.save_path
    }

    // 调用API添加URL
    api.post('/api/torrent', requestData)
        .then(res => {
            taskAdd.value.torrent_res = res['data']
            torrentParsed.value = true
        })
        .finally(() => {
            metadataDownloading.value = false
        })
}

// File 页面的确认处理
const handleFileConfirm = async () => {
    if (!fileSelected.value) {
        ElMessage.warning('请先选择文件')
        return
    }
    metadataDownloading.value = true
    upload.value.submit()
}



// 文件选择变化处理
const handleFileChange = (file, fileList) => {
    fileSelected.value = fileList.length > 0
}


const handleUploadSuccess = (res) => {
    console.log('上传成功:', res)
    metadataDownloading.value = false
    let code = res.code
    if (code >= 200 && code < 300) {
        taskAdd.value.torrent_res = res['data'];
        ElMessage.success('种子上传成功')
        torrentParsed.value = true
    } else {
        ElMessage.error(res.message || '上传失败')
        if (code === 403) {
            localStorage.removeItem("authorization")
            setTimeout(() => {
                location.reload()
            }, 1000)
        }
    }
}

const handleUploadError = (error, file) => {
    console.error('上传失败:', error)
    metadataDownloading.value = false

    let errorMessage = '上传失败'
    if (error && error.message) {
        errorMessage = error.message
    }

    ElMessage.error(errorMessage)
}

const deleteTorrent = () => {
    let hash = taskAdd.value.torrent_res.hash;
    api.del(`/api/torrent?hash=${hash}`)
        .then(res => {
            ElMessage.success("种子删除成功")
        })
}

const handleCancel = () => {
    if (metadataDownloading.value) {
        deleteTorrent();
        //TODO:
        //  如果是 url，中断 export。
        // 如果是 File，中断上传
        metadataDownloading.value = false
    } else if (torrentParsed.value) {
        deleteTorrent();
        torrentParsed.value = false
    }
    dialogVisible.value = false;

}

defineExpose({ show })
</script>